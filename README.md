# JSON REFERENCE v0.4.0 SPECIFICATION
 JSON Reference decodes JSON documents, in a special way, to enable references in the native decoded document not supported by pure [JSON](https://www.json.org/json-en.html) decoders. By "JSON document" we mean a Unicode string of MIME type `application/json` not it's native language dependent *decoded document*.

JSON Reference v0.4.0 succeeds [JSON Reference v0.3.0](https://tools.ietf.org/html/draft-pbryan-zyp-json-ref-03) and is *not* backwards compatible. JSON Reference v0.4.0 requires [JSON Pointer](https://tools.ietf.org/html/draft-ietf-appsawg-json-pointer-04).

Every valid JSON Reference document is a valid JSON document. JSON Reference makes use of two special JSON object properties meaningful to JSON Reference, but not meaningful to JSON, to achieve this: `$ref` and `$id`. They work like this:

**$id**

  - The `$id` property is optional on any object.
  - The `$id` property is analogous to the `name` or `id` [HTML attribute](https://www.w3.org/TR/html401/types.html#type-id). The `$id` keyword value MUST begin with a letter ([A-Za-z]) and may be followed by any number of letters, digits ([0-9]), hyphens ("-"), underscores ("_"), colons (":"), and periods (".")".
  - `$id` values are case sensitive.
  - The `$id` property values MUST be unique within a *JSON document*.
  - Implementations MUST raise an error if duplicate `$id` values are present.

**$ref**

  - Objects with a `$ref` property, such as `{ "$ref": <URI> }` are *entirely replaced* by the value pointed to by the `<URI>`. This value is called the *replacement-value*.
  - All other properties of an object containing a `$ref` key *are ignored*.
  - `<URI>` MUST be a valid URI. Implementations must attempt to resolve the `<URI>` to a *replacement-value* as described below.
  - The fragment component of `<URI>` MUST be empty, a valid `$id` keyword value, or a valid [JSON pointer][json-pointer].
  - If `<URI>` is consists only of a fragment identifier part it MUST be resolved against the current document. The *replacement-value* is the object with the given  `$id` keyword or value pointed to via the JSON Pointer in the case that the fragment is a valid JSON Pointer (pointers a syntactically distinct from valid `$id` names so there is no ambiguity).
  - If `<URI>` is not just a fragment, implementations SHOULD resolve relative URIs against a base URI, then use the value identified by this URI as the *replacement-value*. How the base URI is determined is beyond the scope of this specification.
  - Implementations MUST NOT attempt to load remote resources or any external resource by default. In general implementations should make it clear how and where external values may be retrieved from and require the client to explicitly enable or configure such.
  - If a loaded resource refers to all or part of a JSON document, that JSON document MUST be processed as a standalone document as described above, and NOT considered part of, or substituted into, the referring source JSON document.

**references and replacement-values**

  - [JSON](https://www.json.org/json-en.html) is built on two structures or container types:
    - objects: "A collection of name/value pairs. In various languages, this is realized as an object, record, struct, dictionary, hash table, keyed list, or associative array."
    - arrays: "An ordered list of values. In most languages, this is realized as an array, vector, list, or sequence."
  - JSON also supports five other types: string, number, true, false, null.
  - JSON Reference implementations MUST replace references to the same arrays and objects with "native" references.
  - JSON Reference implementations MAY replace references to other types with references or with copies of the replacement-value.

## WHAT JSON REFERENCE IS NOT
JSON Reference is not JSON Merge or JSON Patch and supporting such is beyond the scope of the specification. The focus of the specification was simply encoding objects with references to themselves.

JSON Reference is not JSON Schema draft v06+'s JSON Reference like de-referencing implementation, although it has been suggested as a practical replacement for it.

## JSON SCHEMA STYLE JSON REFERENCE CONFORMANCE
Like JSON Schema draft v06+ specifications, this implementation breaks with the original JSON  specification in using `$id` instead of `id` for identifiers. However, this implementation diverges with the JSON Schema style JSON Reference in some places.

  - `$id` does not establish the base URI of the document. Instead a base URI is provided by the client, either explicitly, or as the URI from which the resource was loaded.
  - The `$id` keyword does not change the base URI in anyway. By default, each distinct document has a base URI (as above), and any relative URI encountered is qualified against this *singular*, document wide base URI.
  - This implementation places no restriction on where a `$ref` can occur or what a `$ref` can refer to.

# NOTES FOR JSON REFERENCE v0.4.0 IMPLEMENTATIONS

## ON ID POINTERS
According to [JSON Reference](https://tools.ietf.org/html/draft-pbryan-zyp-json-ref-03), the JSON reference fragment part must be [JSON Pointers](https://tools.ietf.org/html/draft-ietf-appsawg-json-pointer-04). However, there is another commonly implemented type of reference: a reference to an object that has an `$id` field. JSON Schema, requires such pointers. JSON v0.4.0 extends v0.3.0 by adding explicit support for them. Example:

    {
      "foo": "bah",
      "a": {
        "$id": "#foo",
      },
      "b": {
        "byid": { "$ref": "#foo" }
        "byref": { "$ref": "#/foo" }
      },
    }

Gives:

    ...
    "b": {
      "byid": { "$id: "#foo" }
      "byref": "bah"
    }


## ON PARSING CIRCULAR JSON REFERENCES
The main issues with JSON References containing JSON Pointers are certain types of circular references (see https://github.com/json-schema/json-schema/wiki/$ref-traps):

### ON POINTERS TO POINTERS
In general circular references are allowed and useful for defining graph like structures. However, consider:

    {
      "foo": { "$ref": "#/bah" },
      "bah": { "$ref": "#/foo" }
    }

This is make no sense because it is *vacuous*: the references are expected to refer to valid JSON, they are not JSON content in an of themselves, so in this case no valid JSON can ever be resolved, and it is illegal. On the other hand, all of the following are legal:

    {
      "foo": { "$ref": "#/bah" },
      "bah": { "$ref": "#/" }
    }

    {
      "foo": { "$ref": "#/" }
    }

    {
      "definitions": {
        "foo": {"properties": {"bar": {"$ref": "#/definitions/bar"}}},
        "bar": {"properties": {"foo": {"$ref": "#/definitions/foo"}}}
      },
      "type": "object",
      "properties": {"foo": {"$ref": "#/definitions/foo"}}
    }

In summary, pointers to pointers are legal, but pointers to pointers resulting in a *pure* pointer loop are illegal. Conceptually, if one can collapse any pointer to pointer chain into an eventual non pointer value it's legal. If not, that's illegal.

### ON POINTERS THAT POINT *THROUGH* OTHER POINTERS
Consider the following document:

    {
      "a": {
        "x": { "$ref": "#/b/c/x" },
      },
      "b": { "$ref": "#/c" },
      "c": {
        "x": "Hey you found me!"
      }
    }

`#/b/c/x` points *through* the pointer at `#/b`. For this to work, we must ensure `#/b` is resolved before `#/b/c/x`.

## REFS TO NON CONTAINER TYPES
This:

    {
      "a": 1,
      "b": { "$ref": "#/a" }
    }

gives a decoded object like:

    {
      "a": 1,
      "b": 1
    }

Whether `a` and `b` actually reference the same storage location is implementation dependent. Implementations MAY allow the client to configure either behavior.

## ENCODING
This JSON Reference specification provides no guidance on how to encode native values containing references to JSON Reference compatible JSON documents.
