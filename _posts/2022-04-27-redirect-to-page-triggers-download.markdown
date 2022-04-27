---
title: Implications of x-content-type-options
layout: post
date: 2022-04-27 09:00:00 +0300
categories: safari frontend http-headers
---

Couple of days back I was debugging a weird behaviour by the Safari browser. It was triggered by this piece of code:

```
window.location.replace(redirectUrl)
```

When chrome, and firefox redirect the current tab to that url, latest safari (15.4) tried to download the resource the url was pointing to. Why was that? What was causing this behaviour? First I was thinking that this must be a bug in Safari, as all the other browser work as I expect, and I couldn't come up with anything that would explain this immediately. However, I decided to dig in a bit to better understand what was causing this. When I curled the resource the response was this:

```
< HTTP/2 200
< date: (redacted)
< content-length: (redacted)
< cf-ray: (redacted)
< cache-control: (redacted)
< set-cookie: (redacted)
< strict-transport-security: (redacted)
< cf-cache-status: (redacted)
< expect-ct: (redacted)
< ot-baggage-auth0-request-id: (redacted)
< ot-tracer-sampled: (redacted)
< ot-tracer-spanid: (redacted)
< ot-tracer-traceid: (redacted)
< x-auth0-requestid: (redacted)
< x-content-type-options: nosniff
< x-ratelimit-limit: (redacted)
< x-ratelimit-remaining: (redacted)
< x-ratelimit-reset: (redacted)
< set-cookie: (redacted)
< set-cookie: (redacted)
< server: (redacted)
< alt-svc: (redacted)
<
```

There's a lot of noise in the response above, but the interesting part there is that the response contains `x-content-type-options: nosniff` header that tells the client that it should respect whatever is defined in the `Content-Type` header. But what goes wrong with this response is, that it does not define the `Content-Type` header.

[MDN webdocs](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/X-Content-Type-Options) says:

> Starting with Firefox 72, top-level documents also avoid MIME sniffing (if Content-type is provided). This can cause HTML web pages to be downloaded instead of being rendered when they are served with a MIME type other than text/html. Make sure to set both headers correctly.

My interpretation of this is that if the `x-content-type-options` header is set, but `Content-Type` is not, then anything can happen, including that the browser tries to download the resource instead of showing it as a web page.
