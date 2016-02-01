---
layout: page
title: Instrumentation for Common Use Cases
---
<div id="toc"></div>

This page aims to illustrate common use cases that developers who instrument their applications and libraries with OpenTracing API need to deal with.

### Tracing a Function

```python
    def top_level_function():
        ctx, span1 = tracer.start_trace('top_level_function')
        try:
            . . . # business logic
        finally:
            span1.finish()
```

As a follow-up, suppose that as part of the business logic above we call another `function2` that we want to report on in our trace. In order to attach a Span reporting this function to the ongoing trace, we need to provide the Context that carries the trace state.. We discuss how it can be done later, for now let's assume we have a helper function `get_current_context` for that:

```python
    def function2():
        span2 = tracer.create_span(get_current_context(), 'function2')
        try:
            . . . # business logic
        finally:
            if span2:
                span2.finish()
```

We assume that, for whatever reason, the developer does not want to start a new trace in this function if one hasn't been started by the caller already.  This is accounted for by using `create_span`, which can deal with `get_current_context` potentially returning `None`.

These two examples are intentionally naive. Usually developers will not want to pollute their business functions directly with tracing code, but use other means like a [function decorator in Python](https://github.com/uber-common/opentracing-python-instrumentation/blob/master/opentracing_instrumentation/local_span.py#L59):

```python
    @traced_function
    def top_level_function():
        ... # business logic
```

### Tracing Server Endpoints

When a server wants to trace execution of a request, it generally needs to go through these steps:

  1. Create a new Span, either attached to an upstream trace by a Context that's been propagated alongside the incoming request (in case the trace has already been started by the client), or a new trace Context if no such propagated Context could be found.
  1. Store the newly created Context in a way that can be propagated throughout the application, either by application code, or by the RPC framework.  (eg. thread-local storage, manual propagation)
  1. Finally, close the Span using `span.finish()` when the server has finished processing the request.

#### Joining to a Trace from an Incoming Request

Let's assume that we have an HTTP server, and the Span is propagated from the client via HTTP headers, accessible via `request.headers`:

```python
    ctx = context.from_string(request.headers)
    span = tracer.join_trace(operation_name=operation, context=ctx)
```

Assuming the same Tracer implementation is used upstream in the client, the Tracer object knows which Context fields it needs to read in order to reconstruct the trace.

The `operation` above refers to the name the server wants to give to the Span. For example, if the HTTP request was a POST against `/save_user/123`, the operation name can be set to `post:/save_user/`. OpenTracing API does not dictate how applications name the spans.

#### Joining to or Starting a Trace from an Incoming Request

The `span` object above can be `None` if the Tracer did not find relevant headers in the incoming request: presumably because the client did not send them. In this case the server needs to start a brand new trace.  To do this, use `start_trace` instead: in the presence of a Context, a trace is continued, and in the absence of one, a new Context and root Span are both created.

```python
    ctx = context.from_string(request.headers)
    span = tracer.start_trace(operation_name=operation, context=ctx)
    span.set_tag('http.method', request.method)
    span.set_tag('http.url', request.full_url)
```

The `set_tag` calls are examples of recording additional information in the Span about the request.

#### In-Process Request Context Propagation

Request context propagation refers to application's ability to associate a certain Context with each request such that this context is accessible in all other layers of the application within the same process. The primary role of Context for tracing is transporting the current trace state as required by the tracer: for instance, the global ID of a request trace and the ID of the last (parent) Span.  However, with Trace Attributes it can be used to provide application layers with access to request-specific values such as the identity of the end user, authorization tokens, and the request's deadline.

Implementation of in-process Context propagation is currently outside the scope of the OpenTracing API, but it is worth mentioning them here to better understand the following sections. There are two commonly used techniques of context propagation:

##### Implicit Propagation

In implicit propagation techniques the context is stored in platform-specific storage that allows it to be retrieved from any place in the application. Often used by RPC frameworks by utilizing such mechanisms as thread-local or continuation-local storage, or even global variables (in case of single-threaded processes).

The downside of this approach is that it almost always has a performance penalty, and in platforms like Go that do not support thread-local storage implicit propagation is nearly impossible to implement.

##### Explicit Propagation

In explicit propagation techniques the application code is structured to pass around a certain _context_ object:

```go

    func HandleHttp(w http.ResponseWriter, req *http.Request) {
        ctx := context.Background()
        ...
        BusinessFunction1(ctx, arg1, ...)
    }

    func BusinessFunction1(ctx context.Context, arg1...) {
        ...
        BusinessFunction2(ctx, arg1, ...)
    }

    func BusinessFunction2(ctx context.Context, arg1...) {
        span := SpanFromContext(ctx)
        span.StartChild(...)
        ...
    }
```

The downside of explicit context propagation is that it leaks what could be considered an infrastructure concern into the application code. This [Go blog post](https://blog.golang.org/context) provides an in-depth overview and justification of this approach.

### Tracing Client Calls and Distributed Context Propgataion

When an application acts as an RPC client, it is expected to start a new tracing Span before making an outgoing request, which will represent the client timings of the request.  Additionally, RPC instrumentation is tasked with propagating the Context along with that request. The following example shows how it can be done for an HTTP request. 

```python
    def traced_request(request, operation, http_client):
        # retrieve current span from propagated request context
        ctx = get_current_context()
        span = tracer.create_span(operation_name=operation, context=ctx)
        span.set_tag('http.url', request.full_url)

        # Propagate the Context via HTTP request headers
        header_ctx = ctx.to_string()

        for key, value in header_ctx.iteritems():
            request.add_header(key, value)

        # define a callback where we can finish the span 
        def on_done(future):
            if future.exception():
                span.log('exception', payload=exception)
            else:
                span.set_tag('http.status_code', future.result().status_code)
            span.finish()

        try:
            future = http_client.execute(request)
            future.add_done_callback(on_done)
            return future
        except Exception e:
            span.log_event_with_payload('exception', e)
            span.finish()
            raise
```

Notes on this example:
  * The `get_current_context()` function is not a part of the OpenTracing API. It is meant to represent some util method of retrieving the current TraceContext, which can be achieved either implicitly or explicitly as discussed above.
  * We assume the HTTP client is asynchronous, so it returns a Future, and we need to add an on-completion callback to be able to finish the current child Span.  The synchronous case is trivially similar for instrumentation.
  * If the HTTP client returns a future with exception, we log the exception to the Span with `log` method.
  * Because the HTTP client may throw an exception even before returning a Future, we use a try/catch block to finish the Span in all circumstances, to ensure it is reported and avoid leaking resources.


### Using Trace Attributes

The client and server examples above propagated the TraceContext over the wire, including both Tracer-specific state related to the current request as well as user-added Trace Attributes. The client may use the Trace Attributes to pass additional data to the server and any other downstream server it might call.

```python

    # client side
    ctx.set_attribute('auth-token', '.....')

    ...

    # server side (one or more levels down from the client)
    token = ctx.get_attribute('auth-token')
```

### Logging Events

We have already used `log` in the client Span use case. Events can be logged without a payload, and not just where the Span is being created / finished. For example, the application may record a cache miss event in the middle of execution, as long as it can get access to the current Span:

```python

    # if the Span isn't handy, grab it
    span = get_current_context().get_current_span()

    # add an event
    span.log_event('cache-miss')
```

The tracer automatically records a timestamp of the event, in contrast with tags which are labels applying to the entire Span. It is also possible to associate an externally provided timestamp with the event, e.g. see [Log (Go)](https://github.com/opentracing/opentracing-go/blob/ca5c92cf/span.go#L53).

### Recording Spans With External Timestamps

There are scenarios when it is impractical to incorporate an OpenTracing compatible tracer into a service, for various reasons. For example, a user may have a log file of what's essentially Span data coming from a black-box process (e.g. HAProxy). In order to get the data into an OpenTracing-compatible system, the API needs a way to record spans with extrenally captured timestamps. The current version of API does not have that capability: see [issue #20](https://github.com/opentracing/opentracing.github.io/issues/20).

### Setting "Debug" Mode

`XXX: we don't have an API for this anymore.`

> Most tracing systems apply sampling to minimize the amount of trace data sent to the system.  Sometimes developers want to have a way to ensure that a particular trace is going to be recorded (sampled) by the tracing system, e.g. by including a special parameter in the HTTP request, like `debug=true`. The OpenTracing API does not have any insight into sampling techniques used by the implementation, so there is no explicit API to force it. However, the implementations are advised to recognized the `debug` Trace Attribute and take measures to record the Span. In order to pass this attribute to tracing systems that rely on pre-trace sampling, the following approach can be used:
> 
> ```python
> 
>     if request.get('debug'):
>         trace_context = tracer.new_root_trace_context()
>         trace_context.set_attribute('debug', True)
>         span = tracer.start_span_with_context(
>             operation_name=operation, 
>             trace_context=trace_context)
> ```
