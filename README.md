# DPack

DPack is a very compact binary format for serializing data structures, designed for efficient, high-performance serialization/parsing, and optimized for web use. For common large data structures in applications, a dpack file is often much smaller than JSON (and much smaller than MsgPack as well), and can be parsed faster than JSON and other formats. dpack leverages structural reuse to reduce size, and uses binary format for performance. dpack has several key features:
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
* Parsing of tokens into rudimentary values and sequences.
* Structuring of rudimentary values/structures into final data structures.

DPack is a binary data format, in the sense that structures are defined through byte-level tokens for machine parsing. But it is primarily specified as character-based format; blocks of data can be entirely decoded using a character set decoding as the first layer of reading, and then the parsing rules operate on the decoded characters. dpack is optimized for, should default to, UTF-8 encoding, but could be encoded in any character set that can encode unicode charaters 0-127. In addition, it can be encoded using additional characters for more efficient space usage when encoded in UTF-16. A DPack message should consist entirely of tokens and strings that are described by the tokens.

## Token Lexing
The basic entity in a dpack message is a token. A token may consist of 1 to 8 characters. The meaning of the characters in a token are determine by their unicode code point number, and further defined by bitwise positions. All token values are based on their character's unicode code point. The token character or characters are used to determine a `type` number from 0 to 3, and an `accompanying` number which is an unsigned integer up to 2^46. A dpack serialized data structure consists entirely of tokens and strings that are read by length specified by tokens, based on the parsing rules.

By default, all token character bytes have an initial 0 bit (if 0 - 127 byte range is used, as this makes it compatible with most character sets). The second bit is always a "stop" bit. A stop bit of one means this is the last byte, a zero means additional bytes are part of the token. In the first byte, the next two bits (3rd and 4th) represent the `type` of the token. The two bits are used to determine the type 0 - 3. The remaining bits, the first four bits of the first byte, and all remaining bytes (up to and including a byte with a stop bit) are used to serialize the accompanying number, which is interpreted by big endian bytes/bits, where the first bits are most significant, and later bytes/bits are less significant. However, if the type is 3 *and* the stop bit is 0, this is a special case where the type is changed to 7 and no further bytes are read (only the last four bits from the first byte are used to compute the number, as if the stop bit was set). For the sake of this specification, we represent tokens as `<type, number>`.
### Examples

Character "R": Code point 82. Binary representation:

`0 1 0 1 0 0 1 0` - Stop bit is set (1), type (0 1) is 1, and accompanying number (0 0 1 0) is 2, producing a token `<1, 2>`.

Character "!D": Code points 33 and 68. Binary representation:

`0 0 1 0 0 0 0 1   0 1 0 0 0 1 0 0` - Stop bit is not set (0), type (1 0) is 2, bits (0 0 0 1) will be combined with next byte. Next byte, stop bit is set. Combined bits (0 0 0 1  0 0 0 1 0 0) make 68 the accompanying number, producing token `<2, 68>`.

Character "4": Code point 52. Binary representation:

`0 0 1 1 0 1 0 0` - Stop bit is not set (0), type (1 1) is 3. However, this is the special case of type 3 and no stop bit so it is treated as type 7 (stopped), and accompanying number (0 1 0 0) is 4, producing token `<7, 4>`.

### Limits
There may be up to 8 bytes, which accomodates up to 46 bits for the accompanying number, therefore the accompanying number must be an unsigned integers under 2^46.

Some types will use the accompanying number to specify the length of a string immediately following the token. When a string is to be read, the number specifies the number of characters to be including in the string, after which the next block can be immediately read. The length of the string is not bytes, but basic multiplane characters. Any supplemental plane should be counted as two characters (surrogates) should be counted as two characters (a pair). In other words, a string length is defined by its UTF-16 encoding (though it may be serialized in UTF-8 in dpack).


### Example Code

This format is designed to be easily parsed with very simple and efficient code. Using standard C/C++/Java/C#/JS/TS syntax, type and number pairs can be lexed efficiently from characters/bytes with just a few lines of code:
```
token = dpackSource[position++] // read first character/byte
if (token >= 48) { // single byte token
	type = (token >>> 4) ^ 4; // shift and xor gives us the type
	number = token & 15; // last 4 bits is the number
} else { // multiple byte token
	type = (token >>> 4) & 11; // shift and omit the stop bit (bit 3)
	number = token & 15; // last 4 bits is high bits of number
	do {
		token = dpackSource[position++]; // get next byte
		number = (number << 6) + (token & 63); // progressively shift the number for big endian numbers
	} while (token < 64); // until a stop bit
```

### Alternate Encodings
Tokens may also consist of higher character codes and a compliant parser should also be to parse characters that extend beyond unicode 127. For characters with code points 128 and above, the character code point should be interpreted as a 16-bit unsigned integer, with the first bit always as 0, the second bit as a stop bit, the third and fourth bit as type bits, and the remaining 12 bits for the accompanying number. These 16-bit character encoding/decodings can be used for greater efficiency where UTF-16 encoding is preferred (which can be faster in languages that internally represent strings with UTF-16, and there is relatively unlimited socket bandwidth, such as interprocess pipes).

## Parsing Types
The `type` value indicates how the token (and any accompanying string) should be parsed into a rudimentary value. The `type` is two bits so there are four basic parse types:

### 1 - Number
This type code means the accompanying number should be directly interpreted as the value, no further parsing is needed for this value.

### 2 - String
The accompanying number indicates the number of following characters that compose a string. The string, following this token, with the length defined by the accompanying number, is the rudimentary value that is produced.

### 3 - Type Definition
This type code is used to define properties and special values. The accompanying number is used to determine the value or property to create. This is the definition of the accompanying number follows:

The first six accompanying number codes are for defining special constant values, and should be converted to these values:
* 0 ("p") - `null` (or `NIL` or `NULL` depending on language)
* 1 ("q") - Reserved
* 2 ("r") - Reserved
* 3 ("s") - `false`
* 4 ("t") - `true`
* 5 ("u") - `undefined` (used as value to indicate a property should be omitted)

The next 10 codes are property definition codes. The parser needs to read the next value after this token as the parameter for the property that is being created or modified.
The first property definition codes are property creation definitions, which define a new property for the current slot. The property definition code can be followed by the value/parameter which defines the *key* that is associated with this property. The value/parameter that defines the key can be elided; if the property code is followed by a sequence or another property definition token, the key is not specified and defaults to `null`. The codes below indicate which property type to be created (and each type defines how the rudimentary values are converted to final values):
* 6 ("v") - Default property type
* 7 ("w") - Array property type
* 8 ("x") - Referencing property type
* 9 ("y") - Numeric property type
* 10 - ("z") - Binary property type
The next three codes are used indicate to other types of modification to the current property. Again, the parser should read the next value. The next value is the parameter for the property modification, and in this case any type of token/value should be treated as the parameter. This may be used to modify a previously defined property, providing additional information about the property:
* 11 ("{") - Metadata - Provides metadata for a property, like a class name/converter that should be used for to transform values instances of specific classes.
* 12 ("|") - Copy property - The parameter should be a number or null, and a number indicates index position of the property that should be copied into this property. A null indicates that the parent of this property should be copied into this property. This is a shallow copy; the source property's type, key, parent, and array of child properties should be copied, but afterwards the target property should have a pointer or reference to the source's child properties. Multiple copy-properties can be sequentially applied to a single property slot to access grandparents and grandchild properties. A copy-property with a null parameter that is followed by a copy-property token can be collapsed to two consecutive copy-property tokens, omitting the null.
* 13 ("}") - Set Referencing Position - The parameter/value should be a number, and the current property should be a referencing property, this resets the index position in the set of referenceable values, for the addition of future strings and sequences. If the value is `null`, the position index is set to null, and future strings and sequences should not be added to the set of referenceable values (until the position is reset to a number).
* 14 ("\~") - Type definition - The parameter can be used to define the property's child properties. The parameter value should be read using the current property, so a sequence of property definitions will populate the child properties. The value returned has no effect or meaning, this used for parsing an object structure without having to return a value (the actual value/object instance can be parsed as the next value after the property).
* 15 - Reserved
(No number above 15 can be represented with the tokens since the stop bit is used to distinguish from type 7.)

### 7 - Start Sequence
A sequence is used to create objects and arrays that consist of multiple values. For an accompanying number of 11 or less, the accompanying number indicates the number of values should be read after this token, for this sequence. Alternately, an accompanying number of 15 indicates the start of an open sequence, which should be read as a sequence of values until the End Sequence token is encountered.

12 - "<" Start-Open-Sequence
13 - "=" Start-Partial-Deferred-Reference-Sequence
14 - ">" End-Open-Sequence
15 - "?" Deferred Reference (see section below)

(No number above 15 can be represented with the tokens since the stop bit is used to distinguish from type 3).

### 0 - Property Slot Index
The property slot index indicates which property slot the following property and/or value should use. The next value (the parameter) should be interpreted as the index position.

This defines the parsing of dpack into rudimentary sequences, strings, constants, numbers, and property definitions that describes the deserialization of these values. Next the values are converted to their final structured deserialized form.

### Examples of parsing tokens to rudimentary values
A token of <1, 2> (number) would parse to a plain integer `2`.

A token of <2, 12> (string), followed by next 12 characters "Hello, World", would parse to the string `"Hello, World"`.

A token of <3, 0> (type) would parse to `null`.

Tokens of <3, 9> <2, 3> "age" would parse to create a numeric property, with key name of "age", denoted as `number-property: "age"` for this specification.

A token of <0, 3> would parse to be a property slot index of 3, denoted as `property-slot-3` for this specification.

A token of <7, 2> would parse to indicate a sequence of length of 2, denoted as `sequence-2 [ value, value ]` for this specification.

Tokens of <7, 3> <3, 0> <1, 3> <3, 4> would parse to a sequence of three values: `sequence-3 [ null, 3, true ]`.

## Structuring
Structuring is the conversion of the rudimentary value, sequences, and property definitions into final usable data structures.

### Property Definitions
Properties are the central entity that describes how rudimentary values are deserialized into final data structures. Every rudimentary value that is parsed should have an associated property to determine its final value. A property has a type and a key (and potentially child properties and referenceable values). The first value in dpack document or stream uses the the root property which is the default type. Again, property definition has a token type code of 3, and the accompanying number defines the  property type that will be created, which will then be used to define how the values will be converted. A property defines how a corresponding rudimentary value is converted to its final value. The property type definitions with more detail are (by property code) are:

#### 6 - Default property type
Number and constant values are preserved (null, true, false, and numbers), strings are preserved, and sequences are converted to structured objects.

#### 7 - Array property type
All values are handled the same as default, except a sequence of values should be interpreted as an array of values.

#### 8 - Referencing property type
The property definition holds an array of referenceable values. Values are handled the same as default except that strings and sequences are stored in the next slot in the values array, and numbers are interpreted as a reference to a value by index position (starting at 0). For example, a number of 1 is a reference to the second value (string or sequence) that had been for this property should be used as the value. A reference can be a forward reference as well. When a property is defined or redefined using the referencing property type, the index position is set (or reset) to zero.

#### 9 - Numeric property type
All values are handled the same as default except that strings should be parsed as a number, following the same rules as JSON. For example, a string of "23.41" should be interpreted as the number `23.41`.

### Structured Objects
By default, sequences are converted (except in the case of values converted with the Array property type). Sequences are converted to objects by iterating through the sequence. Each value in the sequence *should* have a property defined for the value, and the property's key is used to determine what property name in the parent object to apply each value to. The property for each value in the sequence may have a string key or numeric key.

Every property can have an array of child properties. These are used as the properties for interpreting the values in a sequence. When a sequence is encountered, the current property's child properties will be used for the corresponding values. When the current property is a non-array property, this means a sequence is interpreted as a structured object, and the first value in a sequence is interpreted using the property in the first child property slot (index of 0) of the current property, the second value uses the property from the next slot (index of 1), and so on.

For the first sequence parsed for a given property, there won't be any property slots, so the properties must be defined. This is where the property definitions tokens are needed. For the first value in sequence, a property definition can preceed the actual value, to define the property for the first slot, which can then be succeeded by the value. Once the value is parsed or when another property definition is read, the index in the array of property slots is incremented, and a property definition can define the property for the slot and again can be followed by a rudimentary value that is interpreted by the property definition. Once slots have been defined, they can be reused to interpret value without redefining the property.

For the array property, the sequence are also interpreted by the child property(s), but the property slot index is not auto-incremented after each value is read. The property definition of the first slot can be defined before the first value, providing a property definition for property slot index 0, and then all subsequent values will use that same property slot (unless explicitly moved to a different index position).

For example:

`sequence-2 [ default-property: "name" "John", numeric-property: "age" 33 ]`

The first token indicates we are starting a sequence; since the root property is always default type, this means the sequence is a structure object with two values. The first value is preceeded by a Default Property definition, with a key of "name". This property is put in 0 position of the root property's child array. That property is then used to interpret the value of "John", which is preserved as a string, and assigned to the structured object's property named "name". The second value is preceeded by a Numeric Property definition, with a key of "age". This property is put in 1 position of the root property's child array. That property is then used to interpret the value of 33, which is preserved as a number, and assigned to the structured object's property named "age". This results in an object that would be represented in JSON as:
```
{
	"name": "John",
	"age": 33
}
```

Here is more complicated example where a `friends` array property is defined, which then specifies a default property for interpreting the elements of the array. The first element of the array is a sequence which specifies properties `name` and `age`. And then the second element of the array will be interpreted using the same property as the first element, and therefore the same child property slots are already defined (first slot is `name`, second is `age`), so the sequence can interpreted without definining properties again:
`sequence-1 [ array-property: "friends" sequence-2 [
	default-property: null sequence-2 [ default-property: "name" "John", numeric-property: "age" 33 ]
	sequence-2 [ "Sarah", 29 ] ] ]`
{
	"friends": [ {
		name": "John",
		"age": 33
	}, {
		name": "Sarah",
		"age": 29
	} ]
}

Note that if a property definition token is encountered after a previous property definition in an object, it should increment to the next slot. A property definition followed by a property update token should not, the property update should always apply to the current slot.


### Property Slot Index
The property slot index indicates which property slot the following property and/or value should use. By default, the property index starts at 0 when a sequence beings, and for objects structurs increments for each property/value (no auto-incrementation for arrays) in the sequence, but with the property slot index, the accompanying number explicitly indicates the current property index. This can be followed by a property to define a property at specific index slot in the child properties, and/or can be followed by value to use the property at a specific index. When the index is updated with this, the index can continue to auto-increment from the new position (or stay in the new position for arrays).


### Arrays
Like structured objects, arrays are created from sequences. When sequence when the current property is an array type, the seequence is to be interpreted as an array. Within an array sequence, a property with a key of `null` indicates that the value should be appended as the next value in the array. If no properties are defined within an array sequence, the array sequence should default to interpreting the values with the default property type, as if a key of `null` where each value is added to the array.

### Blocks
At the top level of a dpack document or stream, the serialization of a single root value (which may be a complex object with arbitrary size) is a "block". However, a document/stream may have multiple sequential blocks. The first block of the document is the root object/value, but there may be a queue of additional blocks that need to be parsed from deferred references. The blocks are simply dpack values in sequential order (with no delimiters). These are parsed in order as defined by the queuing of the deferred references.

### Deferred References
Deferred references allow the serialized representation of an object to be deferred or moved to a later point in a dpack document so that a parser can parse and deserialize that object later. This can be valuable mechanism for allowing a parent object to be parsed, while deferring the parsing of child objects until necessary, and can improve the utility of progressive parsers.

A deferred reference is represented by the single token/character <7, 15> ("?"). The parser should interpret this as a placeholder for a future object. And the parser should queue up this object to be read/deserialized as a block in the parsing queue in this order: The block for the deferred reference should come *after* all the deferred reference blocks from this block, but *before* all deferred reference blocks from any previous blocks. This ensures that a block can be contiguous with all the blocks for its own deferred references, even if that block is referenced by (deferred references in) other blocks.

When the block is parsed, the root/beginning of the block should be interpreted using the current property when the deferred reference was read.

### Metadata
The metadata type allows additional information to be encoded in a property for the purposes of deserialization. This can include any type of extra metadata to describe the property. However, a metadata parameter that is a string should be interpreted as an indication of the name of the type or class that the value should be deserialized into. A structured object that is read with a property with metadata should generally result in the properties being applied to the properties of an instance of the class or type described by the string.

A metadata property definition may succeed another property creation, to define the metadata of that property, and using property for the first step of deserializing. For example a metadata property that specifies a "Date" type may be used on Number property, to allow for specifying a number for dates with decimal precision.

#### Standard Metadata Class Types
The following standard types (using metadata) should be handled by serializers and parsers if there is corresponding language support:
* Date: The value that should be converted from/to a date should be a serialized number in epoch milliseconds.
* Set: The value that should be converted from/to a Set should be a serialized array of values.
* Map: The value that should be converted from/to a Map should be a two element array, first element being an array of keys, and the second element being an array of values.

<a href="https://dev.doctorevidence.com/"><img src="./assets/powers-dre.png" width="203" /></a>
