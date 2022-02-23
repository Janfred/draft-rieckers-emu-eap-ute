---
stand_alone: true
ipr: trust200902
cat: std
submissiontype: IETF
area: SEC
wg: Internet Engineering Task Force

docname: draft-rieckers-emu-eap-ute-latest

title: "User-assisted Trust Establishment (EAP-UTE)"
abbrev: "EAP-UTE"
lang: en
smart_quotes: no
pi: [toc, sortrefs, symrefs]
author:
  - name: Jan-Frederik Rieckers
    org: Deutsches Forschungsnetz | German National Research and Education Network
    ins: J.-F. Rieckers
    street: Alexanderplatz 1
    code: 10178
    city: Berlin
    country: Germany
    email: rieckers@dfn.de
    abbrev: DFN
    uri: www.dfn.de

normative:
  RFC2104: hmac
  RFC3748: eap
  RFC7542: nai
  RFC8949: cbor

informative:
  RFC9140: eapnoob
  I-D.draft-rieckers-emu-eap-noob-observations:
    title: Observations about EAP-NOOB (RFC 9140)
    author:
      ins: J.-F. Rieckers
      name: Jan-Frederik Rieckers
      org: DFN, Germany
    date: 2022
    seriesinfo:
      Internet-Draft: draft-rieckers-emu-eap-noob-observations-00
    format:
      TXT: https://www.ietf.org/archive/id/draft-rieckers-emu-eap-noob-observations-00.txt


--- abstract

The Extensible Authentication Protocol (EAP) provides support for multiple authentication methdos.
This document defines the EAP-UTE authentication method for a User-assisted Trust Establishment between the peer and the server.
The EAP method is intended for bootstraping Internet-of-Things (IoT) devices without preconfigured authentication credentials.
The trust establishment is achived by transmitting a one-directional out-of-band message between the peer and the server to authenticate the in-band exchange.
The peer must have a secondary input or output interface, such as a display, camera, microphone, speaker, blinking light, or lightsensor, so dynamically generated messages with tens of bytes in length can be transmitted.

--- middle

# Introduction

This document describes a method for registration, authentication, and key derivation for network-connected devices, especially with low computational power and small or no interaction interfaces, such as devices that are part of the Internet of Things (IoT).
These devices may come without preconfigured trust anchors or have no possibility to receive a network configuration that enables them to connect securely to a network.

This document uses the basic design principle behind the EAP-NOOB method described in {{RFC9140}} and aims to improve some key elements of the protocol to better address the needs for IoT devices.
This is mainly achieved by using CBOR with numeric keys instead of JSON to encode the message.

TODO: Also included is extensibility.

The possible problems of EAP-NOOB are discussed in {{I-D.draft-rieckers-emu-eap-noob-observations}}. This document provides a specification which aims to address these concerns.


# Conventions and Definitions

{::boilerplate bcp14-tagged}

TODO frequently used terms

authenticator

peer

server

# EAP-UTE protocol

This section defines the EAP-UTE method.

## Protocol Overview

TODO: The introduction text is basically copied from RFC9140. should be reworded.

The EAP-UTE method execution spans two or more EAP conversations, called Exchanges in this specification.
Each Exchange consists of several EAP request-response pairs.
In order to give the user time to deliver the Out-of-Band message between the peer and the server, at least two separate EAP conversations are needed.

The overall protocol starts with a version and cryptosuite negotiation and peer detection.
Depending of the current state of the peer and server, different exchanges are selected.

If the server or the peer are in the unregistered state, peer and server exchange nonces and keys for the Ephemeral Elliptic Curve Diffie-Hellman.
This is called the Initial Exchange.
The Initial Exchange results in a EAP-Failure, since neither the server nor the peer are authenticated.

After the Initial Exchange, the user-assisted step of trust establishment takes place.
The user delivers one Out-of-Band message either from the peer to the server or from the server to the peer.

While peer and server are waiting for completion of the OOB Step, the peer MAY probe the server by reconnecting, to check for successful transmission of the OOB message.
This probe request will result in a waiting exchange and EAP-Failure, if the server has not yet received the OOB message.

If either the server or the peer have received the OOB message, the probe request will result in a completion exchange.
In the Completion Exchange, peer and server exchange message authentication codes over the previous in-band messages and the OOB message.
The completion exchange may result in EAP-Success.
Once the peer and server have performed a successful completion exchange, both endpoints store the created association in persistent storage.

After a successful Completion Exchange, the peer and server can use the Reconnect Exchange, to create a new association with new cryptographic bindings.
The user-assisted OOB step is not necessary, since the peer and server can infer the mutual authentication by using the persistent data stored in the Completion Exchange.

                                          Waiting
                                          .------.
                                          |      V
       +------------------+         +---------------------+
    .->| Unregistered (0) | Initial | Waiting for OOB (1) |
    |  |   (ephemeral)    |-------->|    (ephemeral)      |
    |  +------------------+         +---------------------+
    |                                 |    |
    |User Reset     .-----------------'    | OOB Input
    |               | Completion           |
    |               |                      |
    |               V                      V
    | +----------------+            +------------------+
    '-| Registered (3) | Completion | OOB Received (2) |
      |  (persistent)  |<-----------|                  |
      +----------------+            +------------------+
{: #statemachine title="EAP-UTE Server-Peer Association State Machine"}

## Messages

### General Message format

All messages are encoded in CBOR {{RFC8949}} as maps. A detailed format description will follow with a first implementation.

TODO: Here comes a list of message fields with their type

#### Thoughts about the message format

EAP-NOOB {{RFC9140}} uses JSON as encoding. Problems of using JSON are discussed in section 2.1 of {{I-D.draft-rieckers-emu-eap-noob-observations}}.

The possible other formats include:

* Static encoding  
  This allows a minimal number of bytes and requires minimal amount of parsing, since format and order of the message fields is exactly specified.
  However, this encoding severely effects the extensibility, unless a specific extension format is used.
  Additionally, a mechanism for optional fields is required.
* CBOR with static fields  
  This approach has a slightly higher number of bytes then the static encoding, but allows for an easier extensibility.
  The required fields can be specified, so the order of the protocol field is static and a parser has minimal effort to parse the protocol fields.
  However, this might be problematic in future protocol versions, when new fields are introduced.
  Like with static encoding, this also requires a mechanism for optional fields in the different message types.
* CBOR map with numeric keys  
  To mitigate the problems of optional fields, while keeping the parsing effort low, CBOR maps with numeric keys can be used.
  All protocol fields are identified by a unique ID, specified in this document.
  A parser can simply loop through the CBOR map. Since CBOR maps have a canonical order, minimal implementations may rely on this fact to parse the information needed.

On the basis of this discussion, this version of the specification will use a CBOR map as message encoding.
However, this is just a suggestion and suggestions for other message formats are highly welcome.

### Server greeting

* Message Type: 1
* Required Attributes:
  * Versions
  * Ciphers
  * ServerInfo
  * Directions
* Optional Attributes:

### Client greeting
* Message Type: 2
* Required Attributes:
  * Version
  * Cipher
  * PeerInfo
  * Direction
  * Nonce_P
  * Key_P
* Optional Attributes:
  * PeerId

### Server Keyshare
* Message Type: 3
* Required Attributes:
  * Key_S
  * Nonce_S
  * MAC_S
* Optional Attributes:
  * PeerId
  * AdditionalServerInfo

### Client Finished
* Message Type: 4
* Reqired Attributes:
  * MAC_P

### Client Completion Request
* Message Type: 5
* Required Attributes:
  * Nonce_P
  * PeerId
* Optional Attributes:
  * OOB-Id

### Server Completion Response
* Message Type: 6
* Required Attributes:
  * Nonce_S
  * MAC_S
* Optional Attributes:
  * OOB-Id

### Client Keyshare
* Message Type: 7
* Required Attributes:
  * PeerId
  * Nonce_P
  * Key_P

## Initial Exchange

    EAP Peer            Authenticator   EAP Server
      |                         |             |
      |<- EAP-Request/Identity -|             |
      |                                       |
      |-- EAP-Response/Identity ------------->|
      |    (NAI=new@eap-ute.arpa)             |
      |                                       |
      |<- EAP-Request/EAP-UTE ----------------|
      |   SERVER GREETING (1)                 |
      |   Versions, Ciphers, ServerInfo,      |
      |   Directions                          |
      |                                       |
      |-- EAP-Response/EAP-UTE -------------->|
      |   CLIENT GREETING (2)                 |
      |   Version, Cipher, PeerInfo,          |
      |   Direction, Nonce_P, Key_P           |
      |                                       |
      |<- EAP-Request/EAP-UTE ----------------|
      |   SERVER KEYSHARE (3)                 |
      |   PeerId, Key_S, Nonce_S, MAC_S       |
      |                                       |
      |-- EAP-Response/EAP-UTE -------------->|
      |   CLIENT FINISHED (4)                 |
      |   MAC_P                               |
      |                                       |
      |<- EAP-Failure ------------------------|
      |                                       |
{: #initialexchange title="Initial Exchange"}

TODO: Do I need MACs here? What are they really for?

## Waiting Exchange

    EAP Peer            Authenticator   EAP Server
      |                         |             |
      |<- EAP-Request/Identity -|             |
      |                                       |
      |-- EAP-Response/Identity ------------->|
      |    (NAI=waiting@eap-ute.arpa)         |
      |                                       |
      |<- EAP-Request/EAP-UTE ----------------|
      |   SERVER GREETING (1)                 |
      |   Versions, Ciphers, Server Info,     |
      |   Directions                          |
      |                                       |
      |-- EAP-Response/EAP-UTE -------------->|
      |   CLIENT COMPLETION REQUEST (5)       |
      |   PeerId, Nonce_P                     |
      |                                       |
      |<- EAP-Failure ------------------------|
      |                                       |
{: #waitingexchange title="Waiting Exchange"}

## Completion Exchange

    EAP Peer            Authenticator   EAP Server
      |                         |             |
      |<- EAP-Request/Identity -|             |
      |                                       |
      |-- EAP-Response/Identity ------------->|
      |    (NAI=waiting@eap-ute.arpa)         |
      |                                       |
      |<- EAP-Request/EAP-UTE ----------------|
      |   SERVER GREETING (1)                 |
      |   Versions, Ciphers, Server Info,     |
      |   Directions                          |
      |                                       |
      |-- EAP-Response/EAP-UTE -------------->|
      |   CLIENT COMPLETION REQUEST (5)       |
      |   PeerId, Nonce_P, [OOB-Id]           |
      |                                       |
      |<- EAP-Request/EAP-UTE ----------------|
      |   SERVER COMPLETION RESPONSE (6)      |
      |   [OOB-Id], Nonce_S, MAC_S            |
      |                                       |
      |-- EAP-Response/EAP-UTE -------------->|
      |   CLIENT FINISHED (4)                 |
      |   MAC_P                               |
      |                                       |
      |<- EAP-Success ------------------------|
      |                                       |
{: #completionexchange title="Completion Exchange"}

## Reconnect Exchange

    EAP Peer            Authenticator   EAP Server
      |                         |             |
      |<- EAP-Request/Identity -|             |
      |                                       |
      |-- EAP-Response/Identity ------------->|
      |                                       |
      |<- EAP-Request/EAP-UTE ----------------|
      |   SERVER GREETING (1)                 |
      |   Versions, Ciphers, Server Info,     |
      |   Directions                          |
      |                                       |
      |-- EAP-Response/EAP-UTE -------------->|
      |   CLIENT COMPLETION REQUEST (5)       |
      |   PeerId, Nonce_P                     |
      |                                       |
      |<- EAP-Request/EAP-UTE ----------------|
      |   SERVER COMPLETION RESPONSE (6)      |
      |   Nonce_S, MAC_S                      |
      |                                       |
      |-- EAP-Response/EAP-UTE -------------->|
      |   CLIENT FINISHED (4)                 |
      |   MAC_P                               |
      |                                       |
      |<- EAP-Success ------------------------|
      |                                       |
{: #reconnectstatic title="Reconnect Exchange without new ECDHE exchange"}

    EAP Peer            Authenticator   EAP Server
      |                         |             |
      |<- EAP-Request/Identity -|             |
      |                                       |
      |-- EAP-Response/Identity ------------->|
      |                                       |
      |<- EAP-Request/EAP-UTE ----------------|
      |   SERVER GREETING (1)                 |
      |   Versions, Ciphers, Server Info,     |
      |   Directions                          |
      |                                       |
      |-- EAP-Response/EAP-UTE -------------->|
      |   CLIENT KEYSHARE (7)                 |
      |   PeerId, Nonce_P, Key_P              |
      |                                       |
      |<- EAP-Request/EAP-UTE ----------------|
      |   SERVER KEYSHARE (3)                 |
      |   Nonce_S, Key_S, MAC_S               |
      |                                       |
      |-- EAP-Response/EAP-UTE -------------->|
      |   CLIENT FINISHED (4)                 |
      |   MAC_P                               |
      |                                       |
      |<- EAP-Success ------------------------|
      |                                       |
{: #reconnectecdhe title="Reconnect Exchange with new ECDHE exchange"}


# Security Considerations

TODO Security

## EAP Security Claims

TODO. See {{RFC3748}}, section 7.2.1


# IANA Considerations

This document has IANA actions.
What they are exactly needs to be defined in detail.

The EAP Method Type number for EAP-UTE needs to be assigned.
The reference implementation will use 255 (Experimental) for now.

Like EAP-NOOB, this draft will probably use a .arpa domain, in this case probably eap-ute.arpa, as default NAI realm.

# Implementation Status

Note to RFC Editor: Please remove this entire section before publication.

There are no implementations yet.

--- back

# Acknowledgements {#Acknowledgements}
{:unnumbered}

TBD
