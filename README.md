# Go Reverse Proxy

A simple configuration-driven reverse proxy written in Go.

Using this as a testing ground for AI driven CI/CD

It has zero dependencies outside of the Go standard library.

## Configuration

Define a slice of type `Config`, with the minimum set of fields being: `Path` and `Upstream`.

Configuration is defined in the [routing/configuration](./routing/configuration.go) file.

Upstreams are defined in the [upstreams](./upstreams/upstreams.go) file.

## Example Config

Below we explain the actual [routing configuration](./routing/configuration.go) committed into this repo...

- [Proxy Request](#proxy-request)
- [Proxy Request using Regular Expression](#proxy-request-using-regular-expression)
- [Proxy Request with Modified Path](#proxy-request-with-modified-path)
- [Override with Modified Path](#override-with-modified-path)
- [Modified Path + Override with Modified Path](#modified-path--override-with-modified-path)
- [Override to Different Upstream](#override-to-different-upstream)
- [Query String Override](#query-string-override)
- [Query String Override with Regular Expression](#query-string-override-with-regular-expression)

### Proxy Request

```go
Config{
  Path:     "/anything/standard",
  Upstream: upstreams.HTTPBin,
}
```

### Requests

- `/anything/standard`

### Result

The request will be proxied straight through to the specified upstream without any modifications.

---

### Proxy Request using Regular Expression

```go
Config{
  Path:     "/anything/(?:foo|bar)$",
  Upstream: upstreams.HTTPBin,
}
```

### Requests

- `/anything/foo`
- `/anything/bar`

### Result

Both requests will be proxied straight through to the specified upstream without any modifications.

---

### Proxy Request with Modified Path

```go
Config{
  Path:       `/(?P<cap>foo\w{3})`,
  Upstream:   upstreams.HTTPBin,
  ModifyPath: "/anything/${cap}",
}
```

### Requests

- `/fooabc`
- `/fooxyz`

### Result

Both requests will be proxied through to the specified upstream but the path will be modified to include the captured information: `/anything/abc` and `/anything/xyz`.

---

### Override with Modified Path

```go
Config{
  Path:     "/(?P<start>anything)/(?P<cap>foobar)$",
  Upstream: upstreams.HTTPBin,
  Override: Override{
    Header:     "X-BF-Testing",
    Match:      "integralist",
    ModifyPath: "/anything/newthing${cap}",
  },
}
```

### Requests

- `/anything/foobar`
- `/anything/foobar` (+ HTTP Request Header `X-BF-Testing: integralist`)

### Result

The request will be proxied straight through to the specified upstream without any modifications. 

If the relevant request header is specified, then the request will be proxied through to the specified upstream but the path will be modified to include the captured information: `/anything/newthingfoobar`.

---

### Modified Path + Override with Modified Path

```go
Config{
  Path:       "/(?P<cap>double-checks)$",
  Upstream:   upstreams.HTTPBin,
  ModifyPath: "/anything/toplevel-modified-${cap}",
  Override: Override{
    Header:     "X-BF-Testing",
    Match:      "integralist",
    ModifyPath: "/anything/override-modified-${cap}",
  },
}
```

### Requests

- `/double-checks`
- `/double-checks` (+ HTTP Request Header `X-BF-Testing: integralist`)

### Result

The request will be proxied through to the specified upstream but the path will be modified to include the captured information: `/anything/toplevel-modified-double-checks`. 

If the relevant request header is specified, then the request will be proxied through to the specified upstream but the path will be modified to include the captured information: `/anything/override-modified-double-checks`.

---

### Override to Different Upstream

```go
Config{
  Path:     "/anything/(?P<cap>integralist)",
  Upstream: upstreams.HTTPBin,
  Override: Override{
    Header:     "X-BF-Testing",
    Match:      "integralist",
    ModifyPath: "/about",
    Upstream:   upstreams.Integralist,
  },
}
```

### Requests

- `/anything/integralist`
- `/anything/integralist` (+ HTTP Request Header `X-BF-Testing: integralist`)

### Result

The request will be proxied straight through to the specified upstream without any modifications. 

If the relevant request header is specified, then the request will be proxied through to a _different_ specified upstream and the path will also be modified.

> Note: although we use a named capture group, we don't actually utilise it anywhere in the rest of the configuration, so it's effectively a no-op.

---

### Query String Override

```go
Config{
  Path:     "/about",
  Upstream: upstreams.HTTPBin,
  Override: Override{
    Query:    "s",
    Match:    "integralist",
    Upstream: upstreams.Integralist,
  },
}
```

### Requests

- `/about`
- `/about?s=integralist`

### Result

The request will be proxied straight through to the specified upstream without any modifications. 

If the relevant query parameter is specified, then the request will be proxied through to a _different_ specified upstream.

---

### Query String Override with Regular Expression

```go
Config{
  Path:     "/anything/querytest",
  Upstream: upstreams.HTTPBin,
  Override: Override{
    Query:      "s",
    Match:      `integralist(?P<cap>\d{1,3})$`,
    MatchType:  "regex",
    ModifyPath: "/anything/newthing${cap}",
  },
}
```

### Requests

- `/anything/querytest`
- `/anything/querytest?s=integralist123`
- `/anything/querytest?s=integralist456`

### Result

The first request will be proxied straight through to the specified upstream without any modifications. 

If the relevant query parameter is specified, then the second and third requests will have their path modified to include the captured information: `/anything/newthing123` and `/anything/newthing456`. 

## Response Headers

We set the following response headers (not all will be set depending on the configuration):

```
X-Forwarded-Host
X-Origin-Host
X-Router-Upstream
X-Router-Upstream-OriginalHost
X-Router-Upstream-OriginalPath
X-Router-Upstream-OriginalPathModified
X-Router-Upstream-Override
X-Router-Upstream-OverrideHost
X-Router-Upstream-OverridePath
```

## Usage

```
make run
```

> Note: the application listens on port `9001`.

```
curl -v http://localhost:9001/some/path/you/configured
```

## Tests

```
make test
```

## Load Test

We use [`vegeta`](https://github.com/tsenart/vegeta) for load testing, so make sure you have that installed.

```
make stress
```

Example output:

```
Requests      [total, rate]            1500, 50.03
Duration      [total, attack, wait]    30.11237994s, 29.982166788s, 130.213152ms
Latencies     [mean, 50, 95, 99, max]  154.522948ms, 96.76258ms, 358.770472ms, 1.076826656s, 2.954136535s
Bytes In      [total, mean]            2039772, 1359.85
Bytes Out     [total, mean]            0, 0.00
Success       [ratio]                  100.00%
Status Codes  [code:count]             200:1500
Error Set:
```

## TODO

- Look at implementing thread pool processing on a host or server basis.
- Verify if DNS caching (or request memoization) would affect latency results?
- Review 301 redirect behaviour to be sure we don't need to handle that differently.
- Flesh out some unit tests (not just integration testing)
