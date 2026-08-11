---
title: "Applicability & Manageability Considerations for SCONE"
abbrev: "SCONE Applicability & Manageability"
docname: draft-ietf-scone-applicability-manageability-latest
category: info
submissionType: IETF

ipr: trust200902
area: "Web and Internet Transport"
workgroup: "Standard Communication with Network Elements"

keyword: [SCONE, access networks, bitrate, throughput advice, applicability, manageability]

stand_alone: yes
smart_quotes: no
pi: [toc, sortrefs, symrefs]

author:
  -
    ins: S. Mishra
    name: Sanjay Mishra
    organization: Verizon
    email: sanjay.mishra@verizon.com
  -
    ins: Z. Sarker
    name: Zaheduzzaman Sarker
    organization: Nokia
    email: zaheduzzaman.sarker@nokia.com
  -
    ins: A. Tomar
    name: Anoop Tomar
    organization: Meta
    email: anooptomar@meta.com
  -
    ins: K. Abbas
    name: Khurram Abbas
    organization: Verizon
    email: khurram.abbas@verizonwireless.com

normative:
  I-D.ietf-scone-protocol:


informative:
  4G-Arch:
    target: https://portal.3gpp.org/desktopmodules/Specifications/SpecificationDetails.aspx?specificationId=24300
    title: System Architecture for the Evolved Packet Core (EPC)
    author:
      - name: 3GPP
    date: 2020-06-01
  5G-Arch:
    target: https://portal.3gpp.org/desktopmodules/Specifications/SpecificationDetails.aspx?specificationId=3144
    title: System Architecture for the 5G System (5GS)
    author:
      - name: 3GPP
    date: 2025-01-07
  RFC3168: RFC3168
  RFC9330: RFC9330


--- abstract
This document describes the Applicability and Manageability considerations for providing throughput guidance to
application endpoints. This guidance is specifically addressed within the context of
communication networks utilizing the Standard Communication with Network Elements (SCONE) protocol.

--- middle

# Introduction

The SCONE protocol {{I-D.ietf-scone-protocol}} provides a signaling mechanism that enables on-path, SCONE-capable network elements to communicate "throughput advice", the advisory maximum sustainable throughput, to application endpoints via SCONE packets in the communication networks.

Network elements capable of rate limiting can send notifications of the advisory maximum sustainable throughput in each direction of the observed traffic. This allows applications, particularly those using Adaptive bitrate (ABR)
mechanisms,to proactively align their transmission rates with network policies. This document addresses the
Applicability and Manageability considerations for deploying the SCONE protocol within communication networks.
It also addresses operational, configuration, and management aspects not covered in the core protocol specification.

## SCONE Protocol Overview and Network Element Function {#scone-overview}

Deploying SCONE in a communication network involves the application
endpoints and any SCONE-capable network elements along the path of a
flow of QUIC UDP datagrams. SCONE is used on a QUIC flow only when its application endpoints
support it. SCONE packets are QUIC long header packets carrying a
SCONE-specific version number, coalesced into the same UDP datagram
as the QUIC packets they accompany.
This ensures that network elements that forward QUIC packets also forward
the SCONE packets.
Any network element can
implement the SCONE network element function that sets
the throughput advice.

To participate in SCONE, a network element is assumed to have the
functional capability to identify and track SCONE-compliant QUIC
flows, recognize and process SCONE packets within those flows, and
map network policies into throughput advice to be inserted into the
SCONE packets (see also {{network-integration}}).

By providing a standardized mechanism, SCONE allows network operators to provide
throughput advice information to QUIC endpoints without custom APIs or per-network integrations. Applications can
self-adapt to the advised bitrate rather than relying on network rate limiters such as policers
or shapers, and the network can update the advised bitrate for an active flow, including to
support tiered subscriber data plans (see {{Section 3.2 of I-D.ietf-scone-protocol}}).

When on-path network elements are present between the server and the client
application end-points, their specific configuration and role will influence the advice they
generate. Different network architectures handle flow visibility and policy enforcement at
different points. In mobile networks, for example, the User Plane Function (UPF) in 5G {{5G-Arch}}
and the Packet Data Network Gateway (P-GW) in 4G network {{4G-Arch}} can generate throughput
advice to guide ABR applications on a per-flow basis. In contrast, other environments,
such as wireline broadband or Wi-Fi, may apply policies at centralized aggregation points
or gateways such as the Broadband Network Gateway serving multiple devices.

# Terms and Definitions

This document uses terms and definitions described in {{I-D.ietf-scone-protocol}}.

# Applicability, Manageability and Operational Considerations

Encompassing deployment of network elements in a wide range of networks, this document
is limited to discussing the core manageability and operational considerations for
the SCONE protocol to support its effective use across these varied network types.

## Flow Awareness and Per-Flow Signaling
As defined in the core SCONE protocol specification {{I-D.ietf-scone-protocol}},
throughput advice is associated with the flow of QUIC UDP datagrams sharing the
same address tuple (IP version, source and destination IP addresses, and UDP ports).

Because throughput advice applies strictly to this specific flow, SCONE network elements
need to unambiguously associate their policy limits with the correct QUIC flows. However,
the act of applying SCONE throughput advice is inherently stateless. To provide advice, a
network element simply identifies a traversing SCONE packet and updates its value based on
the configured policy for that flow or network scope, without needing to maintain active
per-flow state.

While the signaling itself is stateless, managing the operational lifecycle of a SCONE
deployment may require establishing and maintaining per-flow context. Specifically, to execute
the monitoring, logging, and conformance evaluation functions detailed later in this document
(see {{conformance-monitoring}}),
the network element has to track the flow's throughput over multiple monitoring periods. This
per-flow context serves as the operational foundation for validating whether an application is
adhering to the advised rate and for applying any potentially necessary policy enforcement.

## Determining Throughput Constraints
The specific algorithms used to calculate throughput advice are highly
dependent on an operator's network architecture. In practice, these
constraints are often derived from a combination of network policies,
real-time conditions where applicable, and any other business logic
the operator applies. The inputs below are illustrative and will likely
vary by operator, and they are not exhaustive. A SCONE-capable network
element may derive its throughput advice from one or more of the
following:

- Subscriber Policies and Data Plans: The network element bases its throughput advice on the subscriber's data plan. This includes cases where rate limits apply once a subscriber's data volume usage reaches a threshold or usage cap.

- Application-Specific Policies: Operators may set maximum bitrates for
certain types of traffic based on subscription tier or device type, for
example video optimization for ABR video, or traffic
shaping for low-priority bulk transfers such as background software
updates.

- Dynamic Network Conditions: Constraints may be updated as network
conditions change, for example when a flow moves to a different access
network.

- Capacity and Load Management: During periods of unusually high usage,
sustained overuse, or temporary equipment faults, the network element
may temporarily lower its throughput advice to manage shared capacity
and guide application usage.

## Considerations of Processing Complexity {#processing-complexity}
As specified in {{Section 6.1 of I-D.ietf-scone-protocol}}, SCONE-aware endpoints provide
a specific indication on the first SCONE packet to support the identification of a SCONE-capable flow
without any need for compute-intensive flow classification. Additionally, SCONE-capable endpoints,
through rate self-adaptation, remove the need for complex rate-limiting functions in the network
element. Support for SCONE indication and rate self-adaptation reduces complexity and CPU processing
load in the network element.

## Network Element Overload Handling

Processing SCONE packets creates a per-flow update obligation for
network elements. To avoid throughput advice expiring
({{Section 5.4 of I-D.ietf-scone-protocol}}), each active flow
needs at least one rate signal update within each monitoring
period. When the number of concurrent SCONE flows grows unusually
large, as can occur during periods of elevated traffic or network
stress, this update obligation can become a processing concern.

Network element vendors and operators have two broad options for
managing this. One is to provision the network element with
capacity that accommodates the expected range of concurrent flows,
recognizing that what constitutes adequate capacity will vary by
deployment. The other is to handle the condition gracefully when
that capacity is reached. One approach for graceful handling is
for the network element to stop updating SCONE packets for a
subset of flows entirely, rather than attempting to serve all
flows with intermittent or delayed updates. A flow that does not observe a SCONE update
for a full monitoring period will have its
throughput advice expire, causing the endpoint to operate without
SCONE-advice. This
outcome is more predictable than partial updates, which can cause
endpoints to alternate between operating with SCONE throughput
advice and operating without it. Which flows continue to receive
updates and how that selection changes over time are operator
policy decisions. Network element implementations can provide the
configuration options through which operators define and adjust
that policy.

## Reliability of Throughput Advice Delivery
The frequency at which SCONE packets arrive at a network element is set by the
application endpoint, not the network, and varies by application type and
traffic pattern ({{Section 7.1 of I-D.ietf-scone-protocol}}, "Applying
Throughput Advice Signals"). A network element cannot expect a predictable or
uniform signaling cadence from the traffic itself, and instead decides its own
update interval within that flexibility.

A SCONE-enabled network element updates advice in SCONE packets at least twice
per the 67-second monitoring period ({{Section 5.2 of I-D.ietf-scone-protocol}},
approximately every 20 to 30 seconds). This baseline ensures advice reliably
reaches the endpoint and does not expire across the monitoring period, since
SCONE packets are not delivered reliably for a variety of reasons. Operators
may choose to update more frequently than this baseline, at the cost of
additional network element CPU load, in exchange for a lower chance that
advice goes unheard during any single window. A network enforcing fixed,
subscription-based policies can typically rely on this baseline, since the
applicable policy for a flow does not usually change once established. In
practice, the network element can only update advice when a SCONE packet
is available to modify. When the interval has
elapsed, the network element waits for the next SCONE packet, and that arrival is
what sets the real interval between updates.

## Sending Updates for Dynamic Target Throughput Changes
Target throughput advice can change while a flow is active, for example when a
subscriber reaches a data threshold or when the radio access technology changes
or when the bandwidth allocated to the subscriber or the bearer changes
while the flow is already established. When this happens,
the network element prioritizes updating the next traversing SCONE packet
promptly, bypassing its scheduled periodic update interval, to minimize the
application's reaction time to the new limit (see {{Section 9.2 of I-D.ietf-scone-protocol}}).
How soon the application sees the change depends on when the endpoint next sends a SCONE packet,
since the network element cannot originate one.

## Presence of SCONE Network Elements On the Data Path
When multiple SCONE-capable network elements are present on the same data path, they operate
independently, with no synchronization or control-plane coordination required between them.
Each network element only lowers the rate signal, preserving any lower advice already set by
another element on the path, so the endpoint applies the most restrictive advice along the
path (see {{Section 5.4 of I-D.ietf-scone-protocol}}). This lets operators deploy and manage
SCONE network elements independently, without building integration between them.

## Change of Network Element During an Active Flow
The on-path network element can change when an application changes its access network, for
example during QUIC connection migration or a mobility event where the IP address is unchanged.
Because SCONE signaling is stateless, this transition needs no explicit teardown or state
transfer between the old and new network elements. The endpoint and network elements follow the
migration steps defined in {{Section 6.3 of I-D.ietf-scone-protocol}}, where the endpoint sends
SCONE packets early on the new path so a network element there can detect the flow and provide
its own advice. If no SCONE-capable element is present on the new path, the previous advice
expires after a monitoring period ({{Section 5.4 of I-D.ietf-scone-protocol}}) and the
application operates without SCONE-advised limits.

## Monitoring and Logging {#monitoring-and-logging}
SCONE signaling can be integrated into existing operational and
management frameworks to enable monitoring, troubleshooting, and fault
isolation. Metrics of interest include:

- Rate of SCONE advisory messages issued per session
- Correlation between SCONE advisories and user-plane throughput changes
- Error conditions where SCONE signaling fails to reach the intended endpoints

When throughput advice does not appear to be followed, it is
useful to determine whether the advice reached the application
at all, or whether the application received it but did not act
on it. Operators can narrow the diagnosis by
correlating the rate of SCONE advisories issued at the network
element against the observed per-flow throughput over the
following two monitoring periods. If the network element
successfully updates traversing SCONE packets during that window
but the flow's throughput does not change, it indicates either
that all updated packets were dropped downstream before reaching the
application, or that the application received the advice but did
not act upon it, as SCONE is an advisory signal per
{{Section 3.5 of I-D.ietf-scone-protocol}}. Conversely, if the
network element stops observing traversing SCONE packets
arriving from the sender, this suggests either an upstream delivery
failure or the endpoint is no longer sending SCONE packets on that flow.
Recording both the timestamps of updates to packets and the subsequent
per-flow throughput measurements in the logging infrastructure
makes this correlation possible, aligning with the monitoring
guidance in {{Section 7.2 of I-D.ietf-scone-protocol}}.


## Conformance Measurement {#conformance-monitoring}
Networks that choose to provide SCONE throughput advice can implement mechanisms to
monitor QUIC flows and measure conformance to the throughput advice, either per flow of
packets on the same UDP address tuple, or in aggregate across multiple QUIC flows if they
contribute to a shared policy limit (such as a device or network subscription level). This
will allow operators to validate whether applications are following the throughput advice.

While it is expected that operators will implement monitoring at the SCONE Network Element
providing the advice, it could also be performed elsewhere in the network. However, network
elements lack the capability to validate the legitimacy of SCONE packets coalesced with other
QUIC packets. Therefore, operators must ensure a network element evaluates conformance only
against the throughput advice that it set itself, and never enforces limits based on advice
set by other downstream network elements.

When evaluating compliance, network operators will need to account for the time required for
SCONE packets to be updated, received by endpoints, and acted upon by the application.
Operators can accommodate this by utilizing a sliding window approach. Operators should
evaluate QUIC flows against the highest throughput limit advised over the preceding two
monitoring periods (a span of 134 seconds). If a network element cannot update the
throughput advice in every traversing SCONE packet, operators might configure a
longer sliding window to account for the possibility of packet loss.

To simplify the measurement function, reduce computational load, or offload this
function to another node in the network, operators can select any value larger
than the baseline 67 second window ({{Section 5.2 of I-D.ietf-scone-protocol}})
for their measurement and averaging period.

Because some applications will not support SCONE, and others either will not or cannot follow
the provided throughput advice, operators have flexibility in how they handle flows that exceed the limits that are set in their policies.
Before applying rate-limiting (throttling) mechanisms on SCONE-capable flows, operators can use conformance measurement and/or diagnostic approach described in
{{monitoring-and-logging}} to check that the flow requires intervention
in order to maintain target rates per {{Section 7.3 of I-D.ietf-scone-protocol}}.
If the conformance measurement function detects that an application is not respecting the
signaled throughput advice, the network can employ traditional rate-limiting mechanisms, such as dropping or delaying packets, to ensure
the QUIC flow does not exceed the throughput limits set by network policy. Alternatively, operators
can deploy SCONE purely as an advisory signal without any rate-limiting mechanism fallback, prioritizing
cooperative application optimization over strict compliance enforcement.

## In-Band Signaling and Network Integration {#network-integration}
Because SCONE packets are always coalesced with ordinary QUIC packets, SCONE signaling
operates entirely in-band. It does not introduce any additional routing overhead or
require the creation of out-of-band signaling interfaces. Instead, SCONE signaling
inherently traverses the already established network path, such as the existing
connection between a user device and a network gateway, associated with the QUIC flow
for which the network element intends to send throughput advice. This ensures that
SCONE seamlessly integrates into existing architectures without requiring new tunnels
or data paths to be established.


## Interworking with Other Congestion Management Mechanisms
SCONE throughput advice is not a substitute for congestion control mechanisms in
transport protocols utilizing congestion feedback and signals such as acknowledgments,
Explicit Congestion Notification (ECN) {{RFC3168}}, and Low Latency, Low Loss, and Scalable
Throughput (L4S) {{RFC9330}}. Rather, they are complementary. Congestion signals provide real-time
information on loss, delay, and transient congestion for a network path, typically operating
on the time scale of a round-trip time (RTT). In contrast, SCONE throughput advice operates
over a much longer period. Because the network element is generally unaware of the specific
application traffic, it simply provides static or dynamically adapted advice based on available
policy information. Operators can use SCONE to communicate a maximum sustainable throughput
driven by video optimization, subscriber data, or load management policies, independent of
instantaneous link congestion. It is then up to the applications, such as ABR video
clients or bulk downloads, to utilize this advice according to their specific use cases.

For network operators considering co-deployment, SCONE throughput advice is strictly independent
of the IP-layer ECN field. Because SCONE advice is carried within the QUIC payload, updating the
advice does not interact with or modify ECN markings. This independence ensures that operators can
safely deploy SCONE alongside L4S or standard ECN. Real-time congestion feedback mechanisms remain
fully operational and function completely outside the SCONE domain.

Operators should expect that congestion signals might frequently indicate a throughput limit different
from the signaled SCONE advice. In other words, in the best case, the throughput advice is below the
congestion limit and when the application adheres to the advice, congestion control would be
application-limited and not go into action. However, in cases of high load, congestion control would
limit the throughput below the provided advice, as the SCONE advice is only an upper limit. As such,
congestion control or a similar mechanism to react to congestion, such as a circuit breaker, is always
needed in addition to SCONE.

In environments where both are present, congestion control manages the immediate dynamics of the bottleneck
link, while SCONE informs the application of the maximum rate allowed by network policy. Network operators
will benefit from ensuring that throughput advice policies and congestion control configurations are consistent
within scoped deployments, to avoid providing conflicting feedback to applications.

# Security Considerations
This document does not add any additional security considerations. The core security and
privacy considerations for the SCONE protocol are comprehensively defined in
{{Sections 9 and 10 of I-D.ietf-scone-protocol}}.

# IANA Considerations
This document has no IANA actions.

--- back

# Acknowledgments
{:numbered="false"}

The authors thank Wesley Eddy, Renjie Tang, Kevin Smith, Tina Tsou, Tianji Jiang, Lucas Pardue,
and Martin Thomson for their helpful comments and contributions to this document.

The authors are also grateful to Mirja Kühlewind, Gorry Fairhurst, Marcus Ihlar, Qin Wu,
Christian Huitema, and Brian Trammell for their detailed reviews and contributions, which
substantially improved this document.

The authors also thank members of the SCONE Working Group for their review and support
throughout the development of this document.
