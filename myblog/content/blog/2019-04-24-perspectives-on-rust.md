---
title: Perspectives on Building a Medical Device with Rust
layout: post
---

During the redesign of FreePulse following the Nepal pre-clinical trials, I
made the decision to re-write the software in the [Rust programming
language][rustlang]. This decision had far-reaching consequences for the
project, and I wanted to delve a little bit into the motivation for this
decision, the benefits Rust has provided FreePulse, and my experiences with
deep-diving into the world of Rust embedded development.

To be clear: I am _not_ a Rust expert. However, I would make the case that you
don't _have_ to be a Rust expert in order to take advantage of a lot of the
benefit that Rust offers as a language. It's not without its warts, but overall
I feel that the quality, maintanability, and simplicity of the FreePulse
codebase has significantly improved as a result of the rewrite.

## Motivation for Changing Languages

FreePulse was originally written in C++ (the old-repository is open-sourced
[here][old-freepulse-repo]). I used the low-level HAL that was provided with
the STM32F4 chip in order to interface with the hardware and wrote my own
wrappers around accessing peripherals such as GPIO, SPI, and USART. From there,
I built out my application-specific logic as a simple update loop that iterated
through a set of "modules" (in reality, they were monolithic C++ classes) that
each provided an `.update()` method. Unit testing was limited at best.

Over time, I have learned much more about software best practices and the
importance of writing maintainable code. There are several design "red-flags" in
the old code base that made it very difficult to introduce changes without
breaking things: the monolithic classes were difficult to debug, state was
spread over multiple places in the code, and control flow was pretty difficult
to reason about. Perhaps even more importantly, the single-loop style of
updating meant that there was no way to specify the priority of different
tasks. The ECG sampler could not measure until the screen was done painting;
the pulse oximeter calibration would block all UI updates; high priority tasks
were at the mercy of their slower counterparts, and it was impossible to
provide timing guarantees on samplers. As the complexity of the software grew,
these issues become more and more difficult to work around. As a result, I
decided that FreePulse needed to be re-written to be multi-threaded.

Of course, "multi-threaded" in an embedded system generally means concurrency
via interrupt-based programming ([as opposed to
parallelism][concurrency-vs-parallelism], i.e. multiple cores of a CPU
performing operations simultaneously). Even though the embedded system only has
a single core processor, we _conceptually_ want the ability to describe
multiple independent processes that are running simultaneously. In
interrupt-based programming, we define an "idle" thread that is the lowest
priority task; we then define a set of other tasks that run at defined
intervals or in response to hardware events, each with a priority level that
determines whether it can preempt the currently running task when triggered.
This model is supported by the hardware in most embedded systems through the
use of the Nested Vector Interrupt Controller, or NVIC; it allows hardware
events to trigger execution of certain code hooks. Furthermore, each event has
a user-configurable priority level, allowing higher-priority events to preempt
the handling of lower-priority events.

The mechanisms of multi-threading aside, developing the logic of a
multi-threaded application is a complex and error-prone task. Even the most
diligent of engineers can be tripped up by multi-threaded bugs, which often
manifest themselves in mysterious ways and can be notoriously hard to debug.
This problem is one of the primary motivators for using the Rust language,
which provides compile-time guarantees of concurrency safety by the use of its
borrow checker. This doesn't make concurrent code in Rust completely
bulletproof, but it does eliminate whole classes of bugs that otherwise would
be quite difficult to identify and fix.

Concurrency safety wasn't the only reason for my switch to Rust, though. I want
to walk through a couple of quick examples where I think Rust really shines in
the context of FreePulse.

## Examples of Rust in FreePulse

### Pros

#### Hardware State Described By Type State

I can't count how many times I've been bitten in development by incorrectly
initializing a peripheral. This can happen very easily when quickly iterating
on software or changing how the breadboard is wired during development. A
sketched-out example in C++ (I have omitted some function definitions and
intialization for brevity):

```cpp
enum Pin_Num {
    PA0,
    PA1,
    PA2,
};

Pin_Num some_gpio_pin = PA1; // This is just an enum
configure_GPIO(some_gpio_pin, PULL_DOWN, OUTPUT);

int analogRead(Pin_Num pn) {
    uint8_t channel = get_adc_channel(pn);
    ADC_RegularChannelConfig(ADC1,channel,1,ADC_SampleTime_144Cycles);
    ADC_SoftwareStartConv(ADC1);
    while(!ADC_GetFlagStatus(ADC1, ADC_FLAG_EOC)) { };
    return ADC_GetConversionValue(ADC1);

}

// ...
// Somewhere later on, where I forgot this pin is actually set to be OUTPUT
analogRead(some_gpio_pin); // RUNTIME ERROR!
```

In this example, `some_gpio_pin` is of type `Pin_Num`, which is simply an enum
listing the GPIO pins available on the hardware. When we pass in a pin to the
`analogRead` function, we know nothing about the pin other than its name; this
is an issue because `analogRead` actually will not work unless the GPIO pin has
been configured as an analog input. We won't discover this issue until runtime,
when the call to `analogRead` makes our microcontroller mysteriously crash.

However, if we can manage to encode the hardware state inside the type of the
pin, then we can enforce _at compile time_ that we are passing in the correctly
configured hardware. This is accomplished through the use of type state!

Type state is not a language-specific solution-- here is a minimal example of
type state in C++:

```cpp
#include <stdio.h>

template <typename MODE>
class Pin {
    public:
        Pin(int num): num(num) {};
        int get_val () { return this->num; }
        void set_val (int val) { this->num = val; }
    private:
        int num;
};

struct Output {};
struct Input {};

void set_high(Pin<Output> pin) {
    pin.set_val(1);
}

int read(Pin<Input> pin) {
    return pin.get_val();
}

int main(int argc, char** argv) {
    Pin<Output> out_pin = Pin<Output>(23);
    printf("%d", read(out_pin));
}
```

Here, we are using a type variable `MODE` as a _phantom type_, which simply
means that it only exists to "mark" the type of an object with some extra
information. In this case, we are marking the `Pin` type to indicate whether it
is an `Input` or `Output` pin. The last line of this code snippet passes in an
output pin to the `read` function, which is only valid if the pin is configured
as an input. Since we specified the argument type of `read` as `Pin<Input>`, we
now get this helpful compiler error:

```
main.cpp: In function ‘int main(int, char**)’:
main.cpp:26:23: error: could not convert ‘out_pin’ from ‘Pin<Output>’ to ‘Pin<Input>’
     printf("%d", read(out_pin));
```

Awesome! We have successfully dodged a mysterious runtime error by catching
this misconfiguration at compile time. However, there are still a couple of
things about this approach that are less than optimal. One thing that
particularly sticks out is that the `Pin` class has a `set_val()` method
defined even if the pin is configured as an input (or `get_val()` when
the pin is an output). It would be very useful if we could _conditionally_
attach methods to the `Pin` struct.

Enter Rust!

```rust
use std::marker::PhantomData;

struct Input;
struct Output;

struct Pin<MODE> {
    _mode: PhantomData<MODE>,
    num: i32,
}

impl Pin<Output> {
    fn set_high(&mut self) {
        self.num = 1;
    }
}

impl Pin<Input> {
    fn read(&self) -> i32 {
        self.num
    }
}

fn main() {
    let in_pin = Pin::<Input>{num: 42, _mode: PhantomData};
    in_pin.set_high();
}
```

With Rust, we can use the `impl` block to attach methods to the `Pin` struct
_if the struct satisfies the specified type constraints_. In this way, only
input pins have a `read()` method, and only output pins have a `set_high()`
method. Sure enough, if we try to compile this code, we'll get an error on the
last line:

```
error[E0599]: no method named `set_high` found for type `Pin<Input>` in the current scope
  --> main.rs:25:13
   |
6  | struct Pin<MODE> {
   | ---------------- method `set_high` not found for this
...
25 |     out_pin.set_high();
   |             ^^^^^^^^

error: aborting due to previous error

For more information about this error, try `rustc --explain E0599`.
```

So this is great-- we can enforce that pins will only have methods supported by
the configuration described in their type. But how to enforce that these type
constraints are valid? This is where Rust's ownership system helps us out.
We can define constructors for these types that take ownership of a pin,
performs the appropriate configuration, and returns a new pin type with the
configuration parameters specified. Because of Rust's ownership system, an old
reference to a pin that has since been re-configured can no longer be used.
Here is a slightly modified version of the above example that demonstrates this
feature:

```rust
use std::marker::PhantomData;

struct Input;
struct Output;

struct Pin<MODE> {
    _mode: PhantomData<MODE>,
    num: i32,
}

impl Pin<Input> {
    fn read(&self) -> i32 {
        self.num
    }
}

impl Pin<Output> {
    fn set_high(&mut self) {
        self.num = 1;
    }

    fn into_input(self) -> Pin<Input> {
        // reconfigure pin to be an input
        Pin::<Input> { num: self.num, _mode: PhantomData }
    }
}

fn main() {
    let mut out_pin = Pin::<Output>{num: 42, _mode: PhantomData};
    let _in_pin = out_pin.into_input();
    out_pin.set_high();
}
```

This code attempts to re-use `out_pin` after the hardware resource has been
reconfigured to be an input. However, because the conversion from output to
input takes _ownership_ of the pin, `out_pin` is now considered a "moved value"
and is no longer a valid reference. The compiler helpfully reminds us of this:

```
error[E0382]: use of moved value: `out_pin`
  --> main.rs:31:5
   |
30 |     let _in_pin = out_pin.into_input();
   |                   ------- value moved here
31 |     out_pin.set_high();
   |     ^^^^^^^ value used here after move
   |
   = note: move occurs because `out_pin` has type `Pin<Output>`, which does not implement the `Copy` trait

error: aborting due to previous error

For more information about this error, try `rustc --explain E0382`.
```

To summarize: Rust's ability to conditionally bind methods with `impl` blocks
and its ownership system work together to provide a powerful way to express
hardware configuration via type state. I've spent a lot of time covering this
point because it has easily been the biggest quality-of-life improvement since
switching to Rust for FreePulse's development.

#### Syntax Features: `match`, Pattern Matching, and Closures

Rust also has some really nice syntax features that make it easy to express
complex logic in a safe and compiler-checked manner. For instance, here is a
snippet from the FreePulse SpO2 module displaying Rust's `match` statement and
pattern-matching:

```rust
match spo2.afe4400.self_check() {
    Ok(()) => {
        info!("SpO2 passed self-check");
        spo2.state = Spo2State::Idle;
    },
    Err(AfeError::SelfCheckFail) => error!("Spo2 failed self-check"),
    Err(e) => {
        error!("Communication failure during self check: {:?}.", e);
    }
};
```

Compare this to a roughly equivalent statement in C++:

```cpp
SomeResultType result = spo2.afe4400.self_check();
if (is_ok(result)) {
    printf("SpO2 passed self-check\n");
    spo2.state = Spo2State.Idle;
} else if (result.error == AfeError.SelfCheckFail) {
    printf("Spo2 failed self-check\n");
} else {
    printf("Communication failure during self check\n");
}
```

Rust `match` statements are checked by the compiler for exhaustiveness, so if
the output of `spot.afe4400.self_check()` changes later on, the compiler will
throw an error here if there is a new possible output that is not covered in
the match statement. I also happen to think it just looks clean and concise!

In addition to `match`, Rust has first-class support for closures, or anonymous
functions. This is not unique to Rust (C++ has lambdas, for example); however,
it is syntactically simple and easy to use. For instance, here is a closure
that updates the spo2 module after acquiring a lock on the USART1 peripheral:

```rust
let spo2 = resources.SPO2;
resources.USART1.lock(|usart| {
    spo2.update(usart);
});
```

#### Package Management System

Rust provides an excellent package management system via [crates.io][crates]
and `cargo`. This means that third-party dependencies are (a) enumerated and
(b) version-locked in a controlled format. This is another huge quality-of-life
improvement, especially for a medical device project-- since dependencies are
rigorously tracked as part of the project in `Cargo.toml` and the list is under
version control, it is much easier to ensure your builds are repeatable. As a
result of this, it is also much easier to assess risk from dependencies, known
as "SOUP" (software of unknown providence) in regulatory parlance.

#### Active Developer Community

This point cannot be overstated. Rust's developer community is active, engaged,
and incredibly helpful to newcomers. I have benefitted tremendously from
hanging around on the #rust-embedded IRC channel, and through the work of
japaric and the embedded rust workgroup, driver interfaces and standards are
getting developed at an astonishing pace. The [`embedded-hal`][embedded-hal]
project is providing a way for drivers to be interoperable with a wide variety
of microcontrollers; the [svd2rust][svd2rust] library provides mechanisms for
generating Rust hardware abstraction layers directly from the SVD specification
of a microcontroller; and that isn't even scratching the surface of the
[documentation][embedded-docs], [guides][embedded-book], and [high-level
frameworks][cortex-m-rtfm] that are being actively developed. I have been
working with embedded Rust for the past two years, and the amount of progress
made in that time is truly incredible.

### Cons

Although I think that switching to Rust has been a substantial win, there are
still a couple of areas where the switch can make things difficult.

#### Write All Your Own Drivers

Although this is changing with time, currently 99.9% of devices will not have
any driver written in Rust. This means that if you are using a new chip, you'll
need to break out the datasheet and start creating the firmware. It's not all
bad since this is also a good way to learn about the capabilities of the
hardware you are using, but it can be tiresome if you are trying to quickly
test out a possible hardware candidate. It also means that your handwritten
firmware could possibly contain bugs!

For projects with a substantial number of interacting systems, this could be a
significant barrier to using Rust. FreePulse has a relatively small number of
chips besides the microcontroller that require drivers, so this wasn't enough
of an inconvenience to outweigh the pros enumerated above.

#### Devtools Experience Needs to Improve

Compared to the tried-and-true autocomplete, go-to-definition, and other
IDE-like capabilities that are well supported for C-family languages, Rust's
developer tooling is still maturing. Autocomplete and go-to-definition work
_some_ of the time, but they struggle with macros and closures and often do not
work at all on generated microcontroller crates. It's not clear if there's a
timeline for this to be improved. I'm keeping my eye on the [Rust Language
Server][rls] project to see if tool improvements are coming down the pipeline!

## Conclusion

I think the combination of strong compiler checking, expressive type system,
and concise, elegant syntax make Rust a satisfying language in which to write--
and an easier language to read as well. FreePulse's migration to Rust has been
a huge learning experience, and I think the project is much better off as a
result of it.

[rustlang]: https://www.rust-lang.org/
[old-freepulse-repo]: https://github.com/ReeceStevens/freepulse
[concurrency-vs-parallelism]: https://stackoverflow.com/questions/1050222/what-is-the-difference-between-concurrency-and-parallelism#1050257
[f4-type-state]: https://github.com/ReeceStevens/f4/blob/453d572420ca172ffe6725bc6c692c5f374ccbc0/f4/src/gpio.rs#L197-L202
[rust-traits-vs-interfaces]: https://blog.rust-lang.org/2015/05/11/traits.html
[f4-conditional-api]: https://github.com/ReeceStevens/f4/blob/453d572420ca172ffe6725bc6c692c5f374ccbc0/f4/src/gpio.rs#L186-L194
[embedded-hal]: https://github.com/rust-embedded/embedded-hal
[svd2rust]: https://github.com/rust-embedded/svd2rust
[embedded-docs]: https://github.com/rust-embedded/docs
[embedded-book]: https://github.com/rust-embedded/book
[cortex-m-rtfm]: https://github.com/japaric/cortex-m-rtfm
[crates]: https://crates.io/
[rls]: https://github.com/rust-lang/rls
