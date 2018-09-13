# bh-tbd
## Beyond Here: TBD
TBD means "To Be Determined", or "There Be Dragons", depending on how you would like to interpret it. That is fitting as this effort is largely about letting parsers interpret messages as they see fit, in a structured heirarchical way.

### Status
Right now this is just an idea. This repo eventually will hold code, but for now just descriptions of the spec.

## Overview
This effort started after I worked a little bit with protocol buffers. It is a wonderful message spec system that mostly lets you specify a class of message shapes. This is very useful if the sender and reciever have already agreed on message content, but we quickly realized that we wanted to perform multiple levels of validation on the messages coming in to our system. While there are some light validation capabilities in proto2, the group behind protocol buffers is actively moving in the other way by stripping the ability to have required fields in proto3. 

Validation of course has a host of packages and libraries, and in theory one could segregate the two functions. However validation and message encoding are linked at a very deep level. A validation schema defines some messages which are invalid, and therefore limits the entropy of a group of messages tha follow a schema. The more restrictive the schema, the fewer bits will be needed to encode all possible valid messages.

Of course if you want to focus solely on compression, there are libraries to do as well, but they are often cpu intensive on either the encoding or decoding step, and there are a lot of features of good validation: heirarchical types, composting, some level of human interpribility, etc that are actually hindered by compression rather than helped.

But, I do believe there is an interesting middle ground here where we can build something useful focused on validation and efficient message encoding as a unifed whole. This is my attempt to see if that belief plays out.

## The Spec

The basic idea is to have messages with self-describing shape, and embeded type identifiers that allow validators to make efficient and reasonable assumptions about messages, while allowing schemas to bypass processing of information outside of the schema spec, trusting some other process or sub-type to understand it later. This strong and flexible typing is achieved by allowing sub types to extend their parent schemas, and for schemas to be grouped together into a message language. Importing groups of schemas into a new message language in a controlled manner allow for sharing of ideas and best practices around schema creation and use.

### Introducing the TBD
The most basic unit of a message is a TBD. A TBD consists of 3 required shape bytes, an optional 2-byte type identifier, and one or more data bytes. They are called TBDs because a schema will elect NOT to parse the content a TBD if the schema does not place any restrictions upon it's content. However, some processing of the TBDs shape will be needed to understand when a TBD ends. Luckily, all TBDs describe their own shape!

### Basic Shapes
There are 4 basic shapes (and 4 pointer types discussed later), encoded in 3 bits, and all shapes are paired with a size value encoded in 21 bits. The 4 shapes are:
* An anonymous object - (000) - In a properly formed message in a well defined schema language, it is garunteed that an appropriate schema will have enough information, based on the location and size of this blob, to understand and evaluate the content of the bytes. Size denotes number of bytes
* An identified object - (001) - Here, even an appropriate schema may need an additional information to understand the message segments content. A 2-byte type identifier will follow after the size bits, but before object content. A common example would be to identify one of many optional attributes on a type. Size denotes number of bytes.
* Array of multiple shapes - (010) - Each member will define it's own shape. Size is the number of member is the array.
* Array of multiple identical shapes - (011) - Here, there will be be only a single shape definition for all members in the array, in the bytes directly following this defintion. Note, because shape is only defined at a single level, the objects need only share identical top-level shape. Size is the number of member is the array.

These shapres are sufficient to encode all possible message contents. A completely naieve schema can simply break apart a message into it's component shapes.

### Introduction to Types
While self-describing messages are somewhat nice, they can also become very expensive very quickly. For example, a trivial array of `[121,3.14159265359,"ab"]` would take 26 bytes to encode (worse than json!): 3 to define the array of 3 members, 5 to define an int of 2 bytes, 2 for the int value, 5 for define a float of 4 bytes, 4 for the float value, 5 to define the str of 2 bytes, and 2 bytes for the string. 3 + 5 + 2 + 5 + 4 + 5 + 2 = 26

However, by placing a restrictive schema on our message, many of the bytes becomes superflous. The above message segment might be given the following schema:
```
new_type MyNewType(array):
  require ordered:
    int16,
    float32,
    str(length=2)
```

Using the schema, we know that MyNewType must consist of 3 ordered fields. If we optimistically assume (we will address how to deal with failure of assumptions later) that our message producers and consumers are using the same schema language, identifying that the message type provides all of the information needed to define the location of all values in the message. Now the same message can be encoded with just 13 bytes: 5 to define a MyNewType of 8 bytes, 2 bytes for a int16, 4 for a float32, and 2 for a str of length 2.

### Subtypes
 
### Pointers
* Pointers to objects - (100) - This signifies that the object here is identical in shape and content to another object defined earlier. Size is the number of bytes earlier in the message that the object is defined. Chaining pointers is allowed.
* Pointers to unidentified blobs - (101) - If the object being pointed to is a set of unidentified bytes, the pointer must provide a shape definition to aid in parsing.
* Pointer to object with split definition - (110) - If the object being pointed to is defined inside an array of multiple identical shapes, it's shape defintion will non be contiguous with it's content.
