# JSON REFERENCE v0.4.0.alpha

## OVERVIEW
This specification defines JSON Reference (or JsonRef) v0.4.0. JSON Reference v0.4.0 succeeds [JSON Reference v0.3.0][json-ref-v03] and is *not* backwards compatible. JSON Reference v0.4.0 requires [JSON Pointer v0.4.0][json-pointer].

JSON Reference is a way to encode references in [JSON][json] documents. Conceptually, JSON Reference dereferencers extend pure JSON decoders by parsing any JSON reference (`$ref`) occurrences to native language dependent reference types.

## SPECIFICATION
Every valid JSON Reference document is a valid JSON document. JSON Reference makes use of four special object properties meaningful to JSON Reference, but not meaningful to JSON: `$ref`, `$id` and optionally `$refProp`, `$idProp`:

### $id

  - The `$id` property is optional on any object. `$id` is used to uniquely identify a sub object within within the scope of a given JSON document.
  - The `$id` property is analogous to the `name` or `id` [HTML attribute][html-name-attr].
  - `$id` property values must *begin with a letter ([A-Za-z]) and may be followed by any number of letters, digits ([0-9]), hyphens ("-"), underscores ("_"), colons (":"), and periods (".")"*, except in the following case:
  - To support existing practices, implementations may allow an `$id` occurring at the root of a JSON document, and only the root, to be an absolute URI.
  - `$id` property values are case sensitive.
  - `$id` property values must be unique within a *JSON document*. Implementations must raise an error if duplicate `$id` values are present.

### $ref

  - Objects with a `$ref` property, such as `{ "$ref": <URI> }` are *entirely replaced* by the value pointed to by `<URI>`. This value is called the *replacement-value*.
  - All other properties of an object containing a `$ref` key *are ignored*.
  - `<URI>` must be a valid [URI][uri]. Implementations must attempt to resolve the `<URI>` to a *replacement-value* as described below.
  - The fragment component of `<URI>` must be: empty, a valid `$id` property value, or a valid [JSON Pointer][json-pointer].
  - If `<URI>` consists *only* of a fragment identifier component (example: "#frag"):
    - If the identifier starts with "/" the entire identifier is a JSON Pointer. The *replacement-value* is the value pointed to by the JSON pointer from th root of the given document.
    - Else the characters before the first "/" refer to the object in the current document with the matching `$id` and the remaining characters (if any) are a JSON Pointer that should be resolved relative to this object.
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
  - Implementations must NOT attempt to load remote resources or any resource outside the current JSON document scope by default. In general, implementations should make it clear how and where external values may be retrieved from and require the client to explicitly enable or configure this behavior.
  - If a loaded resource refers to all or part of a JSON document, that JSON document must be processed as a standalone document as described above, and NOT considered part of, or substituted into, the referring source JSON document.

### $idProp, $refProp

    - *Optionally* allow for a different property names other than the default "$id", and "$ref" to be used.
    - Must be set on the JSON document root.
    - If not set, the default "$id" and "$ref" are used.
    - Example:

      {
        "$idProp": "$id.f4f06763-8133-4c80-a57c-2d11ea479b7d",
        "$refProp": "$ref.c84b3603-ad3a-4358-b42c-671c7efcc6fa",
        "a": {
          "$id.4f06763-8133-4c80-a57c-2d11ea479b7d": "a",
          "foo": "bah",
          "...": "..."
        },
        "b": { "a": { "$ref.c84b3603-ad3a-4358-b42c-671c7efcc6fa": "#a" } }
      }

### references and replacement-values

  - [JSON](https://www.json.org/json-en.html) is built on two "structural" or "container" types:
    - **objects:** *"A collection of name/value pairs. In various languages, this is realized as an object, record, struct, dictionary, hash table, keyed list, or associative array."*
    - **arrays:** *"An ordered list of values. In most languages, this is realized as an array, vector, list, or sequence."*
  - JSON also supports five other non-structural types of value: *string, number, true, false, null*.
  - JSON Reference implementations must replace references to structural types (arrays and objects) with "native" references (this does not *necessarily* mean native to the programming language, but rather, native to the application). This is called *reference-replacement*.
  - JSON Reference implementations may choose to replace references to non-structural types with references or with copies of the replacement-value. This is called *copy-replacement*. Implementations may choose to make this behavior configurable for non-structural types.

### encoding the property "$ref"

  - A property that is intended to decode to "$ref" (and not be treated as a reference by JSON Reference) should be encoded as "$$ref" and JSON Reference dereferencing should replace "$$ref" with "$ref" after dereferencing all "$ref" instances.

### encoding native references as JSON Reference compatible JSON
This JSON Reference specification provides no guidance on how to encode native objects containing references to JSON Reference compatible JSON documents. Of course, there is nothing prohibiting this from being implemented.

### lazy dereferencing
Lazy dereferencing of `$ref` objects in a decoded JSON document may be supported. Lazy dereferencing attempts to dereference `$ref` objects "as needed". How "as needed" is defined is dependent on the application. All the above rules still apply. The *only* difference between lazy and canonical (upfront) dereferencing is *when* errors are detected and raised to the client. Implementations that use or support lazy dereferencing should make it clear when, how, and if lazy dereferencing is being performed.

## WHAT JSON REFERENCE IS NOT
JSON Reference is not JSON Merge or JSON Patch and supporting such is beyond the scope of the specification. The focus of the specification is simply encoding objects with references to themselves.

JSON Reference is not [JSON Schema][json-schema] draft v06+'s JSON Reference like dereferencing implementation, although it has been suggested as a practical replacement for it.

## JSON SCHEMA STYLE JSON REFERENCE CONFORMANCE
Like JSON Schema draft v06+ specifications, this implementation breaks with the original JSON Reference specifications in using `$id` instead of `id` for object identifiers. However, this specification diverges with the canonical JSON Schema specification of references in some places:

  - `$id` does not establish the base URI of the document. Instead, a base URI should be provided by the client, either explicitly, or as the URI from which the resource was loaded.
  - The `$id` keyword does not change the base URI in anyway. By default, each distinct document has a base URI (as above), and any relative URI encountered should be qualified against this *singular*, document wide base URI.
  - This implementation places no restriction on where a `$ref` can occur or what a `$ref` can refer to.

## NOTES FOR JSON REFERENCE v0.4.0 IMPLEMENTORS

### $id pointers
According to [JSON Reference v0.3.0][json-ref-v03], the JSON reference fragment part must be a [JSON Pointer][json-pointer]. Another commonly implemented type of reference is a reference to an object that has an `$id` field as described above. JSON Schema for example, requires such references. The specification extends v0.3.0 by adding explicit support for this reference type (as described above). Example:

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

### pointers to pointers
In general circular references are allowed and useful for defining graph like structures - which is the whole point in JSON Reference. However, consider:

    {
      "foo": { "$ref": "#/bah" },
      "bah": { "$ref": "#/foo" }
    }

    { "$ref": "#/ }

Neither document makes no sense because they are *vacuous*: the references are expected to refer to valid JSON, they are not content in an of themselves, so in these cases no valid JSON can ever be resolved, and it is illegal. On the other hand, all of the following are legal:

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

In summary, pointers to pointers are legal, but pointers to pointers resulting in a *pure* pointer loop are illegal. Conceptually, if one can collapse any pointer to pointer chain into an eventual non pointer value it's legal. If not, that's illegal.

### pointers that point *through* other pointers
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

### refs to non-structural types
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

---

<p align=center>This page is hosted on [Github](https://github.com/sgpinkus/jsonref.org/).</p>

[json]: https://www.json.org/json-en.html
[json-schema]: https://json-schema.org/
[json-ref-v03]: https://tools.ietf.org/html/draft-pbryan-zyp-json-ref-03
[json-pointer]: https://tools.ietf.org/html/draft-ietf-appsawg-json-pointer-04
[ref-traps]: https://github.com/json-schema/json-schema/wiki/$ref-traps
[html-name-attr]: https://www.w3.org/TR/html401/types.html#type-id
[uri]: https://tools.ietf.org/html/rfc3986
