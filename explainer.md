# Explainer: Sanitizer

## Problems & Goals

Nearly all web apps deal with user input in some form, and need to sanitize
that input before passing it to the DOM or to scripts in order to avoid
unwanted side effects. The web platform provides no support for this, other
than general purpose string manipulation functions.

Arguably, this was initially a winning strategy since both web apps and attacks
were evolving, and browsers were slow to update, and hence the additional
flexibility of a regular JavaScript library was likely the superior choice.
Today, the risks and methods of web attacks are better understood; browsers
are updated frequently, [oftentimes more frequently than many web apps update
their dependencies](cloudflare-js-deps-study); and -- in addition to many, many
successes -- the [limits of building secure JavaScript libraries](dompurify-comment)
have also become more apparent.

[clouflare-js-deps-study]: https://blog.cloudflare.com/javascript-libraries-are-almost-never-updated/
[dompurify-comment]: https://www.usenix.org/sites/default/files/conference/protected-files/enigma16_slides_heiderich.pdf

In particular, the JavaScript language provides numerous ways for scripts to
modify their own runtime (e.g. change prototypes and thus modify the behaviour
of other, existing objects), which provides wonderful flexibility to both
developers and attackers alike. Unfortunately, this flexibility also makes it
hard to write safety-critical code in JavaScript that will still work as
intended in the face of script injection. A browser-native implementation can
provide a substantially stronger security boundary.

### Goal

Provide a **browser-maintained** "ever-green", **safe**, and **easy-to-use**
library for **user input sanitization** as part of the general **web platform**.

* **user input sanitization**: The basic functionality is to take a string,
  and turn it into strings that are safe to use and will not cause inadvertent
  execution of JavaScript.

* **browser-maintained**, "**ever-green**" / as part of the general
  **web platform**: The library is shipped with the browser, and will be
  updated alongside it as bugs or new attack vectors are found.

* **Safe** and **easy-to-use**: The API surface should be small, and the
  defaults should make sense across a wide range of use cases.

### Secondary Goals

* **Easy things should be easy.** This requires easy-to-use and safe defaults,
  and a small API surface for the common case.

* Cover a **reasonably wide range of base requirements**, but be open to more
  advanced use cases or future enhancements. This probably requires some sort
  of configuration or options, ideally in a way that both the developer and a
  security reviewer should be able to reason about them.

* Should be **integratable into other security mechanisms**, both browser
  built-ins and others.

* Be **poly-fillable**, although the polyfill would presumably have different
  security and performance properties.

### Non-goals

Force the use of this library, or any other enforcement mechanism. Some
applications will have sanitization requirements that are not easily met by
a general purpose library. These should continue to be able to use whichever
library or mechanism they prefer. However, the library should play well with
other enforcement mechanisms.

## Proposal

The basic proposal is to develop an API that learns from the
[DOMPurify](dompurify) library. In particular:

[dompurify]: https://github.com/cure53/DOMPurify

* The core API should be a single String-to-String method, plus a
  String-to-Fragment method. I.e., one per supported output type.

  * `saneStringFrom(DOMString value)` => `DOMString`

  * `saneFragmentFrom(DOMString value)` => `DocumentFragment`

* To support different use cases and to keep the API extensible, the
  sanitization should be configurable via an options dictionary.

* To make it easy to review and reason about sanitizer configs, there should
  be sanitizer instances for a given configuration.

  * DOMPurify supports per-call and a global "default" config. Global
    configuration state can be awkward to use when different dependencies
    have different ideas about what the global state should be. Likewise,
    per-call configs can be error prone and hard to reason about, since every
    call site might be a little different.

* There seem to be a handful of common use cases. There should be sensible
  default options for each of these.

### Proposed API

The basic API would be two calls: `.saneStringFrom(value)` to produce a string,
and `.saneFragmentFrom(value)` to produce a DocumentFragment. Sanitizers can
be constructed with a dictionary of options.

```
[Constructor(dictionary options)] interface Sanitizer {
  DOMString saneStringFrom(DOMstring);
  DocumentFragment saneFragmentFrom([StringContext=HTML] DOMString);
  readonly dictionary creation_options;
 };
```

Additionally, there should be a number of pre-configured Sanitizers available
for common cases, so that web authors can simply use sanitizers for cases
where they do not have special requirements.

*[I'm not sure the list below actually makes that much sense. And if it does,
then we'd need to add additional configuration options, because some of these
can't be created using the currently proposed options.]*

```
interface DefaultSanitizers {
  readonly Sanitizer string_only;  // string, without any elements
  readonly Sanitizer simple;  // HTML element content with known-good elements allowed
}
```

## Example usage

A simple web app wishes to take a string (say: a name) and display it on
the page:

```
document.getElementById("...").textContent = sanitizers.html.saneStringFrom(user_supplied_value);
```

The default sanitizers should probably be accessible via the document, or
somesuch. For this example we'll just pretend they're in the global namespace,
which... they wouldn't be.

```
string_only.saneStringFrom("a simple example") => "a simple example"
string_only.saneStringFrom("<b>bold</b> text") => "bold text"
simple.saneStringFrom("<b>bold</b> text") => "<b>bold</b> text"
simple.saneStringFrom("<b>bold</b><script>alert(4)</script> text") => "<b>bold</b> text"
```

### Proposed Options

*[I'm super unsure here. DOMPurify is (rather
full-featured)[dompurify-features], but I take it each feature in there has
found its use case. I suspect we should approach the same feature set, but
take liberties to reduce it if a use case is unclear. The important part
IMHO is to structure the API so that adding options will be easy, which is
well covered by the "options dictionary".]*

[dompurify-features]: https://github.com/cure53/DOMPurify/tree/master/demos#what-is-this

* Element tags (& attributes)
   * allowed tags  (example: ['b', 'em', 'a']
   * allowed attrs  (example: ['href'])
* URIs
   * allowed protocols (example: ['https'])
   * allowed origins  (example: ['https://www.example/', 'https://other.example:8888/']
   * allow regexp (example: '(https|http)://.*\.example/sample/path/.*')

### Open questions / Options / Extensions

This is a list of alternative options that should be considered:

* Should there be additional options about what the string-to-be-sanitized is
  used for?

  * DOMPurify has a options (e.g. WHOLE_DOCUMENT, FORCE_BODY) that specify more
    about the expected output that this proposal does.

* Should URIs be first class citizens in this proposal? (E.g. .saneURIFrom(.))

  * We already have options to restrict URIs, but presently there's no way to
    sanitize a URI by itself.

* Following DOMPurify, we have separate allow-lists for element names and
  attributes. Maybe this should be combined in a dictionary, to allow a
  certain attribute on one element but not on another?

* URI options, specifically URI allow regexp: There is a (currently
  unimplemented) proposal for a [URLPattern primitive)[urlpattern]. This might
  be a better way to specify allowed URIs than what is proposed above.

[urlpattern]: https://github.com/wanderview/service-worker-scope-pattern-matching/blob/master/explainer.md

## Appendix: FAQ

*[TODO. To date, nothing yet has been asked _frequently_.]*

## Appendix: API Feedback

This section intends to summarize & group feedback received on the API,
without judging whether the feedback should be incorporated into the main
document or not. Presumably, this section will remain an eternal
work-in-progress.

### Specify Contracts, Not Behaviour

The default sanitizers are a useful idea. However, as HTML evolves, and
presumably as attacks on browsers evolve, the exact actions that are needed
to be "safe" against any particular class of attacks will change over time.
Hence, the default sanitizers should specify a contract (maybe something like,
"no known XSS-vectors when inserted as a child into a non-script HTML element"),
rather than specific behaviour (such as: allow this specific set of elements).

### Allowed Tags, Allowed Attrs

The independent specification of attributes and elements allowed has been
criticized. Several real-world sanitizers I've been pointed to always decide
on which action to take based on the current tag as context.

The data models seem to be vaguely like:

```
    { tag-name } ✕ { attr-name | #text } ⇒ action
    action = { pass | drop } ∪ some-set-of-string-sanitizers
```

That is, for a combination of tag + attribute, or a tag's text children,
either pass or drop, or apply one of several basic string sanitizers. The
sanitizers might be different for regular text content, URI attributes, or
CSS attributes.

It might be possible to -- instead of listing particular tag/attribute
combinations -- to classify those into a handful of sanitization classes
(like: URI, CSS class, CSS content, regular text), but it's unclear whether
there's widespread agreement on what those classes are, or what members they
should have.

## Appendix: Differences to DOMPurify API

General API differences:

* DOMPurify has a built-in, modifiable default config. The sanitize call
  also accepts an options dictionary on every call. This proposal instead
  allows creation of sanitizer objects from an options dictionary, and offers
  several read-only sanitizers for easy-to-use defaults.

* DOMPurify uses options and a single method with different return types
  depending on the options set. Here, we instead have different methods for
  different return types.

* This has a more limited set of options.

* There is presently no equivalent to [hooks](dompurify-hooks).

[dompurify-hooks]: https://github.com/cure53/DOMPurify#hooks

List of DOMPurify options: (options collected [here](dompurify-options-1) and
[here](dompurify-options-2))

[dompurify-options-1]: https://github.com/cure53/DOMPurify#can-i-configure-dompurify
[dompurify-options-2]: https://github.com/cure53/DOMPurify/blob/master/src/purify.js

* Specify return value type: (Might also influence allowed tags):
   * RETURN_DOM  // returns HTMLBodyElement
   * RETURN_DOM_FRAGMENT  // returns  DocumentFragment
   * RETURN_DOM_IMPORT  // modifier to RETURN_DOM_FRAGMENT
   * RETURN_TRUSTED_TYPE  // return TrustedHTML
* Limit/extend which content is allowed:
   * ALLOWED_TAGS
   * ALLOWED_ATTR
   * FORBID_TAGS
   * FORBID_ATTR
   * ADD_TAGS
   * ADD_ATTR
   * ALLOW_DATA_ATTR
   * ALLOW_UNKNOWN_PROTOCOLS
   * ALLOWED_URI_REGEXP
   * ADD_URI_SAFE_ATTR
   * ALLOW_ARIA_ATTR
   * SAFE_FOR_TEMPLATES  // strip common template markers (like <%..%>)
   * SAFE_FOR_JQUERY
* Profile options, that enable several options at once:
   * USE_PROFILES  // predefined sets of ALLOWED_TAGS
* Miscellaneous::
   * WHOLE_DOCUMENT
   * SANITIZE_DOM
   * FORCE_BODY
   * KEEP_CONTENT
   * IN_PLACE

DOMPurify hooks: (without equivalent in this proposal)
* beforeSanitizeElements
* uponSanitizeElement (No 's' - called for every element)
* afterSanitizeElements
* beforeSanitizeAttributes
* uponSanitizeAttribute
* afterSanitizeAttributes
* beforeSanitizeShadowDOM
* uponSanitizeShadowNode
* afterSanitizeShadowDOM


## Appendix: Interaction with Trusted Types

It's the goal of the specification that the sanitizers can be used
independently, without requiring an additional API. Trusted Types can be
used as an enforcement mechanism, but doesn't provide sanitization
functionality. These two should work well together, but should not depend on
each other.

The most natural fit is to allow sanitizer objects to be used as callbacks
in the Trusted Type config. That requires a minor change to TT, to recognize
this argument type and to automatically use `sanitizer.bind(sanitizer)` for a
callback whenever a sanitizer instance is passed in.

The interaction between `.saneFragmentFrom(.)` and Trusted Types is awkward:
In a Trusted Types world, the transition from string to nodes should be
guarded by TT, and hence the input to this method needs to undergo a
Trusted Types check. The recommendation is to not use that method in TT
programs. If one were to change this API to be geared primarily towards
Trusted Types, one would presumably drop the `.saneFragmentFrom(.)` method
entirely.


