# Tracing Internals

Who: The intended audience of this document is anyone that is interested in
understanding tracing in more detail, and anyone interested in contributing to
tracing.

What: The purpose of this document is to explain how tracing is implemented,
from a higher-level perspective than the API documentation.

Why: Currently, the API-level docs do provide detailed explanations, however, it
can be time-consuming for a newcomer to tracing to be able to understand how
different components within the library interact with each other, especially
across tracing crates. The goal is that the time it takes to onboard and
contribute to tracing can be shortened, and perhaps increase the number of
contributors to the project.

## Assumptions

This doc assumes that the reader already understands how tracing works from an
end-user perspective.

## High-level architecture

An easy way to split up tracing is compile-time behavior vs runtime-behavior.
During compile-time, the instrumentation macros expand, and statically define a
lot of the metadata that will ultimately be output during runtime.

During runtime, tracing calls the methods defined on the
[`Collect`](https://tracing-rs.netlify.app/tracing/trait.collect) trait, or
methods defined on the
[`Subscribe`](https://tracing-rs.netlify.app/tracing_subscriber/subscribe/trait.subscribe)
trait, and may include additional data into the trace (data that is not static
and therefore must be added during runtime).

## Compile-time + startup behavior

Users make use of the
[`span!`](https://github.com/tokio-rs/tracing/blob/55fdc750c2bf39d6d5635aca487b07daca990cfe/tracing/src/macros.rs#L20)
and `event!` macros. The macros expand to code that generates a callsite. A
callsite is what keeps track of that span/event's static Metadata, and also
keeps track of if that span/event is currently enabled (in tracing, this is
called `Interest`).

To get a better understanding of callsites, read the docs
[here](https://tracing-rs.netlify.app/tracing_core/callsite/index.html#registering-callsites).
The TL;DR is that when the user's program starts up and a new span/event is
encountered during runtime for the first time, the `Collect::register_callsite`
method is called to see if this callsite should be recorded or ignored.
`Interest` for all callsites is cached for the duration of the program (but is
re-evaluated if a new collector becomes active or is dropped).

## Runtime behavior

Setting aside the callsite registration process, what remains is the rest of the
methods defined on the `Collect` trait, for example, `new_span` and `exit`.
These methods are called[^1] again and again as code paths that hit those spans
are entered. A concrete implementation of the `Collect` needs to decide what
they want to do when these tracing events happen (here I'm using the word
"event" in a general sense). For example, the implementation may want to log to
stdout when a span is entered, and again when it's exited.

It's completely up to the implementor of the `Collect` impl what it is that they
want to do, and this will vary a lot by use-case. However, one common thing
across implementations is needing to store span data when the span is created,
and make use of it once the span has exited. The `Collect` trait doesn't provide
anything for this out-of-the-box, but the `Subscribe` trait does. The
`Subscribe` trait is a higher-level construct built on top of `Collect`. It
introduces the concept of a
[`Registry`](https://tracing-rs.netlify.app/tracing_subscriber/struct.registry),
which is a concrete implementation of the `Collect` trait. It does nothing
except store span data in an object pool. Then, multiple implementations of the
`Layer` trait can be composed together, and all can make use of the underlying
`Registry` to access span data on demand.

TL;DR
- This model allows retrieving span data event after the `new_span` method has
  already been called
- As opposed to using the `Collect` trait directly and only being able to use a
  single concrete implementation, the `Subscribe` trait allows multiple
  implementations to be used at the same time. This means that you can have one
  `Subscribe` impl that outputs a subset of tracing events to stdout, and
  another that sends another subset as JSON to a socket, for example.


[^1]: How does tracing know which Collect impl should be invoked? For this, we
    use the dispatcher. In the docs' own words, it "is responsible for
    forwarding trace data from the instrumentation points that generate it to
    the collector that collects it." The default collector does nothing, so in
    order for your traces to be used, you must tell the dispatcher to set a
    different collector, either globally, or for the current thread. For more
    details, please refer to the [dispatch module
    docs](https://tracing-rs.netlify.app/tracing_core/dispatch/index.html).