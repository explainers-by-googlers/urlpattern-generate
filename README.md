# **generate() method of URLPattern**

his proposal is an early design sketch by the Chrome Loading team to describe the problem below and solicit feedback on the proposed solution. It has not been approved to ship in Chrome.

## **Authors**

* Takashi Nakayama ([@azaika](https://github.com/azaika))  
* Shunya Shishido ([@sisidovski](https://github.com/sisidovski))

## **Participate**

* [WHATWG URLPattern Issue \#73](https://github.com/whatwg/urlpattern/issues/73)

## **What is this?**

We propose a new method of [URLPattern API](https://www.google.com/url?q=https://github.com/whatwg/urlpattern&sa=D&source=docs&ust=1748236573725324&usg=AOvVaw0fORTwpvELTe2LqCn-xB7_) to generate component strings from a `URLPattern` object. Currently, `URLPattern` primarily focuses on matching URLs against patterns and extracting information from matched URLs using methods like `test()` and `exec()`. There is no standard way to take a `URLPattern` object (like `/user/:id`) and a set of values (like `id=123`) and generate the corresponding URL or component string that matches that pattern (like `/user/123`). The proposed `generate()` method aims to fill this gap. This functionality makes it possible to perform reverse routing or URL construction, and IFT (Incremental Font Transfer) is one such use case needing such features ([W3C IFT Issue \#259](https://github.com/w3c/IFT/issues/259)).

## **Goals**

* Provide a standardized way to construct URLs from URLPattern objects.  
* Enable developers and other standard APIs to easily perform reverse routing or URL generation.

## **Non-Goals**

* Supporting complex template languages beyond what `URLPattern` already has.  
* Making `URLPattern` objects mutable.

## **Interface and example**

### WebIDL

```webidl
enum URLPatternComponent { "protocol", "username", "password", "hostname", "port", "pathname", "search", "hash" };
partial interface URLPattern {
  USVString generate(
      URLPatternComponent component,
      record<USVString, USVString> groups);
};
```

### Example

```javascript
// Example 1: Pattern of a fixed string
let pattern = new URLPattern({ protocol: 'https' });
pattern.generate('protocol', {}); // Returns: 'https'

// Example 2: Simple pattern with one named capture group
let pattern = new URLPattern({ pathname: '/user/:id' });
pattern.generate('pathname', { id: '123' }); // Returns: '/user/123'

// Example 3: Pattern with multiple named capture groups
pattern = new URLPattern({ pathname: '/product/:category/:id' });
pattern.generate('pathname', { category: 'electronics', id: '456' }); // Returns: '/product/electronics/456'

// Example 4: Pattern in hostname
pattern = new URLPattern('https://:subdomain.example.com/');
pattern.generate('hostname', { subdomain: 'store' }); // Returns: 'store.example.com'

// Example 5: TypeError when `groups` is not sufficient
let pattern = new URLPattern({ pathname: '/user/:id' });
pattern.generate('pathname', { }); // Raises TypeError

// Example 6: Ignore redundant input(s)
let pattern = new URLPattern({ pathname: '/user/:id' });
pattern.generate('pathname', { id: '123', more: 'input' }); // Returns: '/user/123'

// Example 7: Non-ascii inputs (see URL Encoding section below)
pattern = new URLPattern('https://:subdomain.example.com/:id');
pattern.generate('hostname', { subdomain: '🍅' }); // Returns: 'xn--fi8h.example.com'
pattern.generate('pathname', { id: '🍅' }); // Returns: '/%F0%9F%8D%85'
```

## **Design**

### URL encoding

`URLPattern` already has encoding rules for components \[[spec](https://urlpattern.spec.whatwg.org/#canon-encoding-callbacks)\], which invokes [basic url parser](https://url.spec.whatwg.org/#concept-basic-url-parser) in the process of compiling patterns. The `generate()` method follows this by encoding given strings in the same way.

### Full-Wildcard and RegExp parts are out of scope for now

Full-Wildcard are the parts represented as `*`. For example, the pathname pattern `/*` accepts `/arbitrary/numbers/of/sections`. RegExps are parts with constraints by user-provided RegExp like `/:id([0-9]+)`. To allow for thorough consideration, support for these features will be suspended initially \[[doc](https://docs.google.com/document/d/1hOG2SzMVab0fBJrV8J8w3wSWsbGZxDLyvjdhDu8WsT0)\]. The following example raises a TypeError.

```javascript
// Example: Pattern with Full-Wildcards
let pattern = new URLPattern({ protocol: '/*' });
pattern.generate('protocol', {}); // Raises TypeError
```

This decision does not hinder IFT because they do not plan to use these functionalities. The same goes for patterns with modifiers like `?`, `+`, or `*` because they are essentially RegExp(s).

## **Alternative considered**

### Record vs. Sequence

There is an option of making the second argument of `generate()` a sequence as follows:

```webidl
enum URLPatternComponent { ... };

partial interface URLPattern {
  USVString generate(
      URLPatternComponent component,
      sequence<USVString> groups);
};
```

In this case, developers pass a sequence like `[‘electronics’,‘456’]` for a pattern `/:category/:id`. However, we prefer record because records supersedes arrays in JavaScript and also align with how groups are represented in  `URLPatternComponentResult` \[[spec](https://urlpattern.spec.whatwg.org/#dictdef-urlpatterncomponentresult)\]. Discussions about this are made in [this doc](https://docs.google.com/document/d/1ca6geyHD40MfHkalgEv4AcBo9-rHK6CrAOxwcWor9WA/).

### Generating URL rather than component string

There may be needs for generating the complete URL, not component strings. Two API interfaces have been discussed to meet such needs:

```javascript
// Option 1: Separation by components, allowing duplicate names
let pattern = new URLPattern('https://:subdomain.example.com/:id');
pattern.generate({
	hostname: { subdomain: 'store' },
	pathname: { id: '123' }
}); // Returns: 'https://store.example.com/123'
new URLPattern('https://:id.example.com/:id').generate({ ... }); // OK

// Option 2: Unified group, throwing for duplicate names
let pattern = new URLPattern('https://:subdomain.example.com/:id');
pattern.generate({
	subdomain: 'store',
	id: '123'
}); // Returns: 'https://store.example.com/123'
new URLPattern('https://:id.example.com/:id').generate({ ... }); // Raises TypeError
```

However, we omit this functionality for the initial step since we don’t have a clear idea on which is good, and these two interfaces interfere with each other.

### Method name

The [path-to-regexp](https://github.com/pillarjs/path-to-regexp) library, the basis for standardizing URLPattern API, provides a similar functionality as [the compile() function](https://github.com/pillarjs/path-to-regexp). However, we adapt the name of `generate()` by following the [URLPattern issue](https://github.com/whatwg/urlpattern/issues/73). This name change is acceptable because we no longer need to maintain the same interface as the original library in URLPattern.  

