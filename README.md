## Registration Information

* Tag: 105 (defined-record)
* Data Item: array
* Semantics: Identify and define a sequence of property names for a record, and use that record definition
* Reference: https://github.com/kriszyp/cbor-records
* Contact: Kris Zyp <kriszyp@gmail.com>


* Tag: 106 (referenced-record)
* Data Item: array
* Semantics: Use a predefined record definition to provide the property values from an array to construct a record.
* Reference: https://github.com/kriszyp/cbor-records
* Contact: Kris Zyp <kriszyp@gmail.com>

## Rationale

In most languages there is a signficant and meaningful distinction between maps and objects/records. The former is often use to represent data with highly variable keys, whereas a record represents an entity with a predictable structure and set of "fields" that may be used describe many instances of a record, potentially within the same data structure. These tags provide a mechanism for efficiently and explicitly encoding and decoding records. There are several key benefits to this:
* This tag provides an explicit indicator of a record structure. This is somewhat complementary to tag [259 (Map datatype with key-value operations)](https://github.com/shanewholloway/js-cbor-codec/blob/master/docs/CBOR-259-spec--explicit-maps.md), but gives a clear, unambiguous indication of records.
* Defining and referencing/reusing a record structure is a much more space efficient encoding. Many data structures may include record/objects with the same structure, and reserializing their entire set of property names for each instance is very inefficient. While the [stringref](http://cbor.schmorp.de/stringref) helps mitigate this, being able to simply reference the whole record structure is more efficient. The use of record structures can substantially decrease the size of encoded data structures.
* Defining and referencing/reusing a record structure can also be much more performant. Besides the performance benefits of simply encoding fewer bytes, decoders can take advantage of optimizations based on knowing the data structure that will be used before the reading the property values, which can yield significant benefits in many languages/environments.
* This aligns well with [CDDL](https://tools.ietf.org/html/rfc8610) which also has the concept of records, and allows for decoders to quickly pair records with structures declared in a CDDL definition.

This tag definition uses an approach of declaring the id of records when defined, which gives encoders more flexibility in how they allocate,  track, and reuse the ids.

## Description

To encode and define a new record/object structure and an instance use the defined-record tag (105). The tag value should be an array, with N+2 elements, where N is the number of properties in the record/object instance to be encoded. The first array element should be a nested record definition array that is the sequence of property names (each element of the nested array is a property name). The second element in the main array should be the record definition id (used to reference it later). This id becomes associated with the record definition, so it can later be referenced. All subsequent elements in the array should be the property values of the current record being encoded, corresponding to the property names, by position, as defined in the record definition array. A decoder should return this record instance (and remember the record definition).

To reuse the record definition and later create another record/object instance using the same set of property names, use the referenced-record tag (106). This can encode a record/object with the same structure, and referencing the previously defined record definition. The value for this tag should be an array, with N+1 elements, where N is the number of properties in the record. The first element should be the id of the record definition to use. All subsequent elements in the array should be the property values of the current record being encoded, corresponding to the property names, by position, as defined in the record definition array. The decoder should return the record/object instance.

A defined-record can use an id of a previously defined record, in which case the previously defined record is discarded and replaced by the new record definition. Encoders are encouraged to limit the number of ids used and cycle through ids as necessary to limit the total number of ids tracked at a time.

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
83					# array(3)
	d8	69			# tag(105) - defined-record
		84			# array(4)
			82		# array(2) - record definition
				64 "name"	# string("name")
				65 "value"	# string("value")
			00		# unsigned(0) - id of 0
			63 "one"	# string("one")
			01		# unsigned(1)
	d8	6a			# tag(106) - referenced-record
		83			# array(3)
			00		# unsigned(0) - id of 0
			63 "two"	# string("two")
			02		# unsigned(2)
	d8	6a			# tag(106) - referenced-record
		83			# array(3)
			00		# unsigned(0) - id of 0
			65 "three"	# string("three")
			03		# unsigned(3)
```
