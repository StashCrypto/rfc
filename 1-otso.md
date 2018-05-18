---
title: OT Sign-On Protocol v1
layout: page
---

Name: [stashcrypto.github.io/rfc/1-otso](https://stashcrypto.github.io/rfc/1-otso)  
Status: draft  
Editor: Justus Ranvier [justus@stashcrypto.com](mailto:justus@stashcrypto.com)

<br>

This document specifies version 1 OTSO, the OT Sign-On Protocol. The use case for OTSO is a web service which wishes to cryptographically prove a connected session is controlled by the owner of a BIP-47 payment code.

See also:
* [http://rfc.zeromq.org/spec:23/ZMTP](http://rfc.zeromq.org/spec:23/ZMTP)
* [http://rfc.zeromq.org/spec:25/ZMTP-CURVE](http://rfc.zeromq.org/spec:25/ZMTP-CURVE)
* [http://rfc.zeromq.org/spec:26/CURVEZMQ](http://rfc.zeromq.org/spec:26/CURVEZMQ)

<br>

<br>

## Preamble

<br>

_Copyright (c) 2018 Stash_

This Specification is free software; you can redistribute it and/or modify it under the terms of the GNU General Public License as published by the Free Software Foundation; either version 3 of the License, or (at your option) any later version. This Specification is distributed in the hope that it will be useful, but WITHOUT ANY WARRANTY; without even the implied warranty of MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See the GNU General Public License for more details. You should have received a copy of the GNU General Public License along with this program; if not, see http://www.gnu.org/licenses.

This Specification is a free and open standard and is governed by the Digital Standards Organization's Consensus-Oriented Specification System.

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119.

<br>

<br>

## Overall Design

<br>

### What Problems does OTSO Address?

OTSO allows blockchain-centric web services to dispense with password-based login by relying on the assumption that their users control a BIP-47 payment code that can serve as their user ID.

### Message Format

Messages used in the OTSO protocol are conveyed via ZeroMQ.

### Message Serialization

Messages used in the OTSO protocol consist of the Google Protobuf definitions provided in this specification.

<br>

<br>

## Formal Specification

<br>

### Architecture

OTSO defines a ZeroMQ message-based dialog between:
* A backend that makes authentication requests, and a backend handler that answers them. 
* A client that desires authenticated access to the backend and a client handler which validates client messages.

The backend handler and client handler are both part of the same component known as the OTSO handler.

The backend, client, and OTSO handler SHALL communicate using the following socket types and transports:

* The backend SHALL use a DEALER socket.
* The backend handler SHALL use a ROUTER socket.
* The backend handler SHALL bind to a user-defined endpoint.
* The backend handler endpoint SHOULD only be accessible to the backend and backend handler.
* The backend SHALL connect to this endpoint.
* Any number of backends MAY connect to a backend handler.
* The backend handler SHALL start before any backend starts.
* The client SHALL use a PUSH socket.
* The client SHALL act as a curve client.
* The client SHALL set the remote curve key to the transport key obtained from the challenge URI.
* The client handler SHALL use a PULL socket.
* The client handler SHALL act as a curve server.
* The client handler SHALL allow connections from any curve client.
* The client handler SHALL bind to a TCP endpoint on a user-defined port.
* The client handler endpoint MUST be accessible to all potential clients.
* The client SHALL connect to this endpoint.
* Any number of clients MAY connect to a client handler.
* The client handler SHALL start before any client starts.
* All participants in the protocol SHALL drop incorrect messages without reply. Messages are incorrect for the following reasons:
  * Invalid syntax
  * References an expired or non-existent previous message

<br>

<br>

## Backend - OTSO handler Protocol

<br>

This protocol specifies the interactions between the backend and the OSO handler.

When a client requests access to a backend resource that requires authentication:

1. The backend prepares an AuthRequest message.
   1. The version field SHALL be set to 1.
   1. The cookie field SHALL consists of an arbitrary value which enables the backend to correlate the request and subsequent notification message pertaining to it with the client session being authenticated.
   1. The size of the cookie field SHALL be no more than 64 bytes.
   1. The size of the cookie field SHOULD be at least 16 bytes.
1. The backend serializes the AuthRequest message and sends it to the backend handler.
1. The backend handler prepares a AuthReply message.
   1. The version field SHALL be set to 1.
   1. The cookie field SHALL be identical to the cookie field from the corresponding AuthRequest message.
   1. The challenge field SHALL consist of a challenge URI whose length does not exceed the limits of an alphanumeric QR code specified by ISO/IEC 18004:2015.
1. The backend handler serializes the AuthReply message and sends it to the backend.

When the OTSO backend completes its interactions with the client:
    
1. The backend handler prepares an AuthResult message
   1. The version field SHALL be set to 1.
   1. The cookie field SHALL be identical to the original AuthRequest message
   1. The paymentcode field SHALL contain the serialized BIP-47 payment code provided by the client, or an empty string
      1. The size paymentcode field SHALL be 80 bytes, or 0 bytes.
   1. The status field SHALL be set to AUTH_SUCCESS if the client successfully proved ownership of the notification address private key corresponding to the BIP-47 payment code in the paymentcode field.
   1. The status field SHALL be set to AUTH_FAILED if the client provides an invalid proof of ownership.
   1. The status field SHALL be set to AUTH_TIMEOUT if the client failed to respond with a user-specified timeout period.
      1. The paymentcode field SHALL be an empty string in the case of a timeout.
1. The backend handler serializes the AuthResult message and sends it to the backend.
1. Any messages from a client that reference an challenge URI corresponding to an AuthRequest for which an AuthResult message has already been created SHALL be ignored.

<br>

### Backend - Client Protocol

When a client requests access to a backend resource that requires authentication:
    
1. The backend SHALL obtain a challenge URI from the backend handler.
1. The backend SHALL present the challenge URI to the client as a QR code or other suitable representation.

When the backend receives an AuthResult message corresponding to the client's session:
    
1. The backend SHALL grant or deny access based on the status of the AuthResult.
1. The backend MAY create a new user account using the paymentcode from the AuthResult as a user ID if an account does not already exist.

<br>

### Client - OTSO handler protocol

When the client receives a challenge URI:
    
1. The client extracts the nonce from the URI and signs it using his notification address private key.
1. The client prepares an AuthResponse message.
   1. The version field SHALL be set to 1.
   1. The challenge field SHALL contain the challenge URI.
   1. The paymentcode field SHALL contain the client's BIP-47 payment code.
      1. The size of the paymentcode field SHALL be 80 bytes.
   1. The signature field SHALL contain the secp256k1 signature of the nonce.
      1. The size of the signature field SHALL be 64 bytes.
1. The client serializes the AuthResponse message and sends it to the client handler.

When the client handler receives an AuthResponse message:

1. The client handler extracts the notification address public key from the paymentcode.
1. The client handler verifies the signature of the nonce embeded in the challenge URI using the notification address public key.
1. The client handler SHALL send the appropriate AuthResult message to the backend.

If the time since the creation of an AuthReply message exceeds a user-defined timeout without receiving a corresponding AuthResponse:
    
1. The client handler SHALL send the appropriate AuthResult message to the backend.

<br>

### Challenge URI format

opentxs://otso/1/<endpoint>/<transport key>/<nonce>

1. endpoint SHALL consist of the client handler's PULL socket endpoint, with the "tcp://" prefix omitted.
1. transport key SHALL be a valid CurveCP public key in Z85 format.
1. nonce SHALL be 32 random bytes encoded in base64.
  
<br>

### Message Definitions

#### AuthRequest

syntax = "proto3";
```protobuf
message AuthRequest {
    uint32 version = 1;
    bytes cookie = 2;
}
```

<br>

#### AuthReply

syntax = "proto3";
```protobuf
message AuthReply {
    uint32 version = 1;
    bytes cookie = 2;
    string challenge = 3;
}
```

<br>

#### AuthResponse

syntax = "proto3";
```protobuf
message AuthResponse {
    uint32 version = 1;
    string challenge = 2;
    string paymentcode = 3;
    bytes signature = 4;
}
```

<br>

#### AuthResult

syntax = "proto3";
```protobuf
enum AuthStatus {
    STATUS_ERROR = 0;
    STATUS_SUCCESS = 1;
    STATUS_FAILED = 2;
    STATUS_TIMEOUT = 3;
}
```
```protobuf
message AuthResult {
    uint32 version = 1;
    string cookie = 2;
    string paymentcode = 3;
    AuthStatus status = 4;
}
```
