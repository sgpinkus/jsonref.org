# JSON REFERENCE v0.4.0.alpha

<!-- MDTOC maxdepth:6 firsth1:2 numbering:0 flatten:0 bullets:1 updateOnSave:1 -->

- [OVERVIEW](#overview)
- [SPECIFICATION](#specification)
   - [$id](#id)
   - [$ref](#ref)
   - [$idProp, $refProp](#idprop-refprop)
   - [References and replacement-values](#references-and-replacement-values)
   - [Lazy dereferencing](#lazy-dereferencing)
   - [Normalizing references to JSON encodable objects](#normalizing-references-to-json-encodable-objects)
- [WHAT JSON REFERENCE IS NOT](#what-json-reference-is-not)
- [JSON SCHEMA STYLE JSON REFERENCE CONFORMANCE](#json-schema-style-json-reference-conformance)
- [NOTES FOR JSON REFERENCE v0.4.0 IMPLEMENTORS](#notes-for-json-reference-v040-implementors)
   - [$id pointers](#id-pointers)
   - [Pointers to pointers](#pointers-to-pointers)
   - [Pointers that point *through* other pointers](#pointers-that-point-through-other-pointers)
   - [Refs to non-structural types](#refs-to-non-structural-types)

<!-- /MDTOC -->

## OVERVIEW
[JSON][json] alone can only capture tree structured objects. JSON Reference (or JsonRef) is a standard way to represent references in JSON documents, to allow graph structured objects to be stored as JSON. Conceptually, JSON Reference extends pure JSON decoding/encoding by parsing any JSON reference (`$ref`) occurrences to native language dependent reference types after decoding, and normalizing references in a JSON reference compatible way before encoding (some implementations may only support dereferencing):

                                               object
                                                ___
                            .-------------.    |   |\     .---------.
                   .------->| Normalize   |--->|   '-|--->| Encode  |-------.
                   |        |             |    |     |    |         |       |
                   |        '-------------'    |_____|    '---------'       |
                   |                                                        v
                object                         object                    string
                 ___                            ___                       ___
        (\_/)   |   |\      .-------------.    |   |\     .---------.    |   |\
        (O.o)   |   '-|     | Dereference |    |   '-|    | Decode  |    |   '-|
        (> <)   |     |<----|             |<---|     |<---|         |<---|     |
                |_____|     '-------------'    |_____|    '---------'    |_____|


This specification defines JSON Reference v0.4.0.alpha. JSON Reference v0.4.0 succeeds [JSON Reference v0.3.0][json-ref-v03] and is not entirely backwards compatible. JSON Reference v0.4.0 relies on the [JSON Pointer v0.4.0][json-pointer] specification.

This specification is hosted on [Github](https://github.com/sgpinkus/jsonref.org/).

## SPECIFICATION
Every valid JSON Reference document is a valid JSON document. JSON Reference makes use of four special object properties meaningful to JSON Reference, but not meaningful to JSON: `$ref`, `$id` and optionally `$refProp`, `$idProp`:

### $id

  - The `$id` property is optional on any object. `$id` is used to uniquely identify a sub object within the scope of a given JSON document.
  - The `$id` property is analogous to the `name` or `id` [HTML attribute][html-name-attr].
  - `$id` property values must *begin with a letter ([A-Za-z]) and may be followed by any number of letters, digits ([0-9]), hyphens ("-"), underscores ("_"), colons (":"), and periods (".")"*, except in the following case:
  - To support existing practices, implementations may allow an `$id` occurring at the root of a JSON document, *and only the root*, to be an absolute URI.
  - `$id` property values are case sensitive.
  - `$id` property values must be unique within a *JSON document*. Implementations must raise an error if duplicate `$id` values are present within a single JSON Document.

### $ref

  - Objects with a `$ref` property, such as `{ "$ref": <URI> }`, are *entirely replaced* by the value pointed to by `<URI>`. This value is called the *replacement-value*.
  - All other properties of an object containing a `$ref` key *are ignored*.
  - If a replacement-value cannot be resolved an error must be raised.
  - `<URI>` must be a valid [URI][uri]. Implementations must attempt to resolve the `<URI>` to a *replacement-value* as follows:
  - The fragment identifier component of `<URI>` must be:
    - empty (example: "#", or just ""),
    - a valid `$id` property value immediately followed by an optional [JSON Pointer][json-pointer] (example: "#a/b/c"),
    - or a JSON Pointer (example: "#/a/b/c").
  - If `<URI>` consists *only* of a fragment identifier component (example: "#frag"):
    - If the identifier is empty (case 1 above), the replacement value is the current document.
    - If the identifier is a JSON Pointer (case 3 above), the *replacement-value* is the value pointed to by the JSON pointer from the root of the current document.
    - If the identifier is a valid `$id` property value immediately followed by an optional JSON Pointer (case 2 above), the JSON Pointer must be resolved relative to the object in the current document with the given `$id` property.
    - Example:

      {
        "a": { "$id": "x", "b": 1 },
        "b": 2,
        "c": { "$ref": "#x/b" },
        "d": { "$ref": "#/b" }
      }

      should give the following after dereferencing:

      {
        "a": { "$id": "a", "b": 1 },
        "b": 2,
        "c": 1
        "d": 2
      }

  - If `<URI>` is not just a fragment identifier component (example: "/some/path#frag"), implementations should first resolve relative URIs against a base URI as described in [RFC 3986][uri], then use the value identified by the absolute URI as the *replacement-value*. How the base URI is determined is beyond the scope of this specification.
  - If a loaded resource refers to all or part of a JSON document, that JSON document must be dereferenced as a standalone document as described above, and resolved to a replacement-value as described above.
  - Implementations *must not* attempt to load remote resources or any resource outside the current JSON document scope by default. In general, implementations should make it clear how and where external values may be retrieved from and require the client to explicitly enable and/or configure this behavior.

### $idProp, $refProp

  - The `$idProp`, and `$refProp` properties *optionally* allow for different property names to be used for "$id", and "$ref" .
  - If set, `$idProp` and `$refProp` must be set on the JSON document root. The property values must be valid JSON property names.
  - If not set, the default property names "$id" and "$ref" are used.
  - Example:

    {
      "$idProp": "$id.607cc38b5ff40",
      "$refProp": "$ref.607cc3a1c764b",
      "a": {
        "$id.607cc38b5ff40": "a",
        "foo": "bah",
        "...": "..."
      },
      "b": { "a": { "$ref.607cc3a1c764b": "#a" } }
    }

### References and replacement-values

  - [JSON](https://www.json.org/json-en.html) is built on two "structural" or "container" types:
    - **objects:** *"A collection of name/value pairs. In various languages, this is realized as an object, record, struct, dictionary, hash table, keyed list, or associative array."*
    - **arrays:** *"An ordered list of values. In most languages, this is realized as an array, vector, list, or sequence."*
  - JSON also supports five other non-structural types of value: *string, number, true, false, null*.
  - JSON Reference implementations must replace references to structural types (arrays and objects) with "native" references (this does not *necessarily* mean native to the programming language, but rather, native to the application). This is called *reference-replacement*.
  - JSON Reference implementations may choose to replace references to non-structural types with references or with copies of the replacement-value. This is called *copy-replacement*. Implementations may choose to make this behavior configurable for non-structural types.

### Lazy dereferencing
Lazy dereferencing of `$ref` objects in a decoded JSON document may be supported. Lazy dereferencing attempts to dereference `$ref` objects "as needed". How "as needed" is defined is dependent on the application. All the above rules still apply. The *only* difference between lazy and canonical (upfront) dereferencing is *when* errors are detected and raised to the client. Implementations that use or support lazy dereferencing should make it clear when, how, and if lazy dereferencing is being performed.

### Normalizing references to JSON encodable objects
This JSON Reference specification provides no strict guidance on how to encode native objects containing references to JSON Reference compatible JSON documents. Of course, this can still be done. The only requirement is that the resulting JSON is actually encodable (tree structured) and uses Json Reference keywords in a manner consistent with this specification.

## WHAT JSON REFERENCE IS NOT
JSON Reference is not JSON Merge or JSON Patch and supporting object merging is beyond the scope of the specification. The focus of the specification is simply encoding objects with references to themselves.

JSON Reference is not [JSON Schema][json-schema] draft v06+'s JSON Reference like dereferencing implementation, although it has been suggested as a replacement for it.

## JSON SCHEMA STYLE JSON REFERENCE CONFORMANCE
Like JSON Schema draft v06+ specifications, this implementation breaks with the original JSON Reference specifications in using `$id` (by default) instead of `id` for object identifiers. However, this specification diverges with the canonical JSON Schema specification of references:

  - `$id` does not establish the base URI of the document. Instead, a base URI should be provided by the client, either explicitly, or as the URI from which the resource was loaded.
  - The `$id` keyword does not change the base URI in anyway. By default, each distinct document has a base URI established for it in some way beyond the scope of this specification, and any relative URI encountered should be qualified against this document wide base URI (resolving to a valid absolute URI, does *not* imply the identified resource should then be retrieved).
  - Unlike in JSON Schema references, this implementation places no restriction on where a `$ref` can occur or what a `$ref` can refer to.

## NOTES FOR JSON REFERENCE v0.4.0 IMPLEMENTORS

### $id pointers
According to [JSON Reference v0.3.0][json-ref-v03], the JSON reference fragment part must be a [JSON Pointer][json-pointer]. Another commonly implemented type of reference is a reference to an object that has an `$id` field as described above. JSON Schema for example, requires such references. The specification extends the previous specification by adding explicit support for this reference type (described above). Example:

    {
      "foo": "bah",
      "a": {
        "$id": "#foo"
      },
      "b": {
        "byid": { "$ref": "#foo" },
        "byref": { "$ref": "#/foo" }
      }
    }

Gives:

    ...
    "b": {
      "byid": { "$id: "#foo" },
      "byref": "bah"
    }

### Pointers to pointers
In general circular references are allowed and useful for defining graph like structures - which is the whole point in JSON Reference. However, consider:

    {
      "foo": { "$ref": "#/bah" },
      "bah": { "$ref": "#/foo" }
    }

    { "$ref": "#/ }

Neither document makes sense because they are *vacuous*: the references are expected to refer to valid content, they are not content in an of themselves, so in these cases no valid value can ever be resolved, and the document is erroneous (raise error). On the other hand, all of the following are fine:

    {
      "foo": { "$ref": "#/bah" },
      "bah": { "$ref": "#/" }
    }

    {
      "foo": { "$ref": "#/" }
    }

    {
      "definitions": {
        "foo": { "properties": { "bar": { "$ref": "#/definitions/bar" } } },
        "bar": { "properties": { "foo": { "$ref": "#/definitions/foo" } } }
      },
      "type": "object",
      "properties": { "foo": { "$ref": "#/definitions/foo" } }
    }

In summary, pointers to pointers are fine, but pointers to pointers resulting in a *pure* pointer loop are erroneous. Conceptually, if one can collapse any pointer to pointer chain into an eventual non pointer replacement-value dereferencing should succeed, if not an error must be raised.

### Pointers that point *through* other pointers
Consider the following document:

    {
      "a": {
        "x": { "$ref": "#/b/x" }
      },
      "b": { "$ref": "#/c" },
      "c": {
        "x": "Hey you found me!"
      }
    }

`#/b/x` points *through* the reference at `#/b`. For this to work, we must ensure `#/b` is resolved before `#/b/x`.

### Refs to non-structural types
This:

    {
      "a": 1,
      "b": { "$ref": "#/a" }
    }

should give a dereferenced object like:

    {
      "a": 1,
      "b": 1
    }

Whether `a` and `b` actually reference the same storage location or a copy is implementation dependent (as explained above). Implementations may choose to allow the client to configure either behavior.

[json]: https://www.json.org/json-en.html
[json-schema]: https://json-schema.org/
[json-ref-v03]: https://tools.ietf.org/html/draft-pbryan-zyp-json-ref-03
[json-pointer]: https://tools.ietf.org/html/draft-ietf-appsawg-json-pointer-04
[ref-traps]: https://github.com/json-schema/json-schema/wiki/$ref-traps
[html-name-attr]: https://www.w3.org/TR/html401/types.html#type-id
[uri]: https://tools.ietf.org/html/rfc3986
