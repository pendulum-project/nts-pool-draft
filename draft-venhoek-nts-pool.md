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
  RFC8915:

informative:
  RFC5905:
  Pool:
    target: https://www.ntppool.org
    title: NTP Pool website

--- abstract

The aim of this document is to describe a proof of concept system for NTS pools that are able to be used by clients without any knowledge beyond plain NTS. The work here focuses purely on creating an intermediate NTS Key Exchange server that can be configured with the addresses of multiple downstream servers and distribute load between them. The parts of pool operation dealing with managing the list of servers are left out of scope for this work.

--- middle

# Introduction

NTS {{RFC8915}} provides authenticity and limited confidentiality for NTP {{RFC5905}}. However, the key exchange preceding the actual time exchange makes it hard to implement a pool for NTS supporting servers in a manner similar to the DNS resolution approach taken to provide the NTP Pool {{Pool}}.

This document aims to provide extensions to the NTS Key Exchange sessions that allow for an implementation of a pool for NTS that:

  - is usable without changes to the client,
  - avoids constraining the downstream time source's cookie format,
  - avoids downstream time sources having potential access to all traffic.

# Conventions and Definitions

Throughout the text, the terms client and server will refer to those roles in an NTS Key Exchange session as specified in {{RFC8915}}. Please note that this means that the pool itself operates in both roles: As a server towards users of the pool, and as a client towards the downstream time sources.

Where further specificity of the role of a participant is needed, we will use the term user to indicate a user of a pool, the term pool to indicate the pool itself, and downstream time source for the time servers that the pool delegates the actual providing of time to.

{::boilerplate bcp14-tagged}

# General pool architecture

We propose a pool model where the pool provides an NTS Key Exchange service to the outside world. A major advantage of this model is that it avoids having to distribute certificates to all downstream time servers. Contrary to {{RFC8915}}, there is no direct TLS connection between the client and the selected downstream time service.

In {{RFC8915}}, cookies are generated based on key material that is extracted from this TLS connection. Our proposed model instead establishes two TLS connections: between the client and the pool, and between the pool and the downstream time server. Because cookies need to be generated using key material from the client, the pool extracts this key material and sends it to the server. The server uses this key material (rather than key material extracted from its connection with the pool) to generate cookies. This way, the pool can remain oblivious to the cookie format of the downstream time server.

# Client facilities for pools

One challenge with getting multiple time sources from a single NTS Key Exchange server is that clients that allow for explicit pool configuration want to end up with multiple independent time sources. Without additional support, a user of a pool might receive a downstream time source it already has from an NTS Key Exchange session, resulting in that session being a waste of time. To avoid unneccessary NTS Key Exchange sessions, we also introduce a record that clients can use to indicate which downstream time servers they don't want, because they already have them.

# Pool authentication

The extensions proposed below allow a client to establish an NTS association with a server with arbitrary keys, not just those extracted from the TLS session. To discourage misuse, it is not desirable to allow arbitrary clients to do this.

Therefore, a server supporting the Fixed Key Request record from {{fixedkey}} MUST authenticate clients using the Fixed Key Request record using TLS client certificates. Support MUST be disabled by default, and when enabled, MUST be limited to an explicitly configured list of clients.

# New NTS record types

## Keep Alive {#keepalive}
Record Type Number: To be assigned by IANA (draft implementations: 0x4000)
Critical bit: 0

Indicates a desire to keep the TLS connection active for more than one message exchange. This can be used by a pool to reuse connections to downstream NTS Key Exchange servers multiple times, reducing load on both the pool and downstream servers.

Client MUST send this record with a body of size 0. Client MUST NOT use Keep Alive unless the request contains a record type allowing the use of Keep Alive. Within this specification, that is limited to the Supported Protocol List and Fixed Key Request records. A server SHOULD ignore any body for the Keep Alive record.

When supported by a server and allowed for the request in question, the server MUST include a Keep Alive record with a body of size 0 in the response and keep the TLS connection active after the response to handle further requests from the client. A client SHOULD ignore any body for the Keep Alive record.

When included in the request or response, the client respectively server MAY, contrary to the requirements in {{RFC8915}}, send another request or response. Any TLS "close_notify" SHALL be sent only after the last request or response respectively to use the connection.

Once a Keep Alive record has been sent by a client, or honored by a server, the TLS connection over which it was sent MUST NOT be used for key extraction. Doing so anyway can result in the reuse of keys and may result in loss of confidentiality or authenticity of the resulting NTP exchanges.

## Supported Next Protocol List {#supportedprotocol}
Record Type Number: To be assigned by IANA (draft implementations: 0x4004)
Critical bit: 1

This record can be used by a pool to query downstream servers about which next protocols they support.

Client MUST send with no body. Clients MAY use Keep Alive in combination with this record. Contrary to {{RFC8915}}, a request with this record SHOULD NOT include a "Next Protocol Negotiation", "AEAD Algorithm Negotiation" or "Fixed Key Request" record.

Server MUST ignore any client body sent and MUST send in response a Supported Next Protocol List record with as data a list of 16-bit integers, giving the protocol IDs the server supports.

When included, the server MUST NOT negotiate a next protocol, AEAD algorithm, or keys for this request.

## Supported Algorithm List {#supportedalgorithm}
Record Type Number: To be assigned by IANA (draft implementations: 0x4001)
Critical bit: 1

This record can be used by a pool to query downstream servers about which AEAD algorithms they support.

Client MUST send with no body. Clients MAY use Keep Alive in combination with this record. Contrary to {{RFC8915}}, a request with this record SHOULD NOT include a "Next Protocol Negotiation", "AEAD Algorithm Negotiation" or "Fixed Key Request" record.

Server MUST ignore any client body sent and MUST send in response a Supported Algorithm List record with as data a list of tuples of two 16-bit integers, the first giving an algorithm ID for the AEAD and the second giving the length of the key for that algorithm ID.

When included, the server MUST NOT negotiate a next protocol, AEAD algorithm, or keys for this request.

We include the algorithm key size in the response so that a pool does not itself need knowledge of which AEAD algorithms exist, and what their key sizes are. Instead, it can use the provided key length when extracting keys from the TLS connection between end user and pool. This allows adoption of new AEAD algorithms without any changes being needed for the pool software.

## Fixed Key Request {#fixedkey}
Record Type Number: To be assigned by IANA (draft implementations: 0x4002)
Critical Bit: 1

When a client is properly authenticated, the server SHOULD NOT perform Key Extraction but rather use the keys provided by the client in the extension field. This allows a pool to do key negotiation on behalf of its users with the downstream NTS Key Exchange servers, even though it terminates the TLS connection.

When used, the client MUST provide an AEAD Algorithm Negotiation record with precisely one algorithm, and a Next Protocol Negotiation record with precisely one next protocol. The data in the Fixed Key Request record must have length twice the key length N of the AEAD algorithm in the AEAD Algorithm Negotiation record. The first N bytes MUST be the C2S Key and the second set of N bytes MUST be the S2C key. Clients MAY use Keep Alive in combination with this record.

MUST NOT be sent by a server. Server SHOULD treat the extension field as unknown when sent by any client not authorized to make fixed key requests.

## NTP Server Deny {#serverdeny}
Record Type Number: To be assigned by IANA (draft implementations: 0x4003)
Critical Bit: 0

When provided by a client, indicates a desire to connect to a server other than the server specified in the record. This can be used to ensure a client receives independent NTP servers from one NTS Key Exchange server without having to potentially try multiple times to get a new server.

A client MAY send multiple of these records if desired. The data in the record SHOULD match that given through an NTPv4 Server Negotiation received in an earlier request from the same NTS Key Exchange server.

MUST NOT be sent by a server. Server MAY at its discretion ignore the request from the client and still provide the given server in an NTPv4 Server Negotiation record.

# Security Considerations

## Pool's position

In the pool design presented above, the pool effectively acts as a man in the middle between the user and the ultimate time source during the NTS Key Exchange portion of the session. This means that the pool has access to the key material of these sessions. Although this is a small additional risk, we consider this acceptable because the pool could already always assign sessions for a user to time servers it controls anyway.

The fact that the pool also gets access to key material makes it less advisable to have a pool as a downstream time source for another pool, as this increases the number of actors with access to the key material even further.

The design above does avoid sharing key material between all downstream time sources. As a consequence, a downstream time source in the pool will not be able to break confidentiality or authenticity of traffic with other downstream time sources of the pool. Furthermore, any traffic directly with the downstream time source has no key material involved that is known to the pool.

## Error handling

To avoid giving multiple downstream time sources access to the key material of the end user, it is important that the keys extracted from the TLS session between the user and the pool are sent to at most one downstream time source. If an error occurs after sending the Fixed Key Request record, either with the TLS connection between the pool and the downstream time source, or by being explicitly reported by the downstream time source to the pool, the pool SHOULD return an error to the user. Retrying with a different downstream time source during the same TLS session may unintentionally leave the user vulnerable to the operator of the originally selected downstream time source.

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

The authors thank Marlon Peters and Marc Schoolderman for their input and discussions during the writing of this document.
