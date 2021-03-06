# cpp2sky

![cpp2sky test](https://github.com/SkyAPM/cpp2sky/workflows/cpp2sky%20test/badge.svg)

Distributed tracing and monitor SDK in CPP for Apache SkyWalking APM

## Build

#### Bazel

Download cpp2sky tarball with specified version.

```
http_archive(
  name = "com_github_skyapm_cpp2sky",
  sha256 = <SHA256>,
  urls = ["https://github.com/skyAPM/cpp2sky/archive/<VERSION>.tar.gz"],
)
```

Add interface definition and library to your project.

```
cc_binary(
  name = "example",
  srcs = ["example.cc"],
  deps = [
    "@com_github_skyapm_cpp2sky//cpp2sky:cpp2sky_interface",
    "@com_github_skyapm_cpp2sky//source:cpp2sky_lib"
  ],
)
```

## Basic usage

#### Config

cpp2sky provides simple configuration for tracer.
The detail information is described in [official protobuf definition](https://github.com/apache/skywalking-data-collect-protocol/blob/master/language-agent/Tracing.proto#L57-L67).

```cpp
#include <cpp2sky/config.pb.h>

int main() {
  using namespace cpp2sky;

  static const std::string service_name = "service_name";
  static const std::string instance_name = "instance_name";
  static const std::string oap_addr = "oap:12800";
  static const std::string token = "token";

  TracerConfig tracer_config;

  config.set_instance_name(instance_name);
  config.set_service_name(service_name);
  config.set_address(oap_addr);
  config.set_token(token);
}
```

#### Create tracer

After you constructed config, then setup tracer. Tracer supports gRPC reporter only, also TLS adopted gRPC reporter isn't available now.
TLS adoption and REST tracer will be supported in the future.

```cpp
TracerConfig tracer_config(oap_addr, token);
TracerPtr tracer = createInsecureGrpcTracer(tracer_config);
```

#### Fetch propagated span

cpp2sky supports only HTTP tracer now.
Tracing span will be delivered from `sw8` and `sw8-x` HTTP headers. For more detail, please visit [here](https://github.com/apache/skywalking/blob/08781b41a8255bcceebb3287364c81745a04bec6/docs/en/protocols/Skywalking-Cross-Process-Propagation-Headers-Protocol-v3.md)
Then, you can create propagated span object by decoding these items.

```cpp
SpanContextPtr parent_span = createSpanContext(parent);
```

#### Create span

First, you must create tracing context that holds all spans, then crete initial entry span.

```cpp
TracingContextPtr tracing_context = tracer->newContext();
TracingSpanPtr tracing_span = tracing_context->createEntrySpan();
```

After that, you can create another span to trace another workload, such as RPC to other services.
Note that you must have parent span to create secondary span. It will construct parent-child relation when analysis.

```cpp
TracingSpanPtr current_span = tracing_context->createExitSpan(current_span);
```

Alternative approach is RAII based one. It is used like below,

```cpp
{
  StartEntrySpan entry_span(tracing_context, "sample_op1");
  {
    StartExitSpan exit_span(tracing_context, entry_span.get(), "sample_op2");

    // something...
  }
}
```

#### Send segment to OAP

Note that TracingContext is unique pointer. So when you'd like to send data, you must move it and don't refer after sending,
to avoid undefined behavior.

```cpp
TracingContextPtr tracing_context = tracer->newContext();
TracingSpanPtr tracing_span = tracing_context->createEntrySpan();

tracing_span->startSpan("sample_workload");
tracing_span->endSpan();

tracer->report(std::move(tracing_context));
```

## Security

If you've found any security issues, please read [Security Reporting Process](https://github.com/SkyAPM/cpp2sky/blob/main/SECURITY.md) and take described steps.

## LICENSE

Apache 2.0 License. See [LICENSE](https://github.com/SkyAPM/cpp2sky/blob/main/LICENSE) for more detail.
