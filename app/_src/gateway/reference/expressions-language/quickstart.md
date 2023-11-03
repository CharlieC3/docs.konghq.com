---
title: Quickstart
content-type: reference
---

## The expressions language

The expressions language is a strongly typed [Domain Specific Language (DSL)](https://en.wikipedia.org/wiki/Domain-specific_language)
that allows user to define comparison operations on various input data.
Results of the comparisons can be combined with logical operations, allowing
complex routing logic to be written easily while ensuring good runtime
matching performance.

## Key concepts

* **Field:** Field contains value extracted from the incoming request. For example,
  the request path or the value of a header field. Field value could also be absent
  in some cases, absent field value will always cause predicate to yield `false`
  no matter the operator. Field always appears on the left hand side of the predicate.
* **Constant value:** Constant value is what the field is compared against based on the
  provided operator. Constant value always appears on the right hand side of the predicate.
* **Operator:** A operator defines the desired comparison action to be performed on the field
  against the provided constant value. Operator always appears in the middle of the predicate,
  between the field and constant value.
* **Predicate:** A predicate compares a field against a pre-defined value using the provided operator and
  returns `true` if the field passed the comparison, or `false` if not.
* **Route:** A route is one or more predicates, combined together with logical operators.
* **Router:** A router is a collection of routes which are all evaluated against incoming
  requests until a match could be found.
* **Priority:** A positive integer that defines the order of evaluation of the router.
  The bigger the priority, the sooner a route will be evaluated. In case of duplicate
  priority value between two routes within the same router, their order of evaluation is undefined.

![Structure of a predicate](/assets/images/products/gateway/reference/expressions-language/predicate.png)

## Examples (HTTP)
### Prefix based path matching

This is one of the most commonly used method for routing. Supposedly you would like
to match HTTP requests that have path starting with `/foo/bar`, you can write the following route:

```
http.path ^= "/foo/bar"
```

### Regex based path matching

If you prefer to match HTTP requests path against a regex, you can write the following route:

```
http.path ~ r#"/foo/bar/\d+"#
```

### Case insensitive path matching

If you want to ignore case when performing the path match, use the `lower()` modifier on the field
to ensure it always returns a lowercase value:

```
lower(http.path) == "/foo/bar"
```

This will now match requests with path of `/foo/bar`, `/FOO/bAr` and etc.

### Match by header value

If you want to match incoming requests by the value of header `X-Foo`, do the following:

```
http.headers.x_foo ~ r#"bar\d"#
```

In case of multiple header value for `X-Foo`, which could happen, if the client sends more than
one `X-Foo` header with different value, the above example will ensure **each** instance of the
value will match regex `r#"bar\d"#`. This is called "all" style matching, meaning each instance
of the field value must pass the comparison for the predicate to return `true` and is the
default behavior.

If this is not desired, it is also possible to turn on "any" style of matching which returns
`true` for the predicate as soon as any of the value passed the comparison:

```
any(http.headers.x_foo) ~ r#"bar\d"#
```

which will return `true` as soon as any value of `http.headers.x_foo` matches regex `r#"bar\d"#`.

Different transformations can be chained together, so the following is also a valid use case
that performs case-insensitive matching:

```
any(lower(http.headers.x_foo)) ~ r#"bar\d"#
```

### Regex captures

It is possible to define regex capture groups in any regex operation which will be made available
later for plugins to use. Currently this is only supported with the `http.path` field:

```
http.path ~ "/foo/(?P<component>.+)"
```

The matched value of `component` will be made available later to plugins such as
[Request Transformer Advanced](/hub/kong-inc/request-transformer-advanced/how-to/templates/).

## Examples (TCP, TLS, UDP)

### Match by source IP and destination port

```
net.src.ip in 192.168.1.0/24 && net.dst.port == 8080
```

This matches all client within the `192.168.1.0/24` subnet and destination port (listened by Kong)
is `8080`. IPv6 addresses are also supported.

### Match by SNI (for TLS routes)

```
tls.sni =^ ".example.com"
```

This matches all TLS connections with SNI ending with `.example.com`.

## How are routes executed

At runtime, Kong builds two separate routers for the HTTP and Stream (TCP, TLS, UDP) subsystem.
Routes are inserted into each router with appropriate `priority` field set. The router is
updated incrementally as configured routes change.

When a request/connection comes in, Kong looks at what field your configured routes require,
and supply the value of these fields to the router execution context, which is evaluated against
the configured routes in descendent order (higher number `priority` routes gets evaluated first).

As soon as a route yields a match, the router stops matching and the matched route is selected
for processing the current request/connection.

![Router matching flow](/assets/images/products/gateway/reference/expressions-language/router-matching-flow.png)

## What's next

Read the [Language References](/gateway/{{ page.kong_version }}/reference/expressions-language/language-references) page
to understand what operators and data types are available in the expressions language.