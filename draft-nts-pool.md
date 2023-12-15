---
title: "NTS extensions for enabling pools"
abbrev: "NTS pools"
category: info

docname: draft-nts-pool-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
# area: AREA
# workgroup: WG Working Group
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
    organization: Tweede Golf B.V.
    email: "david@tweedegolf.com"

normative:

informative:


--- abstract

The aim of this document is to describe a proof of concept system for NTS pools that are able to be used by clients without any knowledge beyond plain NTS. The work here focusses purely on creating an intermediate nts-ke server that can be configured with the addresses of multiple downstream servers and distribute load between them. The parts of pool operation dealing with managing the list of servers are left out of scope for this work.

--- middle

# Introduction

TODO Introduction


# Conventions and Definitions

{::boilerplate bcp14-tagged}

# New NTS records

NTS Record: Keep Alive
Critical bit: 0
Record Type Number: To be assigned by IANA (draft implementations: 0x4000)

Client MUST send with no body. Client MUST NOT use Keep Alive unless the request contains a record type allowing the use of Keep Alive. Within this specification, that is limited to the Supported Protocol List

When supported by server, Server should include a Keep Alive record in the response and keep the TLS connection active after the response to handle further requests from the client.

---

NTS Record: Supported Algorithm List
Critical bit: 1
Record Type Number: To be assigned by IANA (draft implementations: 0x4001)

Client MUST send with no body. Clients MAY use Keep Alive in combination with this record.

Server MUST ignore any client body sent, and MUST send in response a Supported Algorithm List record with as data a list of tuples of two 16 bit integers, the first giving a algorithm ID for the AEAD and the second giving the length of the key for that algorithm ID.

---

NTS Record: Fixed Key Request
Critical Bit: 1
Record Type Number: To be assigned by IANA (draft implementations: 0x4002)

When used, the client MUST provide an AEAD Algorithm Negotiation record with precisely one algorithm. The data in the record must have length twice the key length N of the AEAD algorithm in the AEAD Algorithm Negotiation record. The first N bytes MUST be the C2S Key and the second set of N bytes MUST be the S2C key. Clients MAY use Keep Alive in combination with this record.

MUST not be sent by a server. Server SHOULD treat extension field as unknown for any client not authenticated with a client side tls certificate configured through the servers policy as allowed for

When client is properly authenticated, the server SHOULD not perform Key Extraction for but rather use the keys provided by the client in the extension field.

---

NTS Record: NTP Server Deny
Critical Bit: 0
Record Type Number: To be assigned by IANA (draft implementations: 0x4003)

When provided by a client, indicates a desire to connect to a server other than the server specified in the record. A client MAY send multiple of these records as desired. The data in the record SHOULD match that given through an NTPv4 Server Negotiation received in an earlier request from the same NTS Key Exchange server.

MUST not be sent by a server. Server MAY at its discretion ignore the request from the client and still provide the given server in an NTPv4 Server Negotiation record.

# Security Considerations

TODO Security


# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
