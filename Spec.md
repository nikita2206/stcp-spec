
This is a document describing a toy chat protocol.

Connection
==========

Any TCP-level or higher level protocol can be used as a network layer.
All interactions between client and server are encapsulated in frames which shall start with zero-byte and end with zero-byte.
A content of a frame can contain zero-bytes as well, in this case they have to be escaped as a `\\0` sequence (backslash, zero-byte).

Handshake
=========



