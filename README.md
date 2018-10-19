# dpack

dpack is a very compact binary format for serializing data structures, designed for efficient, high-performance serialization/parsing, and optimized for web use. For common large data structures in applications, a dpack file is often much smaller than JSON (and much smaller than MsgPack as well), and can be parsed faster than JSON and other formats. dpack leverages structural reuse to reduce size, and uses binary format for performance. dpack has several key features:
* Uses internal referencing and reuse of structures, properties, values, and objects for remarkably compact serialization and fast parsing.
* Defined as a valid unicode character string, which allows for single-pass text decoding for faster and simpler decoding (particulary in browser), support across older browsers, and ease of manipulation as a character string. It  can also be encoded in UTF-8, UTF-16, or any ASCII compatible encoding.
* Supports a wide range of types including strings, decimal-based numbers, booleans, objects, arrays, dates, maps, sets, and user-provided classes/types.
* Supports positionally mapped object properties for lazy evaluation of paths for faster access to data without parsing entire data structures (useful for storing, querying, and indexing data in databases).
* Supports referencing of objects which can be used to reorder serialization and reuse objects.
* Optimized to compress well with Huffman/Gzip encoding schemes

This repository is for the specification of dpack. For dpack libraries:
* [JavaScript implementation](https://github.com/DoctorEvidence/dpackjs)

# Specification
DPack is designed for ease of creating high performance implementations, making it easy to use fast bitwise operators, and memory-limited structures. While the dpack messages may appear somewhat complicated description, it is intended to be easily implemented, particularly with typed languages, to be flexible, and to provide protection against excessive memory consumption.

DPack can be described as four layers of well-defined parsing and transformation, to simplify the definition and implementation of DPack. The layers of reading a DPack file are:
* Character decoding from binary data to character-based text.
* Lexing of characters into tokens that define token `type` and `accompanying` numbers.
* Parsing of tokens into rudimentary values and structures.
* Deserialization of rudimentary values/structures into final data structures.

DPack is a binary data format, in the sense that structures are defined through byte-level tokens for machine parsing. But it is primarily specified as character-based format; blocks of data can be entirely decoded using a character set decoding as the first layer of reading, and then the parsing rules operate on the decoded characters. dpack is optimized for, should default to, UTF-8 encoding, but could be encoded in any character set that can encode unicode charaters 0-127. In addition, it can be encoded using additional characters for more efficient space usage when encoded in UTF-16. A DPack message should consist entirely of tokens and strings that are described by the tokens.

## Token Lexing
The basic entity in a dpack message is a token. A token may consist of 1 to 8 characters. The meaning of the characters in a token are determine by their unicode code point number, and further defined by bitwise positions. All token values are based on their character's unicode code point. The token character or characters are used to determine a `type` number from 0 to 3, and an `accompanying` number which is an unsigned integer up to 2^46. A dpack serialized data structure consists entirely of tokens and strings that are read by length specified by tokens, based on the parsing rules.

By default, all token character bytes have an initial 0 bit (if 0 - 127 byte range is used). The second bit is always a "stop" bit. A one means this is the last byte, a zero means additional bytes are part of the token. In the first byte, The next two bits (3rd and 4th) represent the `type` of the token. The remaining bits, the first four bits of the first byte, and all remaining bytes (up to and including a byte with a stop bit) are used to serialize the accompanying number, which is interpreted by big endian bytes/bits, where the first bits are most significant, and later bytes/bits are less significant.
For example:

Character "R": Code point 82. Binary representation:

`0 1 0 1 0 0 1 0` - Stop bit is set (1), type (0 1) is 1, and accompanying number (0 0 1 0) is 2.

Character "!D": Code points 33 and 68. Binary representation:

`0 0 1 0 0 0 0 1   0 1 0 0 0 1 0 0` - Stop bit is not set (0), type (1 0) is 2, bits (0 0 0 1) will be combined with next byte. Next byte, stop bit is set. Combined bits (0 0 0 1  0 0 0 1 0 0) make 68 the accompanying number.

There may be up to 8 bytes, which accomodates up to 46 bits for the accompanying number, therefore the accompanying number must be an unsigned integers under 2^46.

Some types will use the accompanying number to specify the length of a string immediately following the token. When a string is to be read, the number specifies the number of characters to be including in the string, after which the next block can be immediately read. The length of the string is not bytes, but basic multiplane characters. Any supplemental plane should be counted as two characters (surrogates) should be counted as two characters (a pair). In other words, a string length is defined by its UTF-16 encoding (though it may be serialized in UTF-8 in dpack).

dpack has four different reading modes, based on what type of data is being read. Parsing should always start in the open mode, the default reading mode. Each of the reading modes is described below, and the `type` from each token is combined with the accompanying to determine what value is being parsed next based on the reading mode.

### Alternate Encodings
Tokens may also consist of higher character codes and a compliant parser should also be to parse characters that extend beyond unicode 127. For characters with code points 128 and above, the character code point should be interpreted as a 16-bit unsigned integer, with the first bit always as 0, the second bit as a stop bit, the third and fourth bit as type bits, and the remaining 12 bits for the accompanying number. These 16-bit character encoding/decodings can be used for greater efficiency where UTF-16 encoding is preferred (which can be faster in languages that internally represent strings with UTF-16, and there is relatively unlimited socket bandwidth, such as interprocess pipes).

## Parsing Types
The `type` value indicates how the token (and any accompanying string) should be parsed into a rudimentary value. The `type` is two bits so there are four basic parse types:

### 0 - Property Definition
This type defines a property that will specify how the preceeding value should be deserialized. An accompanying number of less than 6 is a property creation, and the parser should read two values after this token. The first is the parameter or key for the property, and the second is the value (interpreted by the property type). An accompanying value of 6 or greater is a property reference, and the parser should just read the next value.

### 1 - Sequence (and Block)
A sequence is used to create objects and arrays that consist of multiple values. For an accompanying number of 10 or less, the accompanying number indicates the number of values should be read after this token, for this sequence. Alternately, an accompanying number of 11 and 13 indicate the start and end of a sequence, respectively. A number of 11 (UTF-8 character "[") means the parser should consequently read values into this sequence until it encounters a value that is an end token, a sequence token with an accompanying number of 13 (UTF-8 characater "]").

An accompanying number of 16 or higher indicates a "block". The parser should read the next (one) value as the value of the block. The accompanying number minus 16 should indicate the number of bytes in the block, and can be used by parsers to defer parsing the contents of the block. A block should also restore the state of all properties after it is read.

### 2 - String
The accompanying number indicates the number of following characters that compose a string. The string, following this token, with the length defined by the accompanying number, is the rudimentary value that is produced.

### 3 - Simple Scalar
This a terminal type that involves no further parsing and directly results in a scalar value being produced. An accompanying number of:
0 - Produces a value of `null`
1 - Produces a value of `undefined` (this is used as a directive in the deserializing of objects)
2 - Produces a value of `true`
3 - Produces a value of `false`
4 and greater - Produces an integer of the accompanying number minus 4.

This defines the parsing of dpack into rudimentary sequences, strings, and scalars and property definitions that describes the deserialization of these values. Next the values are converted to their final deserialized form.

## Deserialization
Deserialization is the conversion of the rudimentary value, structures, and property definitions into final usable data structures.

### Property Definitions
Properties are the central entity that describes how rudimentary values are deserialized into final data structures. Again, property definition has a `type` of 0, and the accompanying number defines the property that will be created, which will then be used to define how the values will be converted. For property creation, the succeeding value always defines the property "key" (except for metadata, in which case the succeeding value is a general parameter). For an accompanying number of less than 5, the following properties can be created based on the accompanying number:
0 - Metadata: The succeeded value is read as the metadata parameter, which can be used for indicating user defined types to convert values to.
1 - Array: The succeeded value (like the rest of the creation properties) defines the property key, and the next value should be deserialized as an array. If the next value is a sequence this sequence of values corresponds to an array of values.
2 - Referencing Type: This property type deserializes values just like the Default property type, *except* that every non-number is consecutively stored in an array on the property, and any number value is a reference to a previous value by index (starting at 0). For example of a number of 3 (type 3, accompanying number 7, UTF-8 character "w"), would reference the index of 3, or the 4th value, that had been read using this property.
3 - Number: This property type deserializes values just like the Default property type, *except* that string values are converted to numbers. The conversion of strings to numbers should follow the same rules as JSON.
4 - Default: For the Default property type, scalar values are preserved (null, true, false, and numbers), strings are preserved, and sequences are converted to structured objects.
5 - Extension: Reserved for extensions

#### Property Identification/References
An accompanying value of 6 or greater is used to identify and reference properties. If a property identification succeeded by a property creation, the created property is assigned the ID defined by the accompanying number. This property can later be referenced and used and preceeded directly by a value, that will be interpreted using the given property. If a property with the given id has already been defined, that id should be replaced with the new property assignement.
If a property identifer succeeds a property creation token, the referenced properties fields (including its array values if a referencing type) should be copied to the created property, but the property key on the created property should not be altered.

### Structured Objects
By default, sequences are converted (except in the case of values converted with the Array property type). Sequences are converted to objects by iterating through the sequence. Each value in the sequence *should* have a property defined for the value, and the property's key is used to determine what property name in the parent object to apply each value to. The property for each value in the sequence may have a string key or numeric key. If the key is `null` this means that object should be replaced by the value. A property with a key of `undefined` should not be applied to the object (should not produce a property on the resulting object).

When an property's value produces a structured object, the first property in the object should be recorded in the property's "child" field, such that any future sequences that are deserialized with this property will default to using the same first property to read the value, and a property does not need to be specified for the first value.

Likewise, each subsequent property in the sequence should be assigned to the previous property's "next" field. And sequence that is converted to a strutured object that reads a sequence value with a property should default to using the property's "next" field as the default property for the reading the next value in the sequence (again a property does not need to be specified before the value if the property is correct for deserializing the next value in the seequence).

### Arrays
Like structured objects, arrays are created from sequences. For arrays, any property that defines (preceeds) a value in the sequence should be assigned to the "child" field of the parent array property, and should be used as the default for deserializing subsequent values in the sequence. Children of arrays should be interpreted using a default property type if no property has been defined on the sequence values.


### Blocks
Blocks can be used to embed pre-serialized dpack, and defer parsing of dpack structures. Again, the accompanying number minus 16 should indicate the number of bytes in the subsequent value (not including the block token). The state of any properties that are created or referenced in the block must be restored (including the referencing arrays, identified properties) after reading the block.

### Metadata
The metadata type allows additional information to be encoded in a property for the purposes of deserialization. This can include any type of extra metadata to describe the property. However, a metadata parameter that is a string should be interpreted as an indication of the name of the type or class that the value should be deserialized into. A structured object that is read with a property with metadata should generally result in the properties being applied to the properties of an instance of the class or type described by the string.

A metadata property definition may succeed another property creation, to define the metadata of that property, and using property for the first step of deserializing. For example a metadata property that specifies a "Date" type may be used on Number property, to allow for specifying a number for dates with decimal precision.

#### Standard Types
The following standard types (using metadata) should be handled by serializers and parsers if there is corresponding language support:
* Date: The value that should be converted from/to a date should be a serialized number in epoch milliseconds.
* Set: The value that should be converted from/to a Set should be a serialized array of values.
* Map: The value that should be converted from/to a Map should be a two element array, first element being an array of keys, and the second element being an array of values.

<a href="https://dev.doctorevidence.com/"><img src="./assets/powers-dre.png" width="203" /></a>
