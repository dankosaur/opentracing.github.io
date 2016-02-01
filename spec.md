---
layout: page
title: Semantic Specification
---
<div id="toc"></div>

## Concepts and Terminology


```
Parentage/causal relationships for spans from a single trace

        [Span A]  <--(the root span)
            |
     +------+------+
     |             |
 [Span B]      [Span C]
     |             |
 [Span D]      +---+-----+
               |         |
           [Span E]  [Span F]
                         |
                +--------+--------+
                |        |        |
            [Span G] [Span H] [Span I]

```

```
Temporal relationships for spans from a single trace

––|–––––––|–––––––|–––––––|–––––––|–––––––|–––––––|–––––––|–> time

 [Span A···················································]
   [Span B··············································]
      [Span D··········································]
    [Span C········································]
         [Span E···········]   [Span F········]
                                [Span G······]
                                [Span H··]
                                        [Span I·]
```

A **Trace** represents the potentially distributed, potentially concurrent data/execution path in a (potentially distributed, potentially concurrent) system. A Trace can be thought of as a tree of Spans. (See the ASCII diagrams above)

A **Span** represents a logical unit of work in the system that has a start time and a duration. Spans may be nested and ordered to model parent-child and casual relationships. Each span has an **operation name**, a presumably human-readable string which concisely names the work done by the span (e.g., an RPC method name, a function name, or the name of a subtask within a larger computation).

Every Span has zero or more **Logs**, each of which being a timestamped event name with an optional structured data payload of arbitrary size.

Every Span may also have zero or more key/value **Tags**, which do not have timestamps and simply annotate the spans.

A **TraceContext** is the current state of the distributed request, propagated both within processes and across the boundaries between them.  The TraceContext contains the IDs that tie the spans of a Trace together.  It also can contain user-defined **Trace Attributes**, which are user-defined key/value pairs propagated in-band with the rest of the TraceContext.


**Trace Attributes** are key/value pairs stored in a TraceContext and propagated _in-band_. Given a full-stack OpenTracing integration, Trace Attributes enable powerful functionality by transparently propagating arbitrary application data: for example, an end-user id may be added as a Trace Attribute in a mobile app, propagate (via the distributed tracing machinery) into the depths of a storage system, and recovered at the bottom of the stack to identify a particularly expensive SQL query.

Trace Attributes come with powerful _costs_ as well; since the attributes are propagated in-band, if they are too large or too numerous they may reduce system throughput or contribute to RPC latency.
**Trace Attributes** vs. **Span Tags**

* Structure: Trace Attributes exist in the propagated TraceContext; Span Tags live within an individual Span.
* Propagation: Trace Attributes are propagated in-band (i.e., alongside the actual application data) across process boundaries with the TraceContext. Span Tags are not propagated since they are not inherited from parent Span to child Span.
* Reporting: Spans are reported out-of-band from application data, so their tags are recorded out-of-band as well.  They should be used to label data for post-hoc analysis in the tracer. Trace Attributes are not necessarily recorded in reports to the tracing backend, though an implementation may elect to.

Also, Trace Attribute keys have a restricted format: implementations may wish to use them as HTTP header keys (or key suffixes), and of course HTTP headers are case insensitive. As such, Trace Attribute keys MUST match the regular expression `(?i:[a-z0-9][-a-z0-9]*)`, and – per the `?i:` – they are case-insensitive. That is, the trace attribute key must start with a letter or number, and the remaining characters must be letters, numbers, or hyphens.


## Platform-Independent API Semantics

OpenTracing supports a number of different platforms, and of course the per-platform APIs try to blend in to their surroundings and do as the Romans do. That said, each platform API must model a common set of semantics for the core tracing concepts described above. In this document we attempt to describe those concepts and semantics in a language- and platform-agnostic fashion.


### Tracer

The `Tracer` interface must have the following capabilities:

- Start a `Span` that has no parent, commonly referred to as a *root* `Span`, and provide new associated `TraceContext` **(py: `start_trace`, go: `StartTrace`)**
- Create and start a new child `Span` with a given operation name, based on provided `TraceContext`, updating the context appropriately **(py: `create_span`, go: `CreateSpan`)**

### Span

The `Span` interface must have the following capabilities:

- Finish the (already-started) `Span`.  Finish should be the last call made to any span instance, and to do otherwise leads to undefined behavior. **(py: `finish`, go: `Finish`)**
- Set a key:value tag on the `Span`. The key must be a `string`, and the value must be either a `string`, a `boolean`, or a numeric type. Behavior for other value types is undefined. If multiple values are set to the same key (i.e., in multiple calls), implementation behavior is also undefined. **(py: `set_tag`, go: `SetTag`)**
- Add a new log event to the `Span`, accepting an event name `string` and an optional structured payload argument. If specified, the payload argument may be of any type and arbitrary size, though implementations are not required to retain all payload arguments (or even all parts of all payload arguments). **(py: `log_event, log_event_with_payload`, go: `LogEvent, LogEventWithPayload`)**


### TraceContext

A TraceContext is responsible for:

a. providing a key/value data store that will be used by the tracer and end-user, 
b. encoding TraceContext instances in a manner suitable for propagation, and
c. for taking that encoded data and using it to generate TraceContext instances that are used to continue an overarching Trace.

Typically the propagation will take place across an RPC boundary, but message queues and other IPC mechanisms are also reasonable places pass TraceContext.

The encoded form of a propagated span is tracer implementation-specific, but should be divided into two components:

1. The core identifying information for the current Span, referred to as the "context snapshot" (for example, in Dapper this would include a `trace_id`, a `span_id`, and a bitmask representing the sampling status for the given trace)
1. Any Trace Attributes

The encoded data is separated in this way for a variety of reasons; the most important is to give OpenTracing users a portable way to opt out of Trace Attribute propagation entirely if they deem it a stability risk.

The propagation and trace-joining methods come in two flavors: binary and text.

- The *text* format is a platform-idiomatic map from (unicode) `string` to `string`; it is better-suited to pretty-printing and debugging
- The *binary* format is an opaque byte array and is better-suited to compact, high-performance encoding, decoding, and transmission

Note that there is no expectation that different tracing systems propagate `TraceContext` in compatible ways. Though OpenTracing is agnostic about the tracing implementation, for successful inter-process handoff it's essential that the processes on both sides of a propagation use the same tracing implementation.

The `TraceContext` interface must have the following capabilities:
- Propagating a TraceContext instance by encoding it as either...
  - a binary value, or
  - a unicode `string->string` map, where the keys are suitable for use in HTTP headers (see the notes about "trace attribute" keys above)
- Returning a new TraceContext instance that recreates stored/propagated TraceContext via either...
  - a binary value
  - a unicode `string->string` map
- A utility method which retrieves the previous span (eg. `get_current_span`)
- Read a Trace Attribute (`get_trace_attribute`)
- Set a Trace Attribute, which is a simple string:string pair. Note that newly-set trace attributes are only guaranteed to propagate to *future* children of the given `Span`. See the diagram below. **(py: `set_trace_attribute`, go: `SetTraceAttribute`)**

```
        [Span A]
            |
     +------+------+
     |             |
 [Span B]      [Span C]  <-- (1) ATTRIBUTE "X" IS SET ON SPAN C,
     |             |             BUT AFTER SPAN E ALREADY STARTED.
 [Span D]      +---+-----+
               |         |
           [Span E]  [Span F]   <-- (2) ATTRIBUTE "X" IS AVAILABLE
                         |              FOR RETRIEVAL BY SPAN F (A
                +--------+--------+     CHILD OF SPAN C), AS WELL
                |        |        |     AS SPANS G, H, AND I.
            [Span G] [Span H] [Span I]
```
