# NGINX HTTP Early Hints (HTTP 103)

Allow nginx to generate its own 103 Early Hints response, when the official
release only supports forwarding them from upstreams.

Requires clients to send "Sec-Fetch-Dest: document" header, to avoid the risk
of misconfigurations accidentally sending 103 on subresources.

## Requirements

* nginx >= 1.29.0 (with 103 Early Hints plumbing)

## License

* BSD-2 to match upstream nginx (for easy conversion to a PR)

## Building (dynamic module)

As a dynamic module:

```sh
./configure --with-compat --add-dynamic-module=/path/to/ngx_http_early_hints
make modules
# copy objs/ngx_http_early_hints_module.so into your modules directory
```

Then load it:

```nginx
load_module modules/ngx_http_early_hints_module.so;
```

## Building (static module)

```sh
./configure --add-module=/path/to/ngx_http_early_hints
```

## Usage

```nginx
http {
    server {
        early_hints on;

        location / {
            # Add the resources you want to early hint
            early_hints_link "</css/app.css>; rel=preload; as=style";
            early_hints_link "</js/app.js>; rel=preload; as=script";

            fastcgi_pass unix:/run/php.sock;
        }
    }
}
```

### Dynamic hints via nginx variables / njs

`early_hints_link` accepts anything nginx can compile as a complex value, so
the hinted resources don't have to be static strings. This lets a `map`,
`js_set`, or any other variable-producing directive drive the value at
request time — for example when the set of resources to hint comes from an
external source (a CMS plugin, an API call recorded by njs, etc.) rather
than from static config:

```nginx
js_import hints from njs/hints.js;
js_set $page_hints hints.build_link_header;

location / {
    early_hints on;
    early_hints_link $page_hints;

    fastcgi_pass unix:/run/php.sock;
}
```

The njs function just needs to return the `Link` header value(s) as a
string (empty string to add nothing for that request):

```js
function build_link_header(r) {
    return '</css/app.css>; rel=preload; as=style';
}

export default { build_link_header };
```

### Fix: dedup flag was reset by internal redirects

Earlier revisions of this module tracked "have we already sent early hints
for this request" in the module's request context (`r->ctx`). nginx
memzeroes `r->ctx` on every internal redirect (`ngx_http_internal_redirect`,
named locations, `try_files`, error_page redispatch, etc.), so that flag
silently reset partway through requests that get internally redirected —
risking a second early-hints response, or hint headers leaking into the
final response.

The fix replaces the ctx flag with an indexed request variable
(`$early_hints_sent`, internal/not user-settable). `r->variables` is not
touched by internal redirects, so the flag now correctly survives across
them. Registering the variable still requires a `get_handler` (even though
it's never actually invoked — the handler reads/writes `r->variables[index]`
directly) purely so `ngx_http_variables_init_vars()` doesn't reject it as
"unknown" at config-load time.

## Security considerations

* **`early_hints_link` values are not sanitized.** Like `add_header`,
  `proxy_set_header` and other directives that accept a complex value,
  whatever the compiled value evaluates to is written directly into the
  `Link` header with no CRLF/control-character stripping. This is fine for
  static config, but if the value is driven by a variable fed from an
  external or semi-trusted source (a `map`, `js_set`, an upstream-reported
  value, etc.), that source is responsible for sanitizing/escaping the
  content before it reaches this directive — otherwise it's a header/response
  splitting vector into the 103 response.
* **Headers set by earlier request-processing phases are not excluded from
  the 103 response.** The handler runs at `NGX_HTTP_PRECONTENT_PHASE` and
  only strips the `Link` header(s) it added itself before falling through to
  the real response; any header already present in `r->headers_out.headers`
  from an earlier phase (e.g. `more_set_headers`, WAF modules, other njs
  handlers) is sent as part of the interim 103 response too, then sent again
  in the final response. Not a bug per se, but worth knowing if a header
  set earlier is sensitive or intended for the final response only.
* **`Sec-Fetch-Dest: document` is a hint, not an access control.** It is
  trivially spoofable by any non-browser client (curl, bots, scanners) — it
  only exists to avoid *accidentally* sending push-like hints to real
  browsers on subresource fetches, not to gate anything sensitive.
