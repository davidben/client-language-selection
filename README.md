# Client-side Language Selection

## Introduction

User agents specify the user’s preferred languages in the `Accept-Language` HTTP header and the `navigator.language` JavaScript property. This header leaks user entropy to all sites, even to sites that do not use it. Multilingual users who configure full language preferences are particularly affected. We wish to reduce fingerprinting capabilities of the web platform, but, at the same time, web content is not useful if users cannot read it, so we wish to preserve the language negotiation functionality. This proposal is an alternative to the [`Lang` Client Hint](https://github.com/WICG/lang-client-hint).

## Goals

This proposal aims to preserve the web’s language negotiation functional while reducing information sent to sites. Specifically, sites should learn the language to display the page in and no more.

## Non-goals

It is not a goal to hide all language information entirely. In particular, it is unrealistic on the web to support language negotiation without the page learning which language to use.

## HTTP representation variants

The existing [HTTP representation variants](https://tools.ietf.org/html/draft-ietf-httpbis-variants-06) proposal introduces `Variants` and `Variant-Key` headers to improve cache performance. Suppose a cache observes the following exchange:

```
   GET /foo HTTP/1.1
   Host: example.com
   Accept-Language: fr, ja;q=0.9

   HTTP/1.1 200 OK
   Content-Language: fr
   Vary: Accept-Language
   Variants: Accept-Language=(es fr)
   Variant-Key: (fr)
```

`Vary` says the resource depends on `Accept-Language`. A priori, the cache requires an exact match to reuse the resource. `Variants` and `Variant-Key` give the cache more information, so it knows a request for `en, fr;q=0.9` can also reuse it because the resource is not available in English.

## Client-side language selection

We can reuse variants information to move language selection to the client. The browser caches what language to request for each origin. (As with other state, this must be based on the the top-level site to avoid cross-site tracking.)

Initially, the cache is empty and the browser sends no `Accept-Language` header. The server then responds with some language, perhaps its default language or a heuristic based on IP geolocation.

```
   GET /foo HTTP/1.1
   Host: example.com
   # The true Accept-Language is ja, fr;q=0.9 (not sent)

   HTTP/1.1 200 OK
   Content-Language: es
   Vary: Accept-Language
   Variants: Accept-Language=(es fr)
   Variant-Key: (es)
```

The browser evaluates the `Variants` header against the true language preferences. If this was correct, it uses the response as-is. Here, however, the correct language was French. It remembers this (to save roundtrips in the future) and retries the request with new headers:

```
   GET /foo HTTP/1.1
   Host: example.com
   Accept-Language: fr
   # The true Accept-Language is ja, fr;q=0.9 (not sent)

   HTTP/1.1 200 OK
   Content-Language: fr
   Vary: Accept-Language
   Variants: Accept-Language=(es fr)
   Variant-Key: (fr)
```

(To avoid infinite loops, the browser should only retry once for each request and never retry on POSTs.)

## Detailed design discussion

### Server language changes

Suppose the above page is later translated to Japanese. The browser then receives:

```
   GET /foo HTTP/1.1
   Host: example.com
   Accept-Language: fr
   # The true Accept-Language is ja, fr;q=0.9 (not sent)

   HTTP/1.1 200 OK
   Content-Language: fr
   Vary: Accept-Language
   Variants: Accept-Language=(es fr ja)
   Variant-Key: (fr)
```

Now the browser finds another representation would be preferable. It updates the cached language and retries the request.

```
   GET /foo HTTP/1.1
   Host: example.com
   Accept-Language: ja
   # The true Accept-Language is ja, fr;q=0.9 (not sent)

   HTTP/1.1 200 OK
   Content-Language: ja
   Vary: Accept-Language
   Variants: Accept-Language=(es fr ja)
   Variant-Key: (ja)
```

### Abuse

A server can learn the full language preferences over multiple page loads, by lying to the browser about the available languages and the selected language. This is abuse of the API and should be mitigated by the browser. Some possibilities:

* Rate limit language changes or distinct language lists from a site.
* Detect the page’s language from content and, if it does not match the advertised language, skip the retry. (Some browsers already implement such detection for translation features.)
* Deduct unique language lists and/or selected languages from the site’s [Privacy Budget](https://github.com/bslassey/privacy-budget).
* When mitigations are applied, display a UI indicator such as “This page is also available in _LANGUAGE_ \[Switch\]”. If the page is already in that language or a more preferred one, the user will be unlikely to switch. If the mitigation was misapplied, the user has an opportunity to fix it.

### Incremental deployment

Note this retry can correct the language selection for _any_ client `Accept-Language` header. Header redactions can thus be deployed incrementally to balance privacy benefits against compatibility and performance.

The browser may initially implement this retry while still sending the first language. As the variants headers are deployed, the browser can then remove locale information, or remove the default header entirely. Or perhaps the browser may decide always sending the first language as an end state is a reasonable tradeoff for performance.

The browser might also consider other variations. If variants support is too low, it could reveal the full `Accept-Language` header after receiving a `Variants`-less `Vary: Accept-Language`. This is more entropy than the full proposal, but it matches [Client Hint proposal](https://github.com/WICG/lang-client-hint) without requiring server changes. Or perhaps it reveals the most preferred language on `Vary` and requires `Variants` to incorporate secondary languages.

### JavaScript

This proposal describes the HTTP negotiation. The page may also use `navigator.languages` to select the language in JavaScript. Depending on how much flexibility to offer sites, there are a number of options:

* Leave `navigator.languages` alone but severely penalize it in the [Private Budget](https://github.com/bslassey/privacy-budget).
* As with HTTP, require the page declare available languages in a `<meta>` tag, evaluate the language in the browser, and reveal only one language in `navigator.languages`. Note there is already an HTML `lang` attribute to declare a page’s language.
* Introduce a JavaScript API such as `navigator.selectLanguage(availableLanguages)` for the site to trigger language negotiation. Note this allows authors to query it multiple times, whereas a `<meta>` tag implies it’s a fixed property of the page and makes it harder to probe multiple lists. The tag is thus preferable from a privacy perspective.

As with the header negotiation, repeat queries may reveal the full language list, so browsers should apply similar mitigations as the header.

## Considered alternatives

### Client hints

The [`Lang` Client Hint proposal](https://github.com/WICG/lang-client-hint) moves the language to a new header, `Sec-CH-Lang`, that is only sent when the site requests it. This would allow incorporating it into mechanisms like [Privacy Budget](https://github.com/bslassey/privacy-budget). This proposal improves on the Client Hint version:

* The `Lang` Client Hint uses a new header, so web developers must rewrite their existing language negotiation logic. This proposal keeps the existing logic intact. Web developers only need to add `Variants` and `Variant-Key` headers, which also give caching benefits.
* The Client Hint only reduces information sent to single-language sites. This proposal additionally reduces the information sent to sites that support language negotiation.
* Switching a leak from passive to active and relying on Privacy Budget assumes the leak is one of many important but uncommon use cases. Such use cases need to be uncommon for legitimate behavior to be unlikely to exceed the budget. However, ensuring users can read the pages they load is important. The fingerprinting solution shouldn’t design against that goal by assuming it is uncommon.

### Reducing language granularity

Safari limits the `Accept-Language` header to the [first language](https://bugs.webkit.org/show_bug.cgi?id=3510#c27). This reduces the entropy in the header, but it means multilingual users are unable to express their preferences. As noted in the discussion on incremental deployment, this proposal works in conjunction with such techniques to preserve functionality.
