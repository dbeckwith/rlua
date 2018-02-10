# rlua -- High level bindings between Rust and Lua

[![Build Status](https://travis-ci.org/chucklefish/rlua.svg?branch=master)](https://travis-ci.org/chucklefish/rlua)

[API Documentation](https://docs.rs/rlua)

[Examples](examples/examples.rs)

This library is a high level interface between Rust and Lua.  Its major goal is
to expose as easy to use, practical, and flexible of an API between Rust and Lua
as possible, while also being completely safe.

`rlua` is designed around "registry handles" to values inside the Lua state.
This means that when you get a type like `rlua::Table` or `rlua::Function` in
Rust, what you actually hold is an integer key into the Lua registry.  This is
different from the bare Lua C API, where you create tables / functions on the
Lua stack and must be aware of their stack location.  This is also similar to
how other Lua bindings systems like
[Selene](https://github.com/jeremyong/Selene) for C++ work, but it means that
using `rlua` may be slightly slower than what you could conceivably write using
the C API.  The reasons for this design are safety and flexibility, and to
prevent the user of `rlua` from having to be aware of the Lua stack at all.

There are currently a few missing pieces of this API:

  * Security limits on Lua code such as total instruction limits / memory limits
    and control over which potentially dangerous libraries (e.g. io) are
    available to scripts.
  * Lua profiling support
  * "Context" or "Sandboxing" support.  There should be the ability to set the
    `_ENV` upvalue of a loaded chunk to a table other than `_G`, so that you can
    have different environments for different loaded chunks.
  * Benchmarks, and quantifying performance differences with what you would
    might write in C.

Additionally, there are ways I would like to change this API, once support lands
in rustc.  For example:

  * Currently, variadics are handled entirely with tuples and traits implemented
    by macro for tuples up to size 12, it would be great if this was replaced
    with real variadic generics when this is available in Rust.

It is also worth it to list some non-goals for the project:

  * Be a perfect zero cost wrapper over the Lua C API
  * Allow the user to do absolutely everything that the Lua C API might allow

## API stability

This library is very much Work In Progress, so there is a some API churn.
Currently, it follows a pre-1.0 semver, so all API changes should be accompanied
by 0.x version bumps.

## Safety and panics

The goal of this library is complete safety, it should not be possible to cause
undefined behavior whatsoever with the API, even in edge cases.  There is,
however, QUITE a lot of unsafe code in this crate, and I would call the current
safety level of the crate "Work In Progress".  Still, UB is considered the most
serious kind of bug, so if you find the ability to cause UB with this API *at
all*, please file a bug report.

Another goal of this library is complete protection from panics and aborts.
Currently, it should not be possible for a script to trigger a panic or abort
(with some important caveats described below).  Similarly to the safety goal,
there ARE several internal panics and even aborts in `rlua` source, but they
should not be possible to trigger, and if you trigger them this should be
considered a bug.

There are some caveats to the panic / abort guarantee, however:

  * `rlua` reserves the right to panic on API usage errors.  Currently, the only
    time this will happen is when passed a registry handle type from a different
    main Lua state.
  * Currently, there are no memory or execution limits on scripts, so untrusted
    scripts can always at minimum infinite loop or allocate arbitrary amounts of
    memory.
  * The internal Lua allocator is set to use `realloc` from `libc`, but it is
    wrapped in such a way that OOM errors are guaranteed to *abort*.  This is
    not currently such a big deal, as this matches the behavior of Rust itself.
    This allows the internals of `rlua` to, in certain cases, call 'm' Lua C API
    functions with the garbage collector disabled and know that these cannot
    error.  Eventually, `rlua` will support memory limits on scripts, and those
    memory limits will cause regular memory errors rather than OOM aborts.

Yet another goal of the library is to, in all cases, safely handle panics
generated by Rust callbacks.  Panic unwinds in Rust callbacks should currently
be handled correctly -- the unwind is caught and carried across the Lua API
boundary as a regular Lua error in a way that prevents Lua from catching it.
This is done by overriding the normal Lua 'pcall' and 'xpcall' with custom
versions that cannot catch errors that are actually from Rust panics, and by
handling panic errors on the receiving Rust side by resuming the panic.

In summary, here is a list of `rlua` behaviors that should be considered a bug.
If you encounter them, a bug report would be very welcome:

  * If your code panics / aborts with a message that contains the string "rlua
    internal error", this is a bug.
  * The above is true even for the internal panic about running out of stack
    space!  There are a few ways to generate normal script errors by running out
    of stack, but if you encounter a *panic* based on running out of stack, this
    is a bug.
  * When the internal version of Lua is built using the `gcc` crate, and
    `cfg!(debug_assertions)` is true, Lua is built with the `LUA_USE_APICHECK`
    define set.  Any abort caused by this internal Lua API checking is
    *absolutely* a bug, particularly because without `LUA_USE_APICHECK` it would
    generally be unsafe.
  * Lua C API errors are handled by lonjmp.  *ALL* instances where the Lua C API
    would longjmp should be protected from Rust, except in a few cases of
    internal callbacks where there are only Copy types on the stack.  If you
    detect that `rlua` is triggering a longjmp over Rust stack frames (other
    than the internal ones), this is a bug!  (NOTE: I believe it is still an
    open question whether technically Rust allows longjmp over Rust stack frames
    *at all*, even if there are only Copy types on the stack.  Currently `rlua`
    uses this to avoid having to write a lot of messy C shims.  It *currently*
    works fine, and it is difficult to imagine how it would ever NOT work, but
    what is and isn't UB in unsafe Rust is not precisely specified.)
