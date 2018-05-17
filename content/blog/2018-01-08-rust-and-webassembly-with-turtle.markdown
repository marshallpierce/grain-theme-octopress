---
layout: post
title: "Rust and WebAssembly with Turtle"
date: "2018-01-08 09:54"
author:
categories: [software, rust, webassembly]
comments: true
published: true
---

In this post, I'll walk through a few of the highlights of getting [Turtle](http://turtle.rs/), a Rust library for creating animated drawings, to run in the browser with WebAssembly.

Rust has [recently](https://github.com/rust-lang/rust/pull/46115) gained built-in support for compiling to [WebAssembly](http://webassembly.org/). 
While it's currently considered experimental and has some rough edges, it's not too early to [start seeing what it can 
do with some demos](https://www.hellorust.com/). After an 
[experiment running rust-base64 via WebAssembly](https://bitbucket.org/marshallpierce/rust-wasm-base64) with surprisingly good performance, I saw [this tweet](https://twitter.com/gcouprie/status/938842985906700288) suggesting that someone port Turtle to run in the browser, and figured I'd give it a try [(see PR)](https://github.com/sunjay/turtle/pull/53). Turtle is a more interesting port than rust-base64's low level functionality: it's larger, and has dependencies on concepts that have no clear web analog, like forking processes and desktop windows. You can run it yourself with any recent browser; just follow [these instructions](https://github.com/sunjay/turtle/pull/53/files#diff-e3d3691703168e79428c9bd1c084f5bc).

## Why Rust + WebAssembly?

See the [official WebAssembly site](http://webassembly.org/) for more detail, but in essence WebAssembly is an escape hatch from 
the misery of JavaScript as the sole language runtime available in browsers. Even though it's already possible to 
transpile to JavaScript with tools like emscripten or languages like TypeScript, that doesn't solve problems like 
JavaScript's lack of proper numeric types. WebAssembly provides a much better target for web-deployed code than 
JavaScript: it's compact, fast to parse, and JIT compiler friendly.

WebAssembly is not competition for JavaScript directly. It's more akin to JVM bytecode. This makes it a great fit for Rust, though, which has a minimal runtime and no garbage collector.

## Turtle

Turtle is a library for making little single-purpose applications that draw graphics ala the 
[Logo](https://en.wikipedia.org/wiki/Logo_%28programming_language%29) language. See below for a simple program (the 
[`snowflake`](https://github.com/sunjay/turtle/blob/5db76f7d12baa6676f84088432e1896c42da80c6/examples/snowflake.rs) 
example in the Turtle repo) that's most of the way through its drawing logic.

<img class='centered' src="${r '/images/turtle/koch-snowflake.png'}"/>

### Turtle's existing architecture

Turtle is designed to be friendly to beginner programmers, so unlike most graphics-related code, it's not based around
the concept of an explicit render loop. Instead, Turtle programs simply take over the main thread and run straightforward-looking
code like this (which draws a circle):

```rust
for _ in 0..360 {
    // Move forward three steps
    turtle.forward(3.0);
    // Rotate to the right (clockwise) by 1 degree
    turtle.right(1.0);
}
```

The small pauses needed for animation are implicit, and programmers can use local variables in a natural way to manage state. This has some implications for running in the browser, as we'll see.

## Compiling to `wasm32`

As with any wasm project, the first step is to get the toolchain:

```sh
rustup update
rustup target add wasm32-unknown-unknown --toolchain nightly
```

Hypothetically, this would be all that's needed to compile the example turtle programs:

```sh
cargo +nightly build --target=wasm32-unknown-unknown --examples
```

However, there were naturally some dependencies that didn't play nice with the wasm32 target, mainly involving [Piston](https://github.com/pistondevelopers/piston), the graphics library used internally. Fortunately, Piston's various crates are nicely broken down to separate the platform-specific parts from platform-agnostic things like math helper functions, so introducing a `canvas` feature to control Piston dependencies (among other things) was all that was needed. That brings us to:

```sh
cargo +nightly build --no-default-features --features=canvas --target=wasm32-unknown-unknown --release --examples
```

## Control flow

The current architecture of a Turtle program running on a desktop uses two processes: one to run the user's logic, and
another to handle rendering. When the program starts, it first forks a render process, then passes control flow to the
user's logic in the parent process. When code like `turtle.forward()` runs, that ends up passing drawing commands (and
 other communication) via stdin/stdout to the child rendering process, which uses those to draw into a 
[Piston](https://github.com/pistondevelopers/piston) window. The two-process model allows graphics rendering to be done
by the main thread (which is required on macOS) while still allowing users to simply write a `main()` function and not
worry about how to run their code in another thread.

Turtle code doesn't let go of control flow until the drawing has completed. This isn't a problem when Turtle code is running in one process and feeding commands to another process to be rendered to the screen, but browser-resident animation *really* wants to run code that runs a little bit at a time in an event loop via [`requestAnimationFrame()`](https://developer.mozilla.org/en-US/docs/Web/API/window/requestAnimationFrame). The browser doesn't update the page until control flow for your code has exited. This has some upsides for browsers (no concerns about thread safety when all code is time-sliced into one thread, for instance), but it in Turtle's case, it means that running in the main thread would update the screen exactly once: when the drawing was complete.

We do have one trick up our sleeve, though: [Web Workers](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API). These allow simple parallelism, but there is no good way to share memory with a worker and the main thread. (Aha, you say! What about [SharedArrayBuffer](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/SharedArrayBuffer)? Nope, it's been disabled due to [Spectre](https://spectreattack.com/).) What we *can* do, though, is send messages to the main thread with [`postMessage()`](https://developer.mozilla.org/en-US/docs/Web/API/Window/postMessage). This way, we can create a worker for Turtle, let it take over that thread entirely, and periodically post messages to be sent to the main thread when it wants to update the UI. 

## Drawing pixels

Let's consider a very simple Turtle program where the user wishes to draw a line. This boils down to `turtle.forward(10)` or equivalent. However, this isn't going to get all 10 pixels drawn all at once, because the turtle has a speed. So, it will calculate how long it should take to draw 10 pixels, and draw intermediate versions as time goes on. The same applies when drawing a polygon with a fill: the fill is partially applied at each point in the animation as the edges of the polygon are drawn. As parts of the drawing accumulate over time as the animation continues, the `Renderer` keeps track of all of them and re-draws them with its Piston `Graphics` backend every frame.

Naturally, the existing graphics stack that uses `piston_window` to draw graphics into a desktop app's window isn't going to seamlessly just start running in the browser and render into a `<canvas>`. So, where should the line be drawn about what is desktop-specific (and will therefore have to be reimplemented for the web)?

One option would be to reimplement the high level turtle interface (`forward()`, etc) in terms of Canvas drawing calls like [`beginPath()`](https://developer.mozilla.org/en-US/docs/Web/API/CanvasRenderingContext2D/beginPath). While it certainly can be done, it would require reimplementing a lot of code that animates, manages partially-completed polygons, draws the turtle itself, etc. The messages sent to the main thread would likely be a a JS array of closures which, when executed in the appropriate context, would update the `<canvas>` in the web page. Alternately, the canvas commands to run could be expressed as Objects with various fields, etc. This saves us from worrying about low level pixel wrangling, but at the cost of creating two parallel implementations of some nontrivial logic, one of which is quite difficult to test or reason about in isolation. It also means that as the drawing gets more complex over time, the list of commands to run grows as well, so performance will decay over time. It also involves many small allocations (and consequently more load on the JS GC).

Another approach would be to implement the Piston `Graphics` trait in terms of the Canvas API. This entails a lot less code duplication because things like animation, drawing the turtle "head" triangle, etc can all be re-used, but it still suffers from the performance decay over time, and has a fair amount of logic that needs to be built in JS, which I was trying to avoid. It's definitely much more practical than the previous option, though.

The approach I chose was to implement `Graphics` to write to an RGBA pixel buffer, which is easy to then display with a `<canvas>`. This has a minimal amount of JS needed, and has heavy, but consistent, allocation and message passing cost. Each frame requires one (large) allocation to copy the entire pixel buffer, which is then sent to the main thread and used to update the `<canvas>`. Obviously there are various ways to optimize this (only sending changed sub-sequences of pixels, etc) but for a first step, sending the entire buffer was functional and reliable. For Turtle's needs, `Graphics` only needs to draw triangles with a solid fill, which is pretty straightforward.

## Passing control flow to Turtle

Getting from `main()` to Turtle logic is actually a little complicated even when running on a desktop, as mentioned above. It is undesirable to have users deal with running turtle code in a different thread, so instead while Turtle is initializing it does its best to fork at the right time. If users are doing nontrivial initialization code of their own (e.g. parsing command line arguments), they have to be careful to let Turtle fork at the right point.

When running as wasm inside a Worker, initialization is convoluted in a different way. When writing a `main()` function for a normal OS, all the context you might want is available via environment variables, command line parameters, or of course making syscalls, but there's nothing like that for wasm unless you build it yourself. While you can declare a [start function](https://webassembly.github.io/spec/core/syntax/modules.html#start-function) for a wasm module, that doesn't solve our problem since this wasm code depends on knowledge from its environment, like how many pixels are in the canvas it will be drawing to.

While I could have doubled down on "call `Turtle::new()` at the right point and hope that it will figure it out" by having some wasm-only logic that called into the JS runtime to figure out what it needed, a small macro seemed like a tidier solution. The macro uses conditional compilation to do the right thing when it's compiled for the desktop or for wasm, with straightforward, global-free code in each case.

Before:

```rust

fn main() {
    // Must start by calling Turtle::new()
    // Secretly forks a child process and takes over stdout
    let mut turtle = Turtle::new();
    // write your drawing code here
}
```

After:

```rust
run_turtle!(|mut turtle| {
    // write your drawing code here
});
```

It's harder for users to screw up, and it opens the way to putting the Turtle logic in its own thread (instead of forking) when running on a desktop. This would allow users to use stdout on the desktop (currently claimed for communication with the child process), and the underpinnings could simply use shared memory instead of doing message passing via json over stdin/stdout. And, of course, it lets us get the right wasm-only initialization code in there too!

Anyway, with that in place, initializing the Worker that hosts the wasm is simple enough (see [worker.js](https://github.com/sunjay/turtle/pull/53/files#diff-8a5b5909f5b3ce1a25a1c499ab30da2c)). Following [Geoffroy Couprie's canvas example](https://www.hellorust.com/demos/canvas/index.html), we simply allocate a buffer with 4 bytes (RGBA) per pixel and call this function provided by the `run_turtle!` macro to kick things off:

```rust
#[allow(dead_code)]
#[cfg(feature = "canvas")]
#[no_mangle]
pub extern "C" fn web_turtle_start(pointer: *mut u8, width: usize, height: usize) {
    turtle::start_web(pointer, width, height, $f);
}
```

When a new frame has been rendered, the Rust logic calls into JS, which copies the contents of the pixel buffer and sends it to the main thread, which simply drops it into the `<canvas>`.

```javascript
const pixelArray = new Uint8ClampedArray(wasmTurtle.memory.buffer, pointer, byteSize);
const copy = Uint8ClampedArray.from(pixelArray);

postMessage({
    'type':   'updateCanvas',
    'pixels': copy
});
```

## Random number generation

When Rust is running on a normal OS, it uses `/dev/random` or equivalent internally to seed a PRNG. However, WebAssembly isn't an OS, so it doesn't offer a source of random numbers (or files, for that matter, in the same way that x86 asm doesn't have files). In practice, this means that Turtle's use of `rand::thread_rng()` dies with an "unreachable code" error. 

Fortunately, implementing our own `Rng` is pretty straightforward: the only required method is `next_u32()`, and that much we can do with JavaScript and the browser's PRNG (32 bit ints are pretty sane in JS; major int weirdness starts at 2^52). First, we'll need a JS function to make a random number between `0` and `u32::max_value()`, using the normal random range idiom:

```javascript
web_prng: () => {
    const min = 0;
    const max = 4294967295; // u32 max

    return Math.floor(Math.random() * (max - min + 1)) + min;
}
```

By making that available to the wasm module when it's initialized, we can then make that callable from Rust:

```rust
extern "C" {
    fn web_prng() -> u32;
}
```

And then use it in our `Rng` implementation:

```rust
pub struct WebRng;

impl ::rand::Rng for WebRng {
    fn next_u32(&mut self) -> u32 {
        unsafe {
            web_prng()
        }
    }
}
```

All that remained was to adjust how Turtle exposed random numbers so that it would use the appropriate `Rng` implementation on a desktop OS vs on wasm.

# Conclusion

Hopefully this has demonstrated that getting Rust code running in the browser via wasm is pretty achievable even for projects that aren't a drop-in fit for the browser's runtime model.

There were other problems to solve (like measuring time, or being able to println for debugging), but this post is already pretty long, so I'll refer you to the [pull request](https://github.com/sunjay/turtle/pull/53) if you want all the details. 
