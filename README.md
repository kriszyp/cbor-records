## Registration Information

### Tag: 105 (defined-record)
* Data Item: array
* Semantics: Identify and define a sequence of property names for a record, and use that record definition
* Reference: https://github.com/kriszyp/cbor-records
* Contact: Kris Zyp <kriszyp@gmail.com>

### Tags: 26880-27135 (referenced-record)
* Data Item: array
* Semantics: Use a predefined record definition to provide the property values from an array to construct a record.
* Reference: https://github.com/kriszyp/cbor-records
* Contact: Kris Zyp <kriszyp@gmail.com>

## Rationale

In most languages, there is a significant and meaningful distinction between maps and objects/records. The former is often used to represent data with highly variable keys, whereas a record represents an entity with a predictable structure and set of "fields" that may be used to describe many instances of a record, potentially within the same data structure. These tags provide a mechanism for efficiently and explicitly encoding and decoding records. There are several key benefits to this:
* This tag provides an explicit indicator of a record structure. This is somewhat complementary to tag [259 (Map datatype with key-value operations)](https://github.com/shanewholloway/js-cbor-codec/blob/master/docs/CBOR-259-spec--explicit-maps.md) which gives a specific indication of an arbitrary keyed map, whereas this tag gives a clear, unambiguous indication of records.
* Defining and referencing/reusing a record structure is a much more space efficient encoding. Many data structures may include record/objects with the same structure, and re-serializing their entire set of property names for each instance is very inefficient. While the [stringref](http://cbor.schmorp.de/stringref) helps mitigate this, being able to simply reference the whole record structure is more efficient. The use of record structures can substantially decrease the size of encoded data structures.
* Defining and referencing/reusing a record structure can also be much more performant. Besides the performance benefits of simply encoding fewer bytes, decoders can take advantage of optimizations based on knowing the data structure that will be used before reading the property values, which can yield significant benefits for constructing objects in many languages/environments.
* This provides more flexibility for arrays of various data structures and nested reuse of data structures than is possible with column-based homogenous tables like those of [RFC-8746](https://tools.ietf.org/html/rfc8746).
* This aligns well with [CDDL](https://tools.ietf.org/html/rfc8610) which also has the concept of records, and allows for decoders to quickly pair records with structures declared in a CDDL definition.

This tag definition uses an approach of declaring the id of records when defined, which gives encoders more flexibility in how they allocate, track, and reuse the ids.

## Description

To encode and define a new record/object structure and an instance, use the defined-record tag (105). The tag value should be an array, with N+2 elements, where N is the number of properties in the record/object instance to be encoded. The first array element should be a nested record definition array that is the sequence of property names (each element of the nested array is a property name). The second element in the main array should be the record definition tag id (used to reference it later, from a unambiguously subsequent position in the document). This tag id becomes associated with the record definition, so it can later be referenced. All subsequent elements in the array should be the property values of the current record being encoded, corresponding to the property names, by position, as defined in the record definition array. A decoder should return this record instance (and store the record definition). The tag id should be a number from 26880-27135, the range reserved from reference-record tags.

To reuse the record definition and create another record/object instance using the same set of property names, we can reference the original record definition. This must be done from a subsequent element in a CBOR array or a child/property-value of the record. To reference a previously defined record definition, we use the referenced-record tag, with the tag id that corresponds to the id specified in the defined record. This can encode a record/object with the same structure, and referencing the previously defined record definition. The value for this tag should be an array, with N elements, where N is the number of properties in the record. Each element in the array should be the property values of the current record being encoded, corresponding to the property names, by position, as defined in the record definition array. The decoder should return the record/object instance.

A record definition must be in an unambiguously “earlier” position in the document than the record reference that references it. An earlier position is defined as a lower position in an array element. A subsequent position is defined as a higher position in an array element. This position is also transitively applied to child values (they are “within” the position of their parent). Also, a parent record definition is defined as an earlier position than its child values (A child/property value may reference a record definition of its parent record).

A defined-record can use a tag id of a previously defined record, in which case the previously defined record is discarded and replaced by the new record definition for that tag id, such that the record definition will be used by subsequent record references. Due to the constrained range of tag ids, encoders need to limit the number of ids used and cycle through ids as necessary to stay with the specified range of tag ids. Encoders SHOULD NOT rely on the order of properties in any CBOR map for referencing a previously defined record, as relying on this order is NOT RECOMMENDED (per RFC-8949). However, since a record structure is represented with an array structure in a CBOR generic data model, referencing record structures from previous record values is deterministically indicated by array order, as defined by CBOR and specified above.

If the array of property values in a referenced record has fewer elements than the array property names, only the subset of property names that positionally align should be used.

Decoders SHOULD be able to handle cases where a defined-record has a record with a property value that uses a referenced-record to reference the containing record definition.

## Examples

Let's consider how we could encode the following JSON using CBOR with the record tags:
```
[
   { "name": "one", "value": 1 },
   { "name": "two", "value": 2 },
   { "name": "three", "value": 3 }
]
```
We can encode this with an array, and using a defined-record for the first element in the array:
```
83                  # array(3)
   d8 69          # tag(105) - defined-record
      84            # array(4)
         82         # array(2) - record definition
            64 "name" # string("name")
            65 "value" # string("value")
         19         # unsigned 16-bit uint
            69 00 # tag id of 26880
                   # record values:
         63 "one" # string("one")
         01      # unsigned(1)
   d9 69 00  # tag(26880) - referenced-record
      83         # array(2)
         63 "two" # string("two")
         02      # unsigned(2)
   d9 69 00  # tag(26880) - referenced-record
      82         # array(2)
         65 "three" # string("three")
         03      # unsigned(3)
```

### Considerations and Alternatives

* There are certainly alternate ways of structuring the arrays that could be used, but this attempts to balance clarity and efficiency.
* This could be specified with just two tags, instead of a range, but this would result in a more complicated procedure referencing record as opposed to this approach.
* The range selection is intentionally based on the aesthetics of reserving tags 0xd8 0x69 and 0xd9 0x69 [...].

