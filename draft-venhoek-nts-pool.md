---
title: "NTS extensions for enabling pools"
abbrev: "NTS pools"
category: std

docname: draft-venhoek-nts-pool-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: INT
workgroup: ntp
keyword:
 - NTP
 - NTS
venue:
#  group: WG
#  type: Working Group
#  mail: WG@example.com
#  arch: https://example.com/WG
  github: "pendulum-project/nts-pool-draft"
  latest: "https://pendulum-project.github.io/nts-pool-draft/draft-nts-pool.html"

author:
 -
    fullname: "David Venhoek"
    organization: Tweede golf B.V.
    email: "david@tweedegolf.com"
 -
    fullname: "Folkert de Vries"
    organization: Tweede golf B.V.
    email: "folkert@tweedegolf.com"
 -
    fullname: "Marc Schoolderman"
    organization: Tweede golf B.V.
    email: "marc@tweedegolf.com"

normative:
  RFC8446:
  RFC8915:

informative:
  RFC5905:
  Pool:
    target: https://www.ntppool.org
    title: NTP Pool website

--- abstract

The aim of this document is to describe a proof of concept system for NTS pools that are able to be used by clients without any knowledge beyond plain NTS. The work here focuses purely on creating an intermediate NTS Key Exchange server that can be configured with the addresses of multiple servers and distribute load between them. The parts of pool operation dealing with managing the list of servers are left out of scope for this work.

--- middle

# Introduction

NTS {{RFC8915}} provides authenticity and limited confidentiality for NTP {{RFC5905}}. However, the key exchange preceding the actual time exchange makes it hard to implement a pool for NTS supporting servers in a manner similar to the DNS resolution approach taken to provide the NTP Pool {{Pool}}.

This document aims to provide extensions to the NTS Key Exchange sessions that allow for an implementation of a pool for NTS that:

  - is usable without changes to the client,
  - avoids constraining the time source's cookie format,
  - avoids time sources having potential access to all traffic.

# Conventions and Definitions

Throughout the text, the terms client and server will refer to those roles in an NTS Key Exchange session as specified in {{RFC8915}}. Please note that this means that the pool itself operates in both roles: As a server towards users of the pool, and as a client towards the time sources.

Where further specificity of the role of a participant is needed, we will use the term user to indicate a user of a pool, the term pool to indicate the pool itself, and time source for the time servers that the pool delegates the actual providing of time to.

{::boilerplate bcp14-tagged}

# General pool architecture

We propose a pool model where the pool provides an NTS Key Exchange service to the outside world. A major advantage of this model is that it avoids having to distribute certificates to all time sources. Contrary to {{RFC8915}}, there is no direct TLS connection between the client and the selected time source.

In {{RFC8915}}, cookies are generated based on key material that is extracted from this TLS connection. Our proposed model instead establishes two TLS connections: between the client and the pool, and between the pool and the time source. Because cookies need to be generated using key material from the client, the pool extracts this key material and sends it to the server. The server uses this key material (rather than key material extracted from its connection with the pool) to generate cookies. This way, the pool can remain oblivious to the cookie format of the time source.

# Communication between the pool and time sources

To facilitate communication between the pool and the time sources, 4 new NTS records are defined in {{records}}. Together these records provide a way for the pool to provide key exchange services to clients on behalf of the time sources.

The Supported Next Protocol List ({{supportedprotocol}}) and Supported Algorithm List ({{supportedalgorithm}}) records allow the pool to ask a time source which protocols and algorithms it supports. This information can be requested by the pool at any time, and can be cached for short periods of time to improve efficiency.

Using knowledge of a time source's supported protocols and algorithms, the pool can then handle client connections for that time source, using the clients indicated desires to choose a concrete next protocol and AEAD algorithm. The pool can then extract the keys from the TLS connection and use the Fixed Key record ({{fixedkey}}) to request cookies for these keys from the time source.

As it is wasteful to setup a new TLS session between the pool and the time source for each of these interactions. To facilitate reuse of the TLS sessions, we further introduce the Keep Alive record ({{keepalive}}). This record allows the pool to indicate to the time source a desire to keep the session alive for more than a single request-response interaction.

## Authenticating the pool to time sources {#poolauth}

Allowing arbitrary clients to keep connections alive for more that a single request-response interaction could open up the server to denial of service due to resource exhaustion. To prevent this, a pool wishing to use the keep alive functionality MUST authenticate itself to the time source using a TLS Client Authentication as defined in {{RFC8446}}. Time sources MUST check that this authentication was successful, and that the requestor is on the list of requestors allowed to use the keep alive mechanism. By default, the list of requestors allowed to use the keep alive mechanism MUST be empty

Furthermore, time sources MAY choose to also restrict the Fixed Key, Supported Next Protocol List and Supported Algorithm List to authenticated clients. If this choice is made, it is suggested that the server treat these records as unrecognized critical records on unauthenticated client's connections.

# Communication between clients and the pool

A client requesting time from the pool can make a normal NTS Key Exchange request to the pool. In the response to the client the pool needs to tell which NTP server is to be used to get the time. This can be done through the already existing NTP Server Record. However, the pool needs to ensure it is present, and therefore MUST add such a record to the response unless one is already provided by the time source.

Clients that are aware they are talking to a pool may want to get multiple independent time sources from that pool. For this, they need to be able to tell the pool which time sources they already have, otherwise they might get a time source that they are already talking to. To achieve this, a client can use the NTP Server Deny record ({{serverdeny}}) to indicate it would rather not receive a particular server. Clients MUST use the precise name given by the pool in a previous NTP Server record, otherwise the pool may not recognize which time source the client is referring to.

# New NTS record types {#records}

## Keep Alive {#keepalive}
Record Type Number: To be assigned by IANA (draft implementations: 0x4000)
Critical bit: 0

Indicates a desire to keep the TLS connection active for more than one message exchange. This can be used by a pool to reuse connections to a time source's NTS Key Exchange servers multiple times, reducing load on both the pool and time sources.

A client MUST send this record with a body of size 0. A client MUST NOT use Keep Alive unless the request contains a record type allowing the use of Keep Alive. Within this specification, that is limited to the Supported Protocol List and Fixed Key Request records. A server SHOULD ignore any body for the Keep Alive record.

When supported by a server and allowed for the request in question, the server MAY include a Keep Alive record with a body of size 0 in the response and keep the TLS connection active after the response to handle further requests from the client. A client SHOULD ignore any body for the Keep Alive record. As keeping a connection active requires additional resources on the server, a server SHOULD NOT respond with a Keep Alive record to unauthenticated clients.

When included in the request or response, the client respectively server MAY, contrary to the requirements in {{RFC8915}}, send another request or response. Any TLS "close_notify" SHALL be sent only after the last request or response respectively to use the connection.

Once a Keep Alive record has been sent by a client, or honored by a server, the TLS connection over which it was sent MUST NOT be used for key extraction. Doing so anyway can result in the reuse of keys and may result in loss of confidentiality or authenticity of the resulting NTP exchanges.

## Supported Next Protocol List {#supportedprotocol}
Record Type Number: To be assigned by IANA (draft implementations: 0x4004)
Critical bit: 1

This record can be used by a pool to query time sources about which next protocols they support.

A client MUST send this record with no body. Clients MAY use Keep Alive in combination with this record. Contrary to {{RFC8915}}, a request with this record SHOULD NOT include a "Next Protocol Negotiation", "AEAD Algorithm Negotiation" or "Fixed Key Request" record.

Servers MUST ignore any client body sent and MUST send in the response a Supported Next Protocol List record with as data a list of 16-bit integers, giving the protocol IDs the server supports. A server MAY treat this record as unknown for clients that are not authenticated as described in {{poolauth}}.

When included, the server MUST NOT negotiate a next protocol, AEAD algorithm, or keys for this request.

## Supported Algorithm List {#supportedalgorithm}
Record Type Number: To be assigned by IANA (draft implementations: 0x4001)
Critical bit: 1

This record can be used by a pool to query time sources about which AEAD algorithms they support.

A client MUST send this record with no body. Clients MAY use Keep Alive in combination with this record. Contrary to {{RFC8915}}, a request with this record SHOULD NOT include a "Next Protocol Negotiation", "AEAD Algorithm Negotiation" or "Fixed Key Request" record.

Servers MUST ignore any client body sent and MUST send in the response a Supported Algorithm List record with as data a list of tuples of two 16-bit integers, the first giving an algorithm ID for the AEAD and the second giving the length of the key for that algorithm ID. A server MAY treat this record as unknown for clients that are not authenticated as described in {{poolauth}}.

When included, the server MUST NOT negotiate a next protocol, AEAD algorithm, or keys for this request.

We include the algorithm key size in the response so that a pool does not itself need knowledge of which AEAD algorithms exist, and what their key sizes are. Instead, it can use the provided key length when extracting keys from the TLS connection between end user and pool. This allows adoption of new AEAD algorithms without any changes to the pool software.

## Fixed Key Request {#fixedkey}
Record Type Number: To be assigned by IANA (draft implementations: 0x4002)
Critical Bit: 1

When a client is properly authenticated, the server SHOULD NOT perform Key Extraction but rather use the keys provided by the client in the extension field. This allows a pool to do key negotiation on behalf of its users with the time source's NTS Key Exchange servers, even though it terminates the TLS connection.

When used, the client MUST provide an AEAD Algorithm Negotiation record with precisely one algorithm, and a Next Protocol Negotiation record with precisely one next protocol. The data in the Fixed Key Request record must have length twice the key length N of the AEAD algorithm in the AEAD Algorithm Negotiation record. The first N bytes MUST be the C2S Key and the second set of N bytes MUST be the S2C key. Clients MAY use Keep Alive in combination with this record.

This record MUST NOT be sent by a server. A server MAY treat this record as unknown for clients that are not authenticated as described in {{poolauth}}.

## NTP Server Deny {#serverdeny}
Record Type Number: To be assigned by IANA (draft implementations: 0x4003)
Critical Bit: 0

When provided by a client, indicates a desire to connect to a server other than the server specified in the record. This can be used to ensure a client receives independent NTP servers from one NTS Key Exchange server without having to potentially try multiple times to get a new server.

A client MAY send multiple of these records if desired. The data in the record SHOULD match that given through an NTPv4 Server Negotiation received in an earlier response from the same NTS Key Exchange server.

MUST NOT be sent by a server. Server MAY at its discretion ignore the request from the client and still provide the given server in an NTPv4 Server Negotiation record.

# Security Considerations

## Pool's position

In the pool design presented above, the pool effectively acts as a man in the middle between the user and the ultimate time source during the NTS Key Exchange portion of the session. This means that the pool has access to the key material of these sessions. Although this is a small additional risk, we consider this acceptable because the pool could already always assign sessions for a user to time servers it controls anyway.

The fact that the pool also gets access to key material makes it less advisable to have a pool as a time source for another pool, as this increases the number of actors with access to the key material even further.

The design above does avoid sharing key material between all time sources. As a consequence, a time source in the pool will not be able to break confidentiality or authenticity of traffic with other time sources of the pool. Furthermore, any traffic directly with the time source has no key material involved that is known to the pool.

## Keep alive and denial of service attack risk

The Keep Alive NTS record allows a client to keep an NTS key exchange connection open for significantly longer than usual. If arbitrary clients were allowed to do this, they could use it trivially run a server out of resources such as file descriptors. It is therefore important that public servers restrict keeping connections alive to a limited set of trusted clients. The suggested mechanism for doing this is to use TLS client authentication for these clients.

## Error handling

To avoid giving multiple time sources access to the key material of the end user, it is important that the keys extracted from the TLS session between the user and the pool are sent to at most one time source. If an error occurs after sending the Fixed Key Request record, either with the TLS connection between the pool and the time source, or by being explicitly reported by the time source to the pool, the pool SHOULD return an error to the user. Retrying with a different time source during the same TLS session may unintentionally leave the user vulnerable to the operator of the originally selected time source.

# IANA Considerations

IANA is requested to allocate the following entries in the Network Time Security Key Establishment Record Types registry {{RFC8915}}:

| Record Type Number | Description | Reference |
| --- | --- | --- |
| [[TBD]] | Keep Alive | [[this memo]] {{keepalive}} |
| [[TBD]] | Supported Next Protocol List | [[this memo]] {{keepalive}} |
| [[TBD]] | Supported Algorithm List | [[this memo]] {{supportedalgorithm}} |
| [[TBD]] | Fixed Key Request | [[this memo]] {{fixedkey}} |
| [[TBD]] | NTP Server Deny | [[this memo]] {{serverdeny}} |


--- back

# Acknowledgments
{:numbered="false"}

The authors thank Marlon Peeters for their input and discussions during the writing of this document.
