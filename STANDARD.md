# netcode.io 1.0

**netcode.io** is a simple protocol for creating secure client/server connections over UDP.

This document describes the standard for this protocol so that people can create their own implementations.

## Architecture

There are three main components in a netcode.io-based architecture:

1. The web backend
2. Dedicated servers
3. Clients

The web backend is a typical web server, for example nginx, which authenticates clients and provides a REST API. Clients are endpoints running the netcode.io protocol that want to connect to dedicated server instances. Dedicated servers are instances of the server-side portion of the game or application running in data centers or the cloud.

The sequence of operations for a client connect are:

1. A client authenticates with the web backend
2. The authenticated client requests to play a game via REST call to the web backend
3. The web backend generates a _connect token_ and returns it to that client over HTTPS
4. The client uses the connect token to establish a connection with a dedicated server over UDP
5. The dedicated server runs logic to ensure that only clients with a valid connect token can connect to it
6. Once a connection is established the client and server exchange encrypted and signed UDP packets

## General Conventions

**netcode.io** is a binary protocol. 

All data is written in little-endian byte order unless otherwise specified.

This applies not only to packet data, but also to sequence numbers converted to byte array nonce values, and to associated data passed in to AEAD encryption primitives.

## Connect Token

A _connect token_ ensures that only authenticated clients can connect to dedicated servers.

The connect token has two parts: public and private.

The private portion is encrypted and signed with a private key shared between the web backend and dedicated server instances.

Prior to encryption the private connect token data has the following binary format.

    [client id] (uint64) // globally unique identifier for an authenticated client
    [num server addresses] (uint32) // in [1,32]
    <for each server address>
    {
        [address type] (uint8) // value of 0 = IPv4 address, 1 = IPv6 address.
        <if IPV4 address>
        {
            // for a given IPv4 address: a.b.c.d:port
            [a] (uint8)
            [b] (uint8)
            [c] (uint8)
            [d] (uint8)
            [port] (uint16)
        }
        <else IPv6 address>
        {
            // for a given IPv6 address: [a:b:c:d:e:f:g:h]:port
            [a] (uint16)
            [b] (uint16)
            [c] (uint16)
            [d] (uint16)
            [e] (uint16)
            [f] (uint16)
            [g] (uint16)
            [h] (uint16)
            [port] (uint16)
        }
    }
    [client to server key] (32 bytes)
    [server to client key] (32 bytes)
    [user data] (256 bytes) // user defined data specific to this protocol id
    <zero pad to 1024 bytes>

This data is variable size but for simplicity is written to a fixed size buffer of 1024 bytes. Unused bytes are zero padded.

Encryption of the connect token private data is performed with the libsodium AEAD primitive *crypto_aead_chacha20poly1305_encrypt* using the following binary data as the _associated data_: 

    [version info] (13 bytes)       // "NETCODE 1.00" ASCII with null terminator.
    [protocol id] (uint64)          // 64 bit value unique to this particular game/application
    [expire timestamp] (uint64)     // 64 bit unix timestamp when this connect token expires

The nonce used for encryption is a 64 bit sequence number that starts at zero and increases with each connect token generated. 

Encryption is performed on the first 1024 - 16 bytes in the buffer, leaving the last 16 bytes to store the HMAC:

    [encrypted private connect token] (1008 bytes)
    [hmac of encrypted private connect token] (16 bytes)

Post encryption, this is referred to as the _encrypted private connect token data_.

Together the public and private data form a _connect token_:

    [version info] (13 bytes)       // "NETCODE 1.00" ASCII with null terminator.
    [protocol id] (uint64)          // 64 bit value unique to this particular game/application
    [create timestamp] (uint64)     // 64 bit unix timestamp when this connect token was created
    [expire timestamp] (uint64)     // 64 bit unix timestamp when this connect token expires
    [connect token sequence] (uint64)
    [encrypted private connect token data] (1024 bytes)
    [num_server_addresses] (uint32) // in [1,32]
    <for each server address>
    {
        [address_type] (uint8) // value of 0 = IPv4 address, 1 = IPv6 address.
        <if IPV4 address>
        {
            // for a given IPv4 address: a.b.c.d:port
            [a] (uint8)
            [b] (uint8)
            [c] (uint8)
            [d] (uint8)
            [port] (uint16)
        }
        <else IPv6 address>
        {
            // for a given IPv6 address: [a:b:c:d:e:f:g:h]:port
            [a] (uint16)
            [b] (uint16)
            [c] (uint16)
            [d] (uint16)
            [e] (uint16)
            [f] (uint16)
            [g] (uint16)
            [h] (uint16)
            [port] (uint16)
        }
    }
    [client to server key] (32 bytes)
    [server to client key] (32 bytes)
    [timeout seconds] (uint32)          // number of seconds with no packets before client times out
    <zero pad to 2048 bytes>

This data is variable size but for simplicity is written to a fixed size buffer of 2048 bytes. Unused bytes are zero padded.

## Challenge Token

Challenge tokens stop clients with spoofed IP packet source addresses from connecting to servers.

Prior to encryption, challenge tokens have the following structure:

    [client id] (uint64)
    [hmac of encrypted private connect token data] (16 bytes)
    [user data] (256 bytes)
    <zero pad to 300 bytes>
    
Challenge tokens are encrypted with the libsodium _crypto_secretbox_easy_ primitive, with a random key that is generated each time a dedicated server is started and a sequence number that starts at zero and increases with each challenge token generated.

Encryption is performed on the first 300 - 16 bytes, and the last 16 bytes store the HMAC of the encrypted buffer:

    [encrypted challenge token] (284 bytes)
    [hmac of encrypted challenge token data] (16 bytes)
    
This is referred to as the _encrypted challenge token data_.

## Packets

**netcode.io** has the following packets:

* _connection request packet_ (0)
* _connection denied packet_ (1)
* _connection challenge packet_ (2)
* _connection response packet_ (3)
* _connection keep alive packet_ (4)
* _connection payload packet packet_ (5)
* _connection disconnect packet packet_ (6)

The first packet type _connection request packet_ (0) is not encrypted and has the following format:

    0 (uint8) // prefix byte of zero
    [version info] (13 bytes)       // "NETCODE 1.00" ASCII with null terminator.
    [protocol id] (8 bytes)
    [connect token expire timestamp] (8 bytes)
    [connect token sequence number] (8 bytes)
    [encrypted private connect token data] (1024 bytes)
    
All other packet types are encrypted. 

Prior to encryption, packet types >= 1 have the following format:

    [prefix byte] (uint8) // non-zero prefix byte
    [sequence number] (variable length 1-8 bytes)
    [per-packet type data] (variable length according to packet type)

The low 4 bits of the prefix byte contain the packet type. The high 4 bits contain the number of bytes for the sequence number in the range [1,8]. 

The sequence number is encoded by omitting high zero bytes. For example, a sequence number of 1000 is 0x000003E8 and requires only three bytes to send its value. Therefore, the high 4 bits of the prefix byte are set to 3 and the sequence data written to the packet is:

    0x8,0xE,0x3
    
The sequence number bytes are _reversed_ when written to the packet like so:

    <for each sequence number byte>
    {
        write_byte( sequence_number & 0xFF )
        sequence_number >>= 8
    }

After the sequence number comes the per-packet type data:

_connection denied packet_:

    <no data>

_connection challenge packet_:

    [challenge token sequence] (uint64)
    [encrypted challenge token data] (360 bytes)
    
_connection response packet_:

    [challenge token sequence] (uint64)
    [encrypted challenge token data] (360 bytes)

_connection keep-alive packet_:

    [client index] (uint32)
    [max clients] (uint32)
    
_connection payload packet_:

    [user payload data] (1 to 1200 bytes)
    
_connection disconnect packet_:
    
    <no data>

The per-packet type data is encrypted using the libsodium AEAD primitive *crypto_aead_chacha20poly1305_encrypt* with the following binary data as the _associated data_: 

    [version info] (13 bytes)       // "NETCODE 1.00" ASCII with null terminator.
    [protocol id] (uint64)          // 64 bit value unique to this particular game/application
    [prefix byte] (uint8)           // prefix byte in packet. stops an attacker from modifying packet type.

Packets sent from client to server are encrypted with the client to server key in the connect token.

Packets sent from server to client are encrypted using the server to client key in the connect token for that client.

Post encryption, packet types >= 1 have the following format:

    [prefix byte] (uint8) // non-zero prefix byte: ( (num_sequence_bytes<<4) | packet_type )
    [sequence number] (variable length 1-8 bytes)
    [encrypted per-packet type data] (variable length according to packet type)
    [hmac of encrypted per-packet type data] (16 bytes)

## Reading Encrypted Packets

The following steps shall be taken when reading an encrypted packet:

* If the packet size is less than 18 bytes then it is to small to possibly be valid, ignore the packet.

* If the low 4 bits of the prefix byte is greater than or equal to 7, the packet type is invalid, ignore the packet.

* The server ignores packets with type _connection challenge packet_. 

* The client ignores packets with type _connection request packet_ and _connection response packet_.

* If the high 4 bits of the prefix byte (sequence bytes) are outside the range [1,8], ignore the packet.

* If the packet size is less than 1 + sequence bytes + 16, it cannot possibly be valid, ignore the packet.

* If the packet type fails the replay protection test, ignore the packet. _See the section on replay protection below for details_.

* If the per-packet type data fails to decrypt, ignore the packet.

* If the per-packet type data size does not match the expected size for the packet type, ignore the packet.

    * 0 bytes for _connection denied packet_
    * 308 bytes for _connection challenge packet_
    * 308 bytes for _connection response packet_
    * 8 bytes for _connection keep-alive packet_
    * [1,1200] bytes for _connection payload packet_
    * 0 bytes for _connection disconnect packet_

* If all the above checks pass, the packet shall be processed.

## Replay Protection

Replay protection stops an attacker from recording a valid packet and replaying it back at a later time in an attempt to break the protocol.

To support replay protection, netcode.io takes the following steps:

* Encrypted packets are sent with 64 bit sequence numbers that start at zero and increase with each packet sent.

* The sequence number is included in the packet header and can be read by the receiver of a packet prior to decryption.

* The sequence number is used as the nonce for packet encryption, so any modification to the sequence number fails the encryption signature check.

The replay protection algorithm is as follows:

1. Any packet older than the most recent sequence number received, minus the _replay buffer size_, is discarded on the receiver side.

2. When a packet arrives that is newer than the most recent sequence number received, the most recent sequence number is updated on the receiver side and the packet is accepted.

3. If a packet is within _replay buffer size_ of the most recent sequence number, it is accepted only if its sequence number has not already been received, otherwise it is ignored.

Replay protection is applied to the following packet types on both client and server:

* _connection keep alive packet_
* _connection payload packet_
* _connection disconnect packet_

The replay buffer is implementation specific, but as a guide, at least a few seconds worth of packets at a typical send rate should be supported. Conservatively, a replay buffer size of 256 entries per-client should be sufficient for most applications.

## Client State Machine

The client has the following states:

* _connect token expired_ (-6)
* _invalid connect token_ (-5)
* _connection timed out_ (-4)
* _connection response timed out_ (-3)
* _connection request timed out_ (-2)
* _connection denied_ (-1)
* _disconnected_ (0)
* _sending connection request_ (1)
* _sending connection response_ (2)
* _connected_ (3)

The initial state is disconnected (0). Negative states represent error states. The goal state is _connected_ (3).

### Request Connect Token

When a client wants to connect to a server, it requests a _connect token_ from the web backend. 

The following aspects are outside the scope of this standard:

1. The mechanism the client uses to request a connection token from the web backend.

2. The mechanism the web backend uses to determine the set of server addresses to include in a connect token.

Once the client has obtained a connect token, its goal is to establish connection to one of the server addresses in the connect token.

To begin this process, it transitions to _sending connection request_ with the first server address in the connect token.

### Sending Connection Request

While in _sending connection request_ the client sends _connection request packets_ to the server at some rate, like 10HZ. 

When the client receives a _connection challenge packet_ from the server, it stores the challenge token data and transitions to _sending challenge response_. This represents a successful transition to the next stage in the connection process.

All other transitions from _sending connection request_ are failure cases. In these cases the client attempts to connect to the next server address in the connect token (eg. transitioning to _sending connection request_ state with the next server address in the connect token). Alternatively, if there are no additional server addresses to connect to, the client transitions to the appropriate error state as described in the next paragraph.

If a _connection request denied_ packet is received while in _sending connection request_ the client transitions to _connection denied_. If neither a _connection challenge packet_ or a _connection denied packet_ are received within the client timeout period specified in the connect token, the client transitions to _connection request timed out_.

### Sending Challenge Response

While in _sending challenge respeonse_ the client sends _challenge response packets_ to the server at some rate, like 10HZ. 

When the client receives a _connection keep-alive packet_ from the server, it stores the client index and max clients from the keep-alive packet, and transitions to _connected_. 

Any _connection payload packets_ received prior to _connected_ are discarded.

All other transitions from _sending challenge response_ are failure cases. In these cases the client attempts to connect to the next server address in the connect token (eg. transitioning to _sending connection request_ with the next server address in the connect token). Alternatively, if there are no additional servers addresses to connect to, the client transitions to the appropriate error state as described in the next paragraph.

If a _connection request denied_ packet is received while in _sending challenge response_ the client transitions to _connection denied_. If neither a _connection keep-alive packet_ or a _connection denied packet_ are received within the client timeout period specified in the connect token, the client transitions to _challenge response timed out_.

### Connect Token Expired

If the entire client connection process (potentially across multiple server addresses) takes long enough that the connect token expires before successfully connecting to a server, the client transitions to _connect token expired_.

This length of time is determined by subtracting the connect token create timestamp from its expiry timestamp.

### Connected

While _connected_ the client buffers _connection payload packets_ received from the server so their payloads may be received by the client application. If no _connection payload packet_ or _connection keep-alive packet_ are received from the server within the client timeout period specified in the connect token, the client transitions to _connection timed out_. 

While _connected_ the client application may send _connection payload packets_ to the server. In the absence of _connection payload packet_ sent by the application, the client generates and sends _connection keep-alive packets_ to the server at some rate, like 10HZ.

While _connected_ if the client receives a _connection disconnect_ packet from the server, it transitions to _disconnected_.

If the client wishes to disconnect from the server, it sends a number of redundant _connection disconnect packets_ before transitioning to _disconnected_.

## Server-Side Connection Process

### Server-Side Overview

The dedicated server must be on a publicly accessible IP address and port.

The server manages a set of n client slots, where each slot from [0,n-1] represents room for one connected client. 

The maximum number of client slots per-server is implementation specific. Typical uses cases are expected in the range of [2,64] but the reference implementation supports up to 256 clients per-server. 

You may support more clients per-server if your implementation is able to handle them efficiently.

### Processing Connection Requests

The server follows these strict rules when processing connection requests:

1. Clients must have a valid connect token to connect.
2. Respond to a client only when absolutely necessary. 
3. Ignore any malformed request as soon as possible, with the minimum amount of work.
4. Make sure any response packet is smaller than the request packet to avoid DDoS amplification.

When a server receives a connection request packet from a client it contains the following data:

    0 (uint8) // prefix byte of zero
    [version info] (13 bytes)       // "NETCODE 1.00" ASCII with null terminator.
    [protocol id] (8 bytes)
    [connect token expire timestamp] (8 bytes)
    [connect token sequence number] (8 bytes)
    [encrypted private connect token data] (1024 bytes)

This packet is not encrypted, however:

* Only the dedicated server instance and the web backend can read the encrypted private connect token data, because it is encrypted with a private key shared between them.

* The important aspects of the packet such as the version info, protocol id and connect token expire timestamp, are protected by the AEAD construct, and thus cannot be modified without failing the signature check.

The server takes the following steps, in this exact order, when processing a _connection request packet_:

* If the packet is not the expected size of 1062 bytes, ignore the packet.

* If the version info in the packet doesn't match "NETCODE 1.00" (13 bytes, with null terminator), ignore the packet.

* If the protocol id in the packet doesn't match the expected protocol id of the dedicated server, ignore the packet.

* If the connect token expire timestamp is <= the current timestamp, ignore the packet.

* If the encrypted private connect token data doesn't decrypt with the private key, using the associated data constructed from: version info, protocol id and expire timestamp, ignore the packet.

* If the decrypted private connect token fails to be read for any reason, for example, having a number of server addresses outside of the expected range of [1,32], or having an address type value outside of range [0,1], ignore the packet.

* If the dedicated server public address is not in the list of server addresses in the private connect token, ignore the packet.

* If a client from the packet source address is already connected, ignore the packet.

* If a client with the client id contained in the private connect token data is already connected, ignore the packet.

* If the connect token has already been used by a different packet source IP address, ignore the packet. 

* Otherwise, add the private connect token hmac + packet source IP address to the history of connect tokens already used.

* If no client slots are available, then the server is full. Respond with a _connection denied packet_.

* Add an encryption mapping for the packet source IP address so that packets read from that address are decrypted with the client to server key in the private connect token, and packets sent to that address are encrypted with the server to client key in the private connect token. This encryption mapping expires in _timeout_ seconds of no packets being sent to or received from that address.

* If for some reason this encryption mapping cannot be added, ignore the packet.

* Otherwise, respond with a _connection challenge packet_ and increment the _connection challenge sequence number_.

## Processing Connection Challenge Packets

The server takes these steps, in this exact order, when processing a _connection response packet_:

* ...