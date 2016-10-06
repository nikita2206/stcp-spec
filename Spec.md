This is a document describing a toy chat protocol.

Connection
==========

Any TCP-level or higher level protocol can be used as a network layer.
All interactions between client and server are encapsulated in frames which shall start with zero-byte and end with zero-byte.
A content of a frame can contain zero-bytes as well, in this case they have to be escaped as a `\\0` sequence (backslash, zero-byte).

Encoding frames
===============

This is a binary protocol hence each frame has its own binary structure. All frames are divided in two sets: client frames and server frames.

#### Client frames
```
Name Code Arguments
HEY 0x1 (version: ui64)
```

#### Server frames
```
Name Code Arguments
HEY 0x1 (version: ui64)
```

Handshake
=========

Initial handshake is done via `HEY` frame. A client has to send `HEY` frame with preferred version of a protocol to the server after TCP connection is established. A server must respond with `HEY` frame with current version of a specification used which 

Authentication
==============

A client has to authenticate now in order to