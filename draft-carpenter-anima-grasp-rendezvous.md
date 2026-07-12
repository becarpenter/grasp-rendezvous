---
title: Using GRASP as an Agent Rendezvous Mechanism
abbrev: GRASP Rendezvous
docname: draft-carpenter-anima-grasp-rendezvous-latest
submissiontype: IETF
ipr: trust200902
area: "Operations and Management"
workgroup: "Autonomic Networking Integrated Model and Approach"
kw:
 - agent
 - discovery
cat: info
venue:
  group: "Autonomic Networking Integrated Model and Approach"
  type: "Working Group"
  mail: "anima@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/anima/"
  github: "becarpenter/grasp-rendezvous"
  latest: "https://becarpenter.github.io/grasp-rendezvous/draft-carpenter-anima-grasp-rendezvous.html"

pi:
  toc: yes
  sortrefs:   # defaults to yes
  symrefs: yes

author:

      -
        ins: B. E. Carpenter
        name: Brian E. Carpenter
        org: The University of Auckland
        abbrev: Univ. of Auckland
        postal:
        - School of Computer Science
        - The University of Auckland
        - PB 92019
        - Auckland 1142
        country: New Zealand
        email: brian.e.carpenter@gmail.com


normative:
  RFC8990:
  RFC8994:
  RFC8949:

informative:
  RFC8991:
  RFC8993:
  RFC9222:
  I-D.rosenberg-agentproto-usecases:
  I-D.zhao-nmop-network-management-agent:
  I-D.zeng-opsawg-applicability-mcp-a2a:
  I-D.eckert-anima-ai4an:
  I-D.farrel-dawn-terminology:
  I-D.king-dawn-requirements:

--- abstract

This document describes how the GeneRic Autonomic Signaling Protocol (GRASP) defined by RFC 8990 may be used as a rendezvous mechanism for one Autonomic Service Agent to find another, and then establish a generic communication channel between them. Such a channel could be used for any form of agent-to-agent (A2A) communication, not limited to GRASP exchanges.

--- middle

# Introduction        {#intro}

The GeneRic Autonomic Signaling Protocol (GRASP) is specified in {{RFC8990}},
and an API is described in {{RFC8991}}. Its purpose is to support discovery, data
synchronization, and negotiation among Autonomic Service Agents (ASAs)
in a self-managing autonomic network. A conceptual model of how
AI agents might fit into an autonomic network may be found in
{{I-D.eckert-anima-ai4an}}. This document addresses how such agents may
discover each other and establish communication.

For the general model of an autonomic network, and for terminology not otherwise
defined here or in RFC 8990, see {{RFC8993}}. General considerations for ASAs
are discussed in {{RFC9222}}.

A basic feature of GRASP is its discovery mechanism, using the M_DISCOVER
and M_RESPONSE messages, which allow an Autonomic Service Agent
to discover another ASA that supports a particular GRASP objective
(as defined in RFC 8990). This can provide the first stage of 'discovery'
as defined in {{I-D.farrel-dawn-terminology}}. 

Following this discovery process, ASAs may
conduct a GRASP synchronization session to share data, or a GRASP
negotiation session to agree on certain parameter settings.

However, in some cases the two agents may require to communicate in some
other way, outside the scope of GRASP synchronization or negotiation.
One example would be to complete the discovery requirements outlined
in {{I-D.king-dawn-requirements}}. More generally, they may need to
perform agent-to-agent (A2A) communication as discussed in documents such as
{{I-D.zeng-opsawg-applicability-mcp-a2a}},
{{I-D.zhao-nmop-network-management-agent}},
{{I-D.rosenberg-agentproto-usecases}} and
[https://a2a-protocol.org/dev/specification/](https://a2a-protocol.org/dev/specification/).
In this case, GRASP can be used purely as a generic rendezvous mechanism
between agents.

Note that GRASP discovery does not discover a specified agent. Instead, it
discovers an agent that supports a specified objective. For example, if
an agent wants to find support for an objective named
"example.org:translate_english_french", it will discover one or more agents
that support an objective of that name. As a result, an agent can be specialised
and only support one objective, or it could support several different
objectives. This is an implementation and deployment choice.

A feature of GRASP discovery that may be essential during a period of network
instability is that it has no dependency on any protocols above the network
layer, and in particular no dependency on DNS or mDNS.

# Rendezvous procedures

There are two methods for an ASA to use GRASP discovery to establish a communications channel.

The choice between the two methods will be fixed as part of the definition
of the GRASP objective concerned (see Section 2.10 of {{RFC8990}}).

## Simple rendezvous {#simple}

The first method is simply to perform discovery and use the resulting locators appropriately. This is most easily described in terms of the GRASP API {{RFC8991}}. First the client ASA issues a discovery call, e.g.,

~~~~
obj1 = objective("example.org:translate_english_french")
discover(asa_handle1, obj1, timeout, minimum_TTL)
~~~~

The client will receive in return a list of locators for ASAs that support the given objective.
According to Section 2.9.5 of {{RFC8990}} these locators may be IP addresses, FQDNs, or URIs.
Each locator is returned with a transport protocol identifier and a port number, if needed.
The client ASA may now use any of those locators to open a transport connection to
another ASA supporting the given objective, with no other GRASP messages required.

For this to operate successfully, the remote ASA must have previously registered itself,
via the GRASP API, as ready to support the same objective, e.g.,

~~~~
obj2 = objective("example.org:translate_english_french")
register(asa_handle2, obj2)
~~~~

This method has the advantage that any type of locator could be used and any transport
protocol could be used. However, both ASAs will have to support the chosen locator type
and transport protocol (one as a client and the other as a server), and provide adequate security.

## Rendezvous via brief GRASP negotiation

Another approach is possible, which creates a GRASP session identifier
and a vestigial GRASP negotiation session. Discovery proceeds as just described.
However, following discovery, the client ASA starts a GRASP negotiation process as
described in Section 2.5.5 of {{RFC8990}}, using the M_REQ_NEG message. Upon receipt
of an M_NEGOTIATE message in reply, instead of continuing to exchange M_NEGOTIATE messages,
both ASAs switch to a straightforward transport layer communication, using the channel
created by the M_REQ_NEG/M_NEGOTIATE exchange. Since GRASP is a CBOR-based protocol,
messages in this case are expected to be encoded in CBOR {{RFC8949}}. Since CBOR itself
can very readily encode JSON and therefore YANG, this is a rather general solution.

Note that although GRASP has a defined maximum message size, it will not limit
these messages, as they are not GRASP messages.

This method has the advantage that it can be entirely handled by GRASP mechanisms plus a
simple send/receive API. However, this means that FQDN and URI locators are not supported,
CBOR is required,
and only the transport protocol supported by GRASP is available (normally TCP protected
by Autonomic Control Plane {{RFC8994}} security).

# Implementation Status \[RFC Editor: please remove]

Rendezvous via GRASP negotiation has been implemented in the form of gsend()
and grecv() primitives added to a
[prototype version of GRASP](https://github.com/becarpenter/graspy). This
amounted to about 60 lines of Python code.

# Security Considerations

The security considerations of {{RFC8990}} apply. The normal deployment  scenario
for GRASP is to run over a secure Autonomic Control Plane {{RFC8994}}, which defines
a strongly enforced trust boundary and protects all traffic cryptographically.
All agents must lie within this trust boundary, which forms a single GRASP domain.

However, when an ASA registers itself as described in {{simple}}, this
registration merely indicates that the responding ASA has successfully joined
the ACP. It does not provide any authentication or authorization for the
agent as such. Mutual authentication and authorization between agents are
out of scope for the rendezvous mechanisms described here. They should be addressed
as part of the protocol used between the agents following a successful
rendezvous.

GRASP supports a flooding mechanism, by which an ASA can advertise the current
value of a GRASP objective to all other ASAs in the same domain. This mechanism
should only be used for information that is potentially needed by all agents.
It is not recommended as an alternative rendezvous mechanism.

# IANA Considerations

No IANA actions are required by this document. For considerations about naming
and registering GRASP objectives, see Section 2.10.1 of {{RFC8990}}.

--- back

# Change Log

## Draft-00

- Original version

# Acknowledgements
{:numbered="false"}

Helpful comments were made by
Artur Hecker,
韩梦瑶 (Han Mengyao),
...
