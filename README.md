This document describes a simple toy chat protocol "STCP", version 1.

TODO: each frame that assumes a response should contain a sort of identifier so that both sides could correlate frame responses to frame requests, this is integral part of multiplexing

Connection
==========

Any TCP-level protocol can be used as a network layer.
All interactions between client and server are encapsulated in frames.

Encoding frames
===============

This is a binary protocol hence each frame has its own structure.
All frames are divided in two sets: client frames and server frames.
All frames definitions are presented below:

#### Client frames
```haskell
Name       | Code     | Arguments
___________|__________|______________________________________________
HEY          0x01       (version: u64)
MYNAMEIS     0x02       (username: str8) (password: str8)
SUP          0x03       (token: str16)
REMEMBERME   0x04       void
LISTCLIENTS  0x05       void
MESSAGE      0x06       (message: str64)
LOGOUT       0x07       void
```

#### Server frames
```haskell
Name       | Code     | Arguments
___________|__________|__________________________
HEY          0x01       (version: u64)
SUP          0x02       void
NOPE         0x03       void
USERERROR    0x04       (code: u16) (message: str16)
NEWTOKEN     0x05       (token: str16)
CLIENTS      0x06       (length: u64) (usernames: str8[length])
ROOMEVENT    0x07       (event: room_event)
```

#### Type definitions
```rust
str8: (length: u8) (value: byte[length])
str16: (length: u16) (value: byte[length])
str64: (length: u64) (value: byte[length])

#          "{u8" declares a tagged union where tag is of u8 size, enumeration starts from 1
room_event: {u8: new_client(username: str8), client_left(username: str8), message(username: str8, message: str64)}
```

#### Basic type interpretation
```rust
u8:  unsigned integer of 8 bit width
u16: unsigned integer of 16 bit width
u64: unsigned integer of 64 bit width
byte: equivalent to u8, different semantic meaning
```

Handshake
=========

Initial handshake is done with a `HEY` frame. A client has to send `HEY` frame
  with preferred version of a protocol to the server after TCP connection is
  established. A server must respond with `HEY` frame with the protocol version
  that it chose to use (if server implements more than one version it can choose
  the closest one to the client's preferred choice). A client then has to
  decide if it will communicate in this protocol, otherwise it can disconnect
  or initiate a new `HEY` handshake.

E.g. for a client that wants to talk version 2 and server agreeing to it, would
  look like:
```haskell
-> 0x01 0x02
<- 0x01 0x02
```

Authentication
==============

After successful handshake a client needs to authenticate itself. It can be
  done with two frames: either `MYNAMEIS` or `SUP` client frame (SUP stands
  for "what's up"). You can think of `MYNAMEIS` as a registration and login
  on a website, combined. And you can think of `SUP` as login with a cookie.
When server gets a `MYNAMEIS` frame, it has to search if given username is
  already registered. If so server needs to verify the password, otherwise to
  register them.
If the username/password pair is incorrect or `SUP` token doesn't exist/too old
  the server responds with `NOPE` frame.
In order for the client to get the cookie token to authenticate with `SUP`, it
  has to send a `REMEMBERME` frame after it has already authenticated.

Here's an example of a client registering and then asking for a cookie:
```haskell
#  MYNAMEIS (3 "nom") (4 "pass")
-> 0x02 0x03 0x6e 0x6f 0x6d 0x04 0x70 0x61 0x73 0x73
<- 0x02
-> 0x04
#  NEWTOKEN (16 "cpb88ujiir3qi6yr")
<- 0x05 0x00 0x10 0x63 0x70 0x62 0x38 0x38 0x75 0x6a 0x69 0x69 0x72 0x33 0x71 0x69 0x36 0x79 0x72
```

Chatting
========

After the client was authenticated it is placed in the default and the only
  room.
When a client is in the room, it is subscribed to all of the events of this
  room. This means that at any time it has to expect `ROOMEVENT` frame to be
  received.
A client can ask for a list of all clients that are currently in this room
  with a `LISTCLIENTS` frame and receive a `CLIENTS` frame in response.
Note that after issuing a `LISTCLIENTS` frame client has to expect `CLIENTS`
  *or* a `ROOMEVENT` frame. If `ROOMEVENT` comes first `CLIENTS` should still be
  expected.
`ROOMEVENT` type is a tagged union, it means that it can be of any of the listed
  types and the type is decided upon a tag. For example in order to encode
  `new_client` event one would have to write a tag of `new_client` variant first
  and then the contents of the `new_client` itself. Here is an example of
  `new_client` and `message` events coming to our client:

```haskell
# ROOMEVENT (room_event.new_client (3 "nom"))
-> 0x07 0x01 0x03 0x6e 0x6f 0x6d
# ROOMEVENT (room_event.message (3 "nom") (8 "sup guys"))
-> 0x07 0x03 0x03 0x6e 0x6f 0x6d 0x08 0x73 0x75 0x70 0x20 0x67 0x75 0x79 0x73
```

In order to send the message one has to issue a `MESSAGE` client frame. When
  server gets a `MESSAGE` frame from a client it has to broadcast this message
  in `ROOMEVENT` frame to all other clients.
