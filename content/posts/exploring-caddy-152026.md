---
title: Exploring Caddy - 1/5/2026
---

These are some of my general notes while reading Caddyserver's reverse proxy module, specifically looking at the round trip.

- `X-Forwarded-Proto` header keeps track of the original *proto* scheme (HTTP/HTTPS) in the request so that the backend can make an informed decision [1].
- Caddy makes a copy of the original request for a few reasons:
	- `http.Request` is not copy-safe, it has data structures within the object like maps that are pointers, which means making shallow copies is modifying the original as well.
	- Middleware may log specific information and relies on the original request, if you modify the original, then you are modifying the correctness of those logs.
	- Race conditions could occur if one goroutine is reading bytes and another is writing.
	- We set the correct headers and metadata _before_ any request actually made to an upstream, so that failure retries are idempotent.
- Requests are streams. Once the body is read from an upstream, it cannot be read again. Attempting to do so will return EOF. In order to allow retries in case of failures, Caddy will keep a buffer of the body that it can be reused across request retries [2],[3].
- The handler will attempt to proxy the request to an upstream until it succeeds with an exponential backoff timer.
- Some clients wait for a 1xx response before sending their request body, so the proxy needs to deliver that response before it can go forward and send the request. The way this is done is by adding a `ClientTrace` object with a `Got1xxResponse` function and passing that into the request context, which is then used by `RoundTrip(req)` [4].
- After the request successfully returns from the backend, we then handle the actually copying it back to the correct client.
- Caddy chooses to manually copy bytes from the source to it's original client because it gives them more clarity on write/read errors. By default `io.CopyBuffer` leaves you with an opaque error. There's an important distinction here for reverse proxies [5].
	- **Write** errors mean that something happened when writing the bytes back to the client. This indicates that the client likely disconnected. Logging the errors is not that important.
	- **Reading** errors means something happened on the upstream or in-flight process and that should be tracked.

#### References
1. https://github.com/caddyserver/caddy/blob/master/modules/caddyhttp/reverseproxy/reverseproxy.go#L818-L831
2. https://github.com/caddyserver/caddy/blob/master/modules/caddyhttp/reverseproxy/reverseproxy.go#L441-L453
3. https://github.com/caddyserver/caddy/blob/master/modules/caddyhttp/reverseproxy/reverseproxy.go#L468-L475
4. https://github.com/caddyserver/caddy/blob/master/modules/caddyhttp/reverseproxy/reverseproxy.go#L877-L901
5. https://github.com/caddyserver/caddy/blob/master/modules/caddyhttp/reverseproxy/streaming.go#L330-L336
