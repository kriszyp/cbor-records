## Registration Information

### Tag: 29284 (record-definitions)
* Data Item: array
* Semantics: Identify and define a set of record structures (each a sequences of property names) that can be referenced as tags in the included value (and the scope for the record tag definitions)
* Reference: https://github.com/kriszyp/cbor-records
* Contact: Kris Zyp <kriszyp@gmail.com>

### Tag: 29285 (inline-record)
* Data Item: array
* Semantics: Identify and define a record structure (a sequence of property names), and use that record structure definition to interpret the included values.
* Reference: https://github.com/kriszyp/cbor-records
* Contact: Kris Zyp <kriszyp@gmail.com>

#### User dynamic tags (record-reference)
This proposal includes explicit declaration of tags that an encoder specifies, but these are specified by the encoder and do not require any a priori knowledge by decoders, nor global registration or coordination.

## Rationale

In most languages, there is a significant and meaningful distinction between dynamic maps or dictionaries and objects or records. The former is often used to represent data with highly variable keys, whereas a record represents an entity with a specific predictable structure or class and set of "fields" that may be used to describe many instances or records of that class or structure, potentially within the same data structure. These tags provide a mechanism for efficiently and explicitly encoding and decoding records. There are several key benefits to this:
* This tag provides an explicit indicator of a record structure. This is somewhat complementary to tag [259 (Map datatype with key-value operations)](https://github.com/shanewholloway/js-cbor-codec/blob/master/docs/CBOR-259-spec--explicit-maps.md) which gives a specific indication of an arbitrary keyed map, whereas this tag gives a clear, unambiguous indication of records.
* Defining and referencing/reusing a record structure is a much more space efficient encoding. Many data structures may include record/objects with the same structure, and re-serializing their entire set of property names for each instance is very inefficient. While the [stringref](http://cbor.schmorp.de/stringref) helps mitigate this, being able to simply reference the whole record structure is more efficient. The use of record structures can substantially decrease the size of encoded data structures.
* Defining and referencing/reusing a record structure can also be much more performant. Besides the performance benefits of simply encoding fewer bytes, decoders can take advantage of optimizations based on knowing the data structure that will be used before reading the property values, which can yield significant benefits for constructing objects in many languages/environments.
* This is designed to facilitate "streaming" encoders that may not know of all data structures prior to beginning encoding, or, for performance reasons, do not wish to expend unnecessary processing to "discover" all structures ahead of time, and can flexibly support up-front declaration of structures or inline just-in-time declarations.
* This provides more flexibility for arrays of various data structures and nested reuse of data structures than is possible with column-based homogenous tables like those of [RFC-8746](https://tools.ietf.org/html/rfc8746).
* This aligns well with [CDDL](https://tools.ietf.org/html/rfc8610) which also has the concept of records, and allows for decoders to quickly pair records with structures declared in a CDDL definition.

This tag definition uses an approach of declaring the id of records when defined, which gives encoders more flexibility in how they allocate, track, and reuse the ids.

## Description

To encode and define a set of record/object structures for a data structure, use the record-definitions tag (29284). To encode and define a single record/object structure and an instance, use the inline-record tag (29285).

### record-definitions
The tag value should be an array. The length of the array should be N+2 where N is the number of record structure definitions that are defined. The last element in the array should always be interpreted as the primary data item (the main returned data item) from the tag, and this data item may include references to the defined record structures. The record structures are defined before this primary data item.

The first element in the array should be an integer specifying the tag id to use for the subsequent record structure. The second element should be the record structure, which should be array of property names. If there are more than three elements in the array, any element before the last element (holding the primary data item) should be an array that also specifies record structures, and each record structure is assigned a tag id that is one greater than the previous record structure.

The tag ids to be assigned to records for referencing can be determined by the encoder, and may be assigned to avoid tag ids in use for other purposes. It is may be easier for an encoder to use unassigned first-come first-serve tag ids (the 32768 - 49999 range).


### inline-record
The tag value should be an array, with N+2 elements, where N is the number of properties in the record/object instance to be encoded. The first array element should be the should be the record definition tag id (used to reference it later, from a unambiguously subsequent position in the document). This tag id becomes associated with the record definition, so it can later be referenced. The second array element should be the record structure definition array that is the sequence of property names (each element of the nested array is a property name). All subsequent elements in the array should be the property values of the current record being encoded, corresponding to the property names, by position, as defined in the record definition array. A decoder that is decoding record structure tags should return this record instance (and store the record definition).

### record-references
To reuse the record definition and create another record/object instance using the same set of property names, we can reference the original record definition. This must be done from a subsequent element in a CBOR array or a child/property-value of the record. To reference a previously defined record definition, we use a tag with the tag id that corresponds to the id specified in the defined record. This can encode a record/object with the same structure, and referencing the previously defined record definition. The value for this tag should be an array, with N elements, where N is the number of properties in the record. Each element in the array should be the property values of the current record being encoded, corresponding to the property names, by position, as defined in the record definition array. The decoder should return the record/object instance.

An inline record definition must be in an unambiguously “earlier” position in the document than the record reference that references it. An earlier position is defined as a lower position in an array element. A subsequent position is defined as a higher position in an array element. This position is also transitively applied to child values (they are “within” the position of their parent). Also, a parent record definition is defined as an earlier position than its child values (A child/property value may reference a record definition of its parent record). If a inline-record tag is used within a record-definitions tag/value, the scope of the defined record/tag is limited to inside that record-definitions tag.

Inline record definitions can use a tag id of a previously defined record (or an id of a tag used for different purposes), in which case the previously defined record/tag definition is discarded and replaced by the new record definition for that tag id, such that the record definition will be used by subsequent record references. To limit memory requirements of decoders, no more than 16,384 records structures should defined at once in a CBOR item, encoders need to limit the number of ids used and cycle through ids as necessary to stay with the specified range of tag ids. Encoders SHOULD NOT rely on the order of properties in any CBOR map for referencing a previously defined record, as relying on this order is NOT RECOMMENDED (per RFC-8949). However, since a record structure is represented with an array structure in a CBOR generic data model, referencing record structures from previous record values is deterministically indicated by array order, as defined by CBOR and specified above.

If the array of property values in a referenced record has fewer elements than the array property names, only the subset of property names that positionally align should be used.

Decoders SHOULD be able to handle cases where a record-definition has a record with a property value that uses a record-reference to reference the containing record definition (to create structures that have self-referencing properties).

## Examples

Let's consider how we could encode the following JSON using CBOR with the record tags:
```
[
   { "name": "one", "value": 1 },
   { "name": "two", "value": 2 },
   { "name": "three", "value": 3 }
]
```
We can encode this with an array, and use record-definitions around the array:
```
D9 72 64             -- tag(29284) - record-definitions
   83                -- array(3)
      19             -- unsigned 16-bit uint
         80 00       -- assign tag id of 32768
      82             -- array(2) - record structure definition
         64 "name"   -- string("name")
         65 "value"  -- string("value")
      83             -- array(3)
         D9 80 00    -- tag(32768) - record-reference
            83       -- array(2)
               63 "one" -- string("one")
               01    -- unsigned(1)
         D9 80 00    -- tag(32768) - record-reference
            83       -- array(2)
               63 "two" -- string("two")
               02    -- unsigned(2)
         D9 80 00    -- tag(32768) - record-reference
            82       -- array(2)
               65 "three" -- string("three")
               03    -- unsigned(3)
```
The generic data model representation would be:
```
[
   tag(29284): array(4):[
      32768,
      array(2):["name", "value"],
      array(3):[
         tag(32768): array(2):[
            "one",
            1
         ],
         tag(32768): array(2):[
            "two",
            2
         ],
         tag(32768): array(2):[
            "three",
            3
         ]
      ]
   ]
]
```
Alternately, we can encode this with an array, and use an inline-record for the first element in the array:
```
83                -- array(3)
   D9 72 64       -- tag(29284) - record-definition
      84          -- array(4)
         82       -- array(2) - record definition
            64 "name" -- string("name")
            65 "value" -- string("value")
         19       -- unsigned 16-bit uint
            80 00 -- tag id of 32768
                  -- record values:
         63 "one" -- string("one")
         01       -- unsigned(1)
   D9 80 00       -- tag(32768) - record-reference
      83          -- array(2)
         63 "two" -- string("two")
         02       -- unsigned(2)
   D9 80 00       -- tag(32768) - record-reference
      82          -- array(2)
         65 "three" -- string("three")
         03       -- unsigned(3)
```
The generic data model representation would be:
```
[
   tag(29284): array(4):[
      array(2):["name", "value"],
      32768,
      "one",
      1
   ],
   tag(32768): array(2):[
      "two",
      2
   ],
   tag(32768): array(2):[
      "three",
      3
   ]
]
```

