# sigfox

# Callback

When creating URL callbacks, you can specify the endpoint of the script that will receive the callbacks sent by Sigfox, However, if you omit to specify this, then the "defaultCallback" script is used by default and will therefore be invoked by Sigfox upon message availability.

"defaultCallback" uses the "parser" script to parse parameters sent along with the request it receives. The latter also forwards the parsing 
to a specific parser, if one is defined. Hence, you can implement your own parser and specify its path in the "specificParser" variable of the "parserconfig" script.

The "defaultCallback" script does not implement any specific logic and is just a pass through. It will however invoke the "execute(data)" function of the scripts of which path if declared in the "handlers" variable of the "parserconfig" script. You can hence implement callback handler scripts to handle the data that is sent by Sigfox.

## Defining the message template to be used by the callback

When creating a callback on Sigfox, you can specify a message template (bodyTemplate field when creating a URL callback, lineTemplate field when creating a Batch URL callback and message when creating an Email callback).

The message template can be parameterized with the following: device},{time},{duplicate},{snr},{station},{data},{avgSnr},{lat}{lng}{rssi},{seqNumber}

The template's structure will vary depending on the channel you are using (URL, Batch URL, Email) and for the URL channel, depending on the specified message type. Note that default templates are predefined by the connector if you do not specify anything yourself.

- URL + content-type application/x-www-form-urlencoded: "device={device}&data={data}&time={time}"
- URL + content-type application/json: {"device":"{device}","data":"{data}","time":"{time}"}
- URL + content-type text/plain: "device:{device}; data:{data}; time:{time}"

## Understanding the callback payload 

When creating a callback, it is necessary to define the format of the expected payload, in order to allow Sigfox to understand how to parse the data it is sending. The below is taken from Sigfox's API documentation:

The "custom format" grammar is as follows :
- format = field_def [" " field_def]* ;
- field_def = field_name ":" byte_index ":" type_def ;
- field_name = (alpha | digit | "#" | "_")* ;
- byte_index = [digit*] ;
- type_def = bool_def | char_def | float_def | uint_def ;
- bool_def = "bool:" ("0" | "1" | "2" | "3" | "4" | "5" | "6" | "7") ;
- char_def = "char:" length ("0" | "1" | "2" | "3" | "4" | "5" | "6" | "7");
- float_def = "float:" ("32" | "64") [ ":little-endian" | ":big-endian" ] ("0" | "1" | "2" | "3" | "4" | "5" | "6" | "7");
- uint_def = "uint:" ["1" - "64"] [ ":little-endian" | ":big-endian" ] ("0" | "1" | "2" | "3" | "4" | "5" | "6" | "7");
- int_def = "int:" ["1" - "64"] [ ":little-endian" | ":big-endian" ] ("0" | "1" | "2" | "3" | "4" | "5" | "6" | "7");
- length = number* ;
- digit = "0" | "1" | "2" | "3" | "4" | "5" | "6" | "7" | "8" | "9"

A field is defined by its name, its position in the message bytes, its length and its type :
- the field name is an identifier including letters, digits and the '-' and '_' characters.
- the byte index is the offset in the message buffer where the field is to be read from, starting at zero. If omitted, the position used is the current byte - for boolean fields, the next byte for all other types if the previous element has no bite offset and the last byte used if the previous element has a bite offset. For the first field, an omitted position means zero (start of the message buffer)
Next comes the type name and parameters, which varies depending on the type :
- boolean : parameter is the bit position in the target byte
- char : parameter is the number of bytes to gather in a string, and optionally the bit offset where to start the reading of the first byte, Default value is 7 for the offset
- float : parameters are the length in bits of the value, which can be either 32 or 64 bits, optionally the endianness for multi-bytes floats, and optionally the bit offset where to start the reading of the first byte. Default is big endian and 7 for the offset. Decoding is done according to the IEEE 754 standard.
- uint (unsigned integer) : parameters are the number of bits to include in the value, optionally the endianness for multi-bytes integers, and optionally the - bit offset where to start the reading of the first byte. Default is big endian and 7 for the offset.
int (signed integer) : parameters are the number of bits to include in the value, optionally the endianness for multi-bytes integers, and optionally the bit - offset where to start the reading of the first byte. Default is big endian and 7 for the offset.

## Examples :
Format	                                                        Message (in hex)	        Result
int1::uint:8 int2::uint:8	                                    1234	                    { int1: 0x12, int2: 0x34 }
b1::bool:7 b2::bool:6 i1:1:uint:16	                            C01234	                    { b1: true, b2: true, i1: 0x1234 }
b1::bool:7 b2::bool:6 i1:1:uint:16:little-endian	            801234	                    { b1: true, b2: false, i1: 0x3412 }
b1::bool:7 b2::bool:6 i1:1:uint:16:little-endian i2::uint:8	    80123456	                { b1: true, b2: false, i1: 0x3412, i2:0x56 }
str::char:6 i1::uint:16 i2::uint:32	                            41424344454601234567890A	{ str: “ABCDEF”, i1: 0x123, i2:0x4567890A }
str::char:6 i1::uint:16 i2::uint:32:2	                        41424344454601234567890A	{ str: “ABCDEF”, i1: 0x123, i2:0x167890A }
