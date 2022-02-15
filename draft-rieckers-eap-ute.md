---
title: "User-assisted Trust Establishment (EAP-UTE)"
abbrev: "EAP-UTE"
category: std
ipr: trust200902
consensus: true
submissionType: IETF

docname: draft-rieckers-eap-ute-latest

stand_alone: yes
smart_quotes: no
pi: [toc, sortrefs, symrefs]

author:
 -
    name: Jan-Frederik Rieckers
    ins: J.-F. Rieckers
    organization: Verein zur Foerderung eines Deutschen Forschungsnetzes e.V.
    street: Alexanderplatz 1
    code: '10178'
    city: Berlin
    country: Germany
    email: rieckers@dfn.de
    abbrev: DFN-Verein

normative:
  RFC2104: hmac
  RFC3748: eap
  RFC7542: nai

informative:
  RFC9140: eapnoob


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

This document uses the basic design principle behind the EAP-NOOB method described in {{RFC9140}} but uses binary representation instead of JSON.


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

### Message format



## Initial Exchange

    EAP Peer            Authenticator   EAP Server
      |                         |             |
      |<- EAP-Request/Identity -|             |
      |                                       |
      |-- EAP-Response/Identity ------------->|
      |    (NAI=new@eap-ute.arpa)             |
      |                                       |
      |<- EAP-Request/EAP-UTE ----------------|
      |   Versions, Ciphers, ServerInfo,      |
      |   Directions                          |
      |                                       |
      |-- EAP-Response/EAP-UTE -------------->|
      |   Version, Cipher, PeerInfo,          |
      |   Direction, Nonce_P, Key_P           |
      |                                       |
      |<- EAP-Request/EAP-UTE ----------------|
      |   PeerId, Key_S, Nonce_S, MAC_S       |
      |                                       |
      |-- EAP-Response/EAP-UTE -------------->|
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
      |   Versions, Ciphers, Server Info,     |
      |   Directions                          |
      |                                       |
      |-- EAP-Response/EAP-UTE -------------->|
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
      |   Versions, Ciphers, Server Info,     |
      |   Directions                          |
      |                                       |
      |-- EAP-Response/EAP-UTE -------------->|
      |   PeerId, Nonce_P, [OOB-Id]           |
      |                                       |
      |<- EAP-Request/EAP-UTE ----------------|
      |   [OOB-Id], Nonce_S, MAC_S            |
      |                                       |
      |-- EAP-Response/EAP-UTE -------------->|
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
      |    (NAI=waiting@eap-ute.arpa)         |
      |                                       |
      |<- EAP-Request/EAP-UTE ----------------|
      |   Versions, Ciphers, Server Info,     |
      |   Directions                          |
      |                                       |
      |-- EAP-Response/EAP-UTE -------------->|
      |   PeerId, Nonce_P                     |
      |                                       |
      |<- EAP-Request/EAP-UTE ----------------|
      |   Nonce_S, MAC_S                      |
      |                                       |
      |-- EAP-Response/EAP-UTE -------------->|
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
      |    (NAI=waiting@eap-ute.arpa)         |
      |                                       |
      |<- EAP-Request/EAP-UTE ----------------|
      |   Versions, Ciphers, Server Info,     |
      |   Directions                          |
      |                                       |
      |-- EAP-Response/EAP-UTE -------------->|
      |   PeerId, Nonce_P, Key_P              |
      |                                       |
      |<- EAP-Request/EAP-UTE ----------------|
      |   Nonce_S, Key_S, MAC_S               |
      |                                       |
      |-- EAP-Response/EAP-UTE -------------->|
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

Like EAP-NOOB, this draft will probably use the eap-ute.arpa domain as default NAI realm.

# Implementation Status

Note to RFC Editor: Please remove this entire section before publication.

There are no implementations yet.

--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
