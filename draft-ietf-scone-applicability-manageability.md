---
title: "Manageability Considerations for SCONE"
abbrev: "SCONE Manageability"
docname: draft-ietf-scone-applicability-manageability-latest
category: info
submissionType: IETF

ipr: trust200902
area: "Web and Internet Transport"
workgroup: "Standard Communication with Network Elements"
keyword: [SCONE, access networks, bitrate, throughput advice, manageability]

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
  SCONE: I-D.ietf-scone-protocol


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
This document describes the manageability considerations for network operators
for providing throughput advice to
application endpoints. This guidance is specifically addressed within the context of
communication networks utilizing the Standard Communication with Network Elements (SCONE) protocol {{SCONE}}.

--- middle

# Introduction

The SCONE protocol {{SCONE}} provides a signaling mechanism that enables on-path, SCONE-capable network elements to communicate "throughput advice", the advisory maximum sustainable throughput, to application endpoints via SCONE packets in the communication networks.

Network elements capable of rate limiting can send notifications of the advisory maximum sustainable throughput in each direction of the observed traffic. This allows applications, particularly those using Adaptive bitrate (ABR)
mechanisms,to proactively align their transmission rates with network policies. This document addresses the
manageability considerations for deploying the SCONE protocol within communication networks.
The SCONE core protocol specification provides guidance for network deployment,
specifically details on how to apply throughput advice signals in the SCONE header,
on protocol requirements on flow monitoring and considerations for flows that exceed
throughput advice (see {{Section 7 of SCONE}}). This document complements this guidance
by discussing various deployment options and providing additional manageability
considerations for network operators.

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
self-adapt to the throughput advice rather than relying on network rate limiters such as policers
or shapers, and the network can update the throughput advice for an active flow, including to
support tiered subscriber data plans (see {{Section 3.2 of SCONE}}).

When on-path network elements are present between the server and the client
application end-points, their specific configuration and role will influence the advice they
generate. Different network architectures handle flow visibility and policy enforcement at
different points. In mobile networks, for example, the User Plane Function (UPF) in 5G {{5G-Arch}}
and the Packet Data Network Gateway (P-GW) in 4G network {{4G-Arch}} can generate throughput
advice to guide ABR applications on a per-flow basis. In contrast, other environments,
such as wireline broadband or Wi-Fi, may apply policies at centralized aggregation points
or gateways such as the Broadband Network Gateway serving multiple devices.


# Terms and Definitions

This document uses terms and definitions described in {{SCONE}}.

# Manageability and Operational Considerations

Encompassing deployment of network elements in a wide range of networks, this document
is limited to discussing the core manageability and operational considerations for
the SCONE protocol to support its effective use across these varied network types.


## Flow Awareness and Per-Flow Signaling
As defined in the core SCONE protocol specification {{SCONE}},
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
As specified in {{Section 6.1 of SCONE}}, SCONE-aware endpoints add SCONE indication bytes
as the last two bytes of the payload of each UDP datagram that starts a new flow, to support the
identification of a SCONE-capable flow without any need for compute-intensive flow
classification. Additionally, SCONE-capable endpoints, through rate self-adaptation, remove
the need for complex rate-limiting functions in the network element. Support for SCONE
indication and rate self-adaptation reduces complexity and CPU processing load in the network
element. Similarly, as explained in {{Section 7.1 of SCONE}}, detection and modification of a
SCONE packet to update the rate signal field is also a lightweight operation for a network
element.

## Network Element Overload Handling

Processing SCONE packets creates a per-flow update obligation for
network elements. To avoid throughput advice expiring
({{Section 5.4 of SCONE}}), each active flow
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
traffic pattern ({{Section 7.1 of SCONE}}, "Applying
Throughput Advice Signals"). A network element cannot expect a predictable or
uniform signaling cadence from the traffic itself, and instead decides its own
update interval within that flexibility.

Applications are incentivized to keep the sending frequency low to
avoid high overhead, but still need to manage a sending rate that
enables adaptation to throughput advice changes in a reasonable time.
Even a high frequency is expected to be on the order of multiple
seconds, since most applications can only apply a change at discrete
intervals, such as an ABR application's next video segment. Sending at
a much higher frequency than that would only waste resources for both
the network and the endpoints without additional benefit.

A SCONE-enabled network element updates advice in SCONE packets at least twice
per the 67-second monitoring period ({{Section 5.2 of SCONE}},
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
the network element updates the next SCONE packet it observes,
rather than waiting for its update interval to elapse, to minimize the
application's reaction time to the new limit (see {{Section 9.3 of SCONE}}).
How soon the application sees the change depends on when the endpoint next sends a SCONE packet,
since the network element cannot originate one.

## Presence of Multiple SCONE Network Elements On the Data Path

When multiple SCONE-capable network elements are present on the same data path, they operate
independently, with no synchronization or control-plane coordination required between them.
A network element might receive a packet that already includes a rate signal
set to a valid value (not unknown). The network element replaces the rate signal
if it wishes to signal a lower value for throughput advice; otherwise,
the original values are retained, preserving any lower advice already set by
another element on the path. This way the endpoint applies the most restrictive advice along the
path (see {{Section 7.1 of SCONE}}). This lets operators deploy and manage
SCONE network elements independently, without building integration between them.


## Change of Network Element

If the network path changes, for example during QUIC connection migration, the flow
can be routed through a different SCONE network element. Because SCONE signaling is
stateless, this transition requires no explicit teardown or state transfer between
the old and new network elements.

As defined in {{Section 6.3 of SCONE}}, the endpoint sends
SCONE packets early on the new path, so the new network element can detect the flow and provide
its own advice. Because this is not a new flow, the new network element does not observe the
flow-start indication described in {{Section 6.1 of SCONE}}. It detects the flow directly from
the SCONE packet itself.

If no SCONE-capable element is present on the new path, the previous advice
expires after a monitoring period ({{Section 5.4 of SCONE}}) and the
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
{{Section 3.5 of SCONE}}. Conversely, if the
network element stops observing traversing SCONE packets
arriving from the sender, this suggests either an upstream delivery
failure or the endpoint is no longer sending SCONE packets on that flow.
Recording both the timestamps of updates to packets and the subsequent
per-flow throughput measurements in the logging infrastructure
makes this correlation possible, aligning with the monitoring
guidance in {{Section 7.2 of SCONE}}.


## Conformance Measurement {#conformance-monitoring}
Networks that choose to provide SCONE throughput advice can implement mechanisms to
monitor QUIC flows and measure conformance to the throughput advice, either per flow of
packets on the same UDP address tuple, or in aggregate across multiple QUIC flows if they
contribute to a shared policy limit (such as a device or network subscription level). This
will allow operators to validate whether applications are following the throughput advice.

While it is expected that operators will implement monitoring at the SCONE Network Element
providing the advice, it could also be performed elsewhere in the network. However, network
elements lack the capability to validate the legitimacy of SCONE packets coalesced with other
QUIC packets. Therefore, operators need to ensure a network element evaluates conformance only
against the throughput advice that it set itself, rather than enforcing limits based on advice
set by other downstream network elements.

When evaluating compliance, network operators will need to account for the time required for
SCONE packets to be updated, received by endpoints, and acted upon by the application.
Operators can accommodate this by utilizing a sliding window approach. Operators should
evaluate QUIC flows against the highest throughput limit advised over the preceding two
monitoring periods (a span of 134 seconds). If a network element cannot update the
throughput advice in every traversing SCONE packet, operators might configure a
longer sliding window to account for the possibility of packet loss.

{{Section 7.2 of SCONE}} explains that the second monitoring period in this
window compensates for SCONE packets not being delivered reliably. Two distinct delays make
up that margin. The first, and usually the dominant one, is simply the interval between SCONE
packets. A network element can only act when one arrives, so its updates lag the endpoint's
own sending cadence. Network transit delay and the endpoint's own processing delay are both
small by comparison and can typically be disregarded. The second is the possibility that a
given SCONE packet carrying updated advice is lost before reaching the endpoint, against which
the additional monitoring period also provides margin.

A network element that tracks how many SCONE packets it has sent since
a change in advice may be able to shorten this window for its own
purposes. If it has sent at least two updates carrying the new advice,
it can restart the monitoring period for one more 67-second interval
and reach a decision after that period, rather than waiting a full two
periods from the original change. Operators that do not track sent
updates this closely should simply wait the full two periods, which
remains the safe and simpler choice. {{Section 7.2 of SCONE}} presents
the two-period baseline as illustrative, not mandatory, but cautions
that monitoring more strictly than that baseline risks misclassifying a
compliant application as non-conformant, while a longer window carries
no such risk. Operators should only shorten the window when they can
track sent updates this reliably, weighing the added tracking
complexity and how a missed update is treated against the benefit of a
faster measurement cycle.

To simplify the measurement function, reduce computational load, or offload this
function to another node in the network, operators can select any value larger
than the baseline 67 second window ({{Section 5.2 of SCONE}})
for their measurement and averaging period.

Because some applications will not support SCONE, and others either will not or cannot follow
the provided throughput advice, operators have flexibility in how they handle flows that exceed the limits that are set in their policies.
Before applying rate-limiting (throttling) mechanisms on SCONE-capable flows, operators can use conformance measurement and/or diagnostic approach described in
{{monitoring-and-logging}} to check that the flow requires intervention
in order to maintain target rates per {{Section 7.3 of SCONE}}.
If the conformance measurement function detects that an application is not following the
signaled throughput advice, the network can employ traditional rate-limiting mechanisms, such as dropping or delaying packets, to ensure
the QUIC flow does not exceed the throughput limits set by network policy. Alternatively, operators
can deploy SCONE purely as an advisory signal without any rate-limiting mechanism fallback, prioritizing
cooperative application optimization over strict compliance enforcement.

## Network Integration of SCONE In-Band Signaling {#network-integration}
Because SCONE packets are always coalesced with ordinary QUIC packets, SCONE signaling
operates entirely in-band. SCONE signaling
inherently traverses the already established network path, such as the existing
connection between a user device and a network gateway, associated with the QUIC flow
for which the network element intends to send throughput advice. As such,
SCONE seamlessly integrates into existing network architectures without requiring
new data paths, e.g. using tunnels, nor introducing any additional routing overhead.
Similarly, operators are not required to deploy any additional out-of-band signaling interfaces.


## Interworking with Other Congestion Management Mechanisms

As stated in {{Section 3.1 of SCONE}},
SCONE throughput advice is not a substitute for congestion control
utilizing congestion signals such as packet loss based on transport acknowledgments, delay, or
Explicit Congestion Notification (ECN) {{RFC3168}} as also used by Low Latency, Low Loss, and Scalable
Throughput (L4S) {{RFC9330}}. Rather, they are complementary.

Congestion control applies a throughput limit different from the signaled throughput advice.
Congestion control manages the immediate dynamics of the bottleneck
link, while SCONE informs the application of the maximum rate allowed by network policy.
Congestion signals provide real-time
information on transient congestion for a network path, typically operating
on the time scale of a round-trip time (RTT), whereas  SCONE throughput advice operates
over a much longer period.

When the throughput advice is below the
congestion limit and the application adheres to it, congestion control remains
application-limited and does not act. However, in cases of high load, congestion control would
limit the throughput below the provided advice, as the throughput advice is only an upper limit. As such,
congestion control or a similar mechanism to react to congestion, such as a circuit breaker, is always
needed in addition to SCONE.

Operators can use SCONE to communicate static or dynamically adapted advice based on available
policy information, e.g. based on subscriber data, or load management policies.
Operators set this advice independent of
instantaneous link congestion and do not need to coordinate throughput advice policies with
congestion control. It is then up to the applications, such as ABR video
clients or bulk downloads, to utilize this advice according to their specific use cases.

Network operators can deploy SCONE alongside L4S or standard ECN as two
independent network functions.
Throughput advice is carried within the QUIC payload, which does not interact
with or modify ECN markings of the IP-layer ECN field.
Real-time congestion feedback mechanisms remain outside the SCONE domain.

# Security Considerations
This document does not add any additional security considerations. The core security and
privacy considerations for the SCONE protocol are comprehensively defined in
{{Sections 9 and 10 of SCONE}}.

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
