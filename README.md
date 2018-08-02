# dpack

dpack is a very compact binary format for serializing data structures, designed for efficient, high-performance serialization/parsing, and optimized for web use. For common large data structures in applications, a dpack file is often much smaller than JSON (and much smaller than MsgPack as well), and can be parsed faster than JSON and other formats. dpack leverages structural reuse to reduce size, and uses binary format for performance. dpack has several key features:
* Uses internal referencing and reuse of structures, properties, values, and objects for remarkably compact serialization and fast parsing.
* Defined as a valid unicode character string, which allows for single-pass text decoding for faster and simpler decoding (particulary in browser), support across older browsers, and ease of manipulation as a character string. It  can also be encoded in UTF-8, UTF-16, or any ASCII compatible encoding.
* Supports a wide range of types including strings, decimal-based numbers, booleans, objects, arrays, dates, maps, sets, and user-provided classes/types.
* Supports positionally mapped object properties for lazy evaluation of paths for faster access to data without parsing entire data structures (useful for storing, querying, and indexing data in databases).
* Supports referencing of objects which can be used to reorder serialization and reuse objects.
* Optimized to compress well with Huffman/Gzip encoding schemes

This repository is for the specification of dpack. For dpack libraries:
* (JavaScript implementation)[https://github.com/DoctorEvidence/dpackjs]

# Specification
dpack is designed for ease of creating high performance implementations, making it easy to use fast bitwise operators, and memory-limited structures. While the dpack messages may appear somewhat complicated description, it is intended to be easily implemented, particularly with typed languages, to be flexible, and to provide protection against excessive memory consumption.

dpack is a binary data format, in the sense that structures are defined through byte-level tokens for machine parsing. But it is primarily specified as character-based format; blocks of data can be entirely decoded using a character set decoding, and then the parsing rules operate on the decoded characters. dpack is optimized for, should default to, UTF-8 encoding, but could be encoded in any character set that can encode unicode charaters 0-127. In addition, it can be encoded using additional characters for more efficient space usage when encoded in UTF-16.

## Tokens/Character Decoding
The basic entity in a dpack message is a token. A token may consist of 1 to 8 characters. The meaning of the characters in a token are determine by their unicode code point number, and further defined by bitwise positions. All token characters should be converted to a character code. The token character or characters are used to determine a `type` number from 0 to 3, and an `accompanying` number which is an unsigned integer up to 2^46. A dpack serialized data structure consists entirely of tokens and strings that are read by length specified by tokens, based on the parsing rules.

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

## Type and Reading Mode
The `type` is combined with the reading mode to determine the type of the value being parsed. The following reading modes and types are specified. Again, parsing should always start in open mode:

### Open Mode
In open mode, `type` is interpreted as:
* type 0: Special types. The value is determined by the accompanying number:
Constants:
* 0 - null
* 1 - false
* 2 - true
* 3 - undefined

Value Types:
* 5 - object - Read the next token in object mode and return it.
* 6 - string - Read the next token in string mode and return it.
* 7 - open - Read the next token in open mode (no-op) and return it.
* 8 - extended type - read the next token in open mode to determine type, and read the next token in open mode to get value to provide for extension.
* 9 - error - Read the next token in open mode to determine error that occurred.

Block level types (can only occur at the top level of a block):
* 12 - Value with identifier. Read the next value in open mode, to determine the id, and the next data value in open mode to determine the value/object to associate with the id.
* 13 - Begin block and define length in bytes of the next section data. Read the next value in open mode as a number to determine length in bytes, and then the next token after that in open mode, as a separate block, to determine value. The length may alternately be specified as 0 if the length is unknown.
* 14 - End block - This token marks the end of the block, and all structure and property tables and ids within that block can be discarded for subsequent parsing.
* 15 - Library import - This allows a library of structures/properties to be imported into the current block. The first value should be read in open mode, and identifies the library to import. The properties and structures within the current block should be updated as if the identified library had been parsed as part of this block. The second value should be read open mode and specify the current/next id of the next object structure. The third value should be read in open mode should specify the current/next id of the next property. Finally, the next value should be read in open mode, with the updated state of the block, and returned.

* type 1: An unsigned integer less 2^46. The accompanying number is the actual parsed value (an unsigned integer).
* type 2: A string representing a decimal number. The accompanying number is the length of the subsequent string, which should be serialized as a number. For example, the following string could be "-2.32". This string should parsed as a number according to standard Java/C/JavaScript number rules. This can also be "Infinity" or "-Infinity".
* type 3: An array of values. This should return an array of values, each to be successively read in open mode. The accompanying number specifies the number of values in the array.

#### Defining Blocks
Blocks are the container for a sequence dpack serialized and referenceable structures, properties, and property values. Each block has its own set of structure, property, and values (each starting with the same ids). As specified as the open mode, the beginning of blocks of data can be specified with type 0/#12 in open mode, and can define sub-blocks within other blocks. Block definition token are also the only token that specify a length in bytes instead of characters. This makes it possible for binary data to be parsed, looking for blocks (which may be preceded by identifiers using type 0/#9), and character decoding of the contents of blocks to be deferred until its data needs to be parsed. For example, a multiple block data structure could be defined:

type 0/#12 begin-block | specify 20 bytes | (block of 20 bytes of utf-8 encoded characters, including an object reference to object with id of 1 ) | type 0/#9 id-value | id 1 | type 0/#12 begin-block | specify 30 bytes | (block of 30 bytes of utf-8 encoded characters defining object with id of 1)

When a block starts, all the structure, property, and value references start anew. When a block ends, these can all be discarde, and any parent block state of references should resume.

### Structured Object Mode
In object mode, `type` is interpreted:
* type 0: Object identify/reference - Identifies this objects, connecting it to an identity, that may be defined or used elsewhere. The accompanying number defines its id. The next value should be read in object mode to read any properties assigned to it immediately. An id of zero indicates the value is null.
* type 1: Object with inline property structure. An structured object is serialized and will be parsed by first reading a structure definition of the properties, and then the values of the property. The accompanying number defines the number of properties (and values). The property structure is parsed in property mode, with the given number of properties, which define the property names and types that will be created on the parsed object. This is followed by the property values, in the same order and position as the property definitions. The corresponding property definition defines which read mode to use for each value (i.e. if the first property defines string type, the first value should be read in string mode).
* type 2: Object with structure reference. The accompanying number is a reference to a previously defined property structure. The structure reference is followed by the values to parsed and assigned to the object based on the position and type as defined in the referenced structure. The referenced structure is a back-reference, and is found by counting sequentially in reverse through the dpack document/file/stream by the inline-defined structures. For example, if the accompanying number is 1, then the last (most recently) defined property structure before this point in the document, is the referenced structure. A number of 2 would be the second to last (0 never used as a back-reference, it should be parsed as null).
type 3: An array of objects, each to be successively read in object mode. The accompanying number specifies the number of objects in the array.

#### Object and Property Referencing
Objects structures and properties can be reused by referencing them. Identification of the structures and properties is automatically assigned by incrementation by when a structure or property appears in a dpack block. Ids start at 16. The first structure (object with inline property structure) is given an id of 16, the next an id of 17, and so on. The same is true of property declarations, property declarations have their own separate ids starting at 0 (incremented separately from structures). Ids restart to 16 when they reach 1024. Also, everytime a struture or poperty is defined *or* referenced it is also should have an id recorded for its last 4 bits. When the first structure is given an id of 16, it can also be referenced by id of 0. If the 30th declared object structure, with an id of 45 is referenced by id of 45, the last four bits (3), it can immediately be referenced again by id of 3 (until another id with 3 in the last four bits overwrites its slot).

### String Mode
In string mode, `type` is interpreted:
* type 0: Non-referenceable string. The next characters form the string value, and the accompanying number defines the length of the string in characters. An accompanying number of 0 indicates a value of null.
* type 1: Referenceable string. The next characters form the string value, and the accompanying number defines the length of the string in characters. This string value can be referenced using a back reference. The ids for a string reference start at 0, and are scoped to the specific property that this value is defined for. The property may have been defined in an object with structure through references or inline. Each object that uses this property, and this type increments the id, adding a new string that can be later referenced. The ids for string properties restart at 0 when incremented to 16 (ids are 4-bit references).
* type 2: String reference. The accompanying number is a reference by id to a previous referenceable string value for this property, according to the identification described above. For example, a property "foo" could have a value of "bar", serialized with type 1, and the next object instance could also have a property "foo" with value of "bar" and reference it with type 2 and accompanying number 0.
* type 3: An array of strings. Each value is to be successively read in string mode. The accompanying number specifies the number of strings in the array.

### Property Mode:
In property mode, `type` is interpreted:
* type 0: Property reference. This references a previously declared property according to the identification/referencing rules above, where the accompanying number is the id of the property that should be used here.
* type 1: Object type. The accompanying number defines the length of the string of immediately preceding characters that define the property name. When reading the values for this object, this property's value should be read in object mode.
* type 2: String type. The accompanying number defines the length of the string of immediately preceding characters that define the property name. When reading the values for this object, this property's value should be read in string mode.
* type 3: Open type. The accompanying number defines the length of the string of immediately preceding characters that define the property name. When reading the values for this object, this property's value should be read in open mode.

In property mode, if the accompanying number is 0, the property is read as an extended property, following a different parsing rule. When the accompanying number is 0, the next value should be read in open mode, and this defines property name (may be string, but also allows for non-string property names). And the value after that should also be read in open mode, and this specifies the metadata for the property. The property metadata can be used to define the property type that may be used to convert property values to certain language types. A property metadata value that is a string should be understood to be a string identifying the name of a type that the value should be converted or coerced to.

#### Standard Types
The following standard types should be handled by serializers and parsers if there is corresponding language support:
* Date: The value that should be converted from/to a date should be a serialized number in epoch milliseconds.
* Set: The value that should be converted from/to a Set should be a serialized array of values.
* Map: The value that should be converted from/to a Map should be a two element array, first element being an array of keys, and the second element being an array of values.

<a href="https://dev.doctorevidence.com/"><img src="./assets/powers-dre.png" width="203" /></a>
