---
# vim: spelllang=en
###
# Internet-Draft Markdown Template
#
# This template uses kramdown-rfc: https://github.com/cabo/kramdown-rfc
# You can replace the entire file if you prefer a different format.
# Change the file extension to match the format (.xml for XML, etc...)
#
###
title: "A recommendation for filtering address records in stub resolvers"
abbrev: "AAAA filtering considerations"
category: info

docname: draft-caletka-aaaa-filtering-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
#area: AREA
#workgroup: WG Working Group
keyword:
 - IPv6
 - DNS
 - Address records
venue:
#  group: WG
#  type: Working Group
#  mail: WG@example.com
#  arch: https://example.com/WG
  github: oskar456/ietf-aaaa-filtering
  latest: https://oskar456.github.io/ietf-aaaa-filtering/draft-caletka-aaaa-filtering.html

author:
 -
    fullname: Ondřej Caletka
    organization: RIPE NCC
    email: ondrej@caletka.cz

normative:
  RFC3493:
  RFC3927:
  RFC4291:
  RFC6724:
  RFC8305:

informative:
  RFC5684:
  IANA:
    target: https://www.iana.org/assignments/iana-ipv6-special-registry
    title: IANA IPv6 Special-Purpose Address Registry
...

--- abstract

Since IPv4 and IPv6 addresses are represented by different resource records in
the DNS, operating systems capable of running both IPv4 and IPv6 need to make
two queries when resolving a host name. This document discusses conditions, under
which the stub resolver can optimize the process by not sending one of the
queries if the host is connected to a single-stack network.


--- middle

# Introduction

Most operating systems support both IPv6 and IPv4 networking stack. When such a
host is connected to a dual-stack network, whenever a process requests
resolution of a DNS name, two DNS queries need to be issued - one for an A
record representing IPv4 address, one for a AAAA record representing IPv6
address. The results of such queries are then merged and ordered based on
[RFC6724] or used as input for the Happy Eyeballs algorithm [RFC8305].

When such a host is connected to a single-stack network, only one DNS query need
to be sent: there is no point of sending out AAAA record query if the host has
no IPv6 connectivity or sending out A query if the host has no IPv4
connectivity. Such an optimization however has to consider any possible mean of
obtaining connectivity for particular address family: including but not limited
to IPv6 Transition Mechanisms or VPNs.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Connectivity detection algorithm

Whenever an application asks the stub resolver to resolve a domain name without
specifying address family, stub resolver follows this algorithm for each address
family supported by the operating system:

 1. Read routing table of particular address family.
 2. Check for presence of a route towards a destination that is not a Link-Local
    range for such address family, ie. Section 2.5.6 of [RFC4291] for IPv6 and [RFC3927] for IPv4.
 3. If such a route is present, send the corresponding name query to the DNS:
    AAAA query for the address family of IPv6, A query for IPv4.

It is necessary to consider ANY route towards non Link-Local address space and
not just default route and/or default network interface. Such a detection would
cause issues with Split-mode VPNs providing only particular routes for the
resources reachable via VPN.

# Filtering DNS results

If the host does not have a full connectivity to both address families (there
are no default gateways for both IPv4 and IPv6), it is possible that the IP(v6)
address obtained from the DNS falls into the address space not covered by a
route. This should not be problem for a properly written applications, since
[RFC6724] requires applications to try connecting to all addresses received from
the stub resolver.

However, in order to minimize impact on poorly designed applications, the stub
resolver MAY remove addresses not covered by an entry in the routing table from
the list of DNS query results sent to the application.

## Filtering IPv4-mapped addresses

As an extension to the filtering of DNS results, the stub resolver MAY also
remove IPv4-mapped IPv6 addresses (Section 2.5.5.2 of [RFC4291]) from the list of DNS
query results sent to the application.

IPv4-mapped IPv6 addresses are not valid destination addresses [IANA],
therefore they should never appear in the AAAA records. Sending IPv4-mapped IPv6
address to the application might cause address family confusion for applications
using IPv4 compatibility of IPv6 sockets [RFC3493].

# Effects of not doing address record filtering

The optimization described above is OPTIONAL. A stub resolver of a dual-stack
capable host can always issue both A and AAAA queries to the DNS, merge and
order the results and send them to the application even if it has only a
single-stack connectivity. Sending packets to a destination not covered by an
entry in the routing table will be immediately refused, so a properly written
application will quickly fall through the list of addresses to the one using the
same address family as the connectivity of the host.

However, it should be noted that such behavior increases load on the DNS system.
If such an optimization is removed (for instance by a software update) on a
large single-stack networks, this might overload parts of the DNS
infrastructure, since the number of queries doubles.

# Security Considerations

Reducing the number of queries allows an attacker observing the DNS traffic to
figure out which address families the host uses.

Sudden disabling of the optimization can overload parts of the DNS
infrastructure due to doubling the number of queries.


# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
