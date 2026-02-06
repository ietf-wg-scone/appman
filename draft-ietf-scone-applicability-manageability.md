---
title: "Applicability & Manageability Considerations for SCONE"
abbrev: "SCONE Applicability & Manageability"
docname: draft-ietf-scone-applicability-manageability-latest
category: info
submissionType: IETF

ipr: trust200902
area: "Web and Internet Transport"
workgroup: "Standard Communication with Network Elements"

keyword: [SCONE, access networks, bit rate, throughput advice, applicability, manageability]

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
    target: https://datatracker.ietf.org/doc/draft-ietf-scone-protocol/
    title: Standard Communication with Network Elements (SCONE) Protocol
    author:
      - name: M. Thomson
      - name: C. Huitema
      - name: K. Oku
      - name: M. Joras
      - name: M. Ihlar
    date: 2025-07
    seriesinfo: "Internet-Draft, draft-ietf-scone-protocol, Work in Progress"

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

--- abstract

This document describes the Applicability and Manageability considerations for providing throughput guidance to
application endpoints. This guidance is specifically addressed within the context of telecommunications service
provider networks utilizing the Standard Communication with Network Elements (SCONE) protocol.

--- middle

# Introduction

The SCONE protocol {{I-D.ietf-scone-protocol}} provides a signaling mechanism that enables on-path, SCONE-capable network elements to
communicate "throughput advice", the advisory maximum allowable bit rate, to application endpoints via SCONE packets in the telecommunications
service provider networks.

Network elements capable of rate limiting can send notifications of the advisory maximum allowable bit rate in each direction of the observed
traffic. This allows applications, particularly those using adaptive bit-rate (ABR) mechanisms, to proactively align their transmission rates
with network policies. This document addresses the Applicability and Manageability considerations for deploying the SCONE protocol within service
provider networks. It also addresses operational, configuration, and management aspects not covered in the core protocol specification.

To participate in SCONE, a network element is assumed to have the functional capability to identify and track scone compliant application flows,
recognize and process SCONE packets within those flows and map network policies into throughput advice to be inserted into the SCONE packets. While
on-path network elements may exist at various points between the server and the client application end-points, their specific configuration and role
will influence the advice they generate. Different network architectures handle flow visibility and policy enforcement at different points.
In mobile networks, for example, the User Plane Function (UPF) in 5G {{5G-Arch}} and the Packet Data Network Gateway (P-GW) in 4G network {{4G-Arch}}
can generate throughput advice to guide ABR applications on a per-flow basis. In contrast, other environments, such as wireline broadband or Wi-Fi,
may apply policies at centralized aggregation points or gateways such as the Broadband Network Gateway serving multiple devices.

Encompassing deployment of network elements in a wide range of networks, this document is limited to discussing the core Applicability and Manageability
considerations for the SCONE protocol to ensure its consistent and effective use across varied network paths.


# Terms and Definitions

This document uses terms and definitions described in {{I-D.ietf-scone-protocol}}.

# Applicability and Manageability Considerations

## Flow session awareness
SCONE signaling operates only over established sessions. SCONE Network Elements
ought to be able to unambiguously associate throughput advice with
application flows. Each session is bound to an IP address and port,
ensuring SCONE packets are routed precisely without affecting unrelated traffic.

## Per-Flow Signaling
Throughput advice is applied on a UDP 4-tuple basis. SCONE Network Elements
ought to maintain flow-specific context to ensure signaling correctness.
This enables applications to receive targeted throughput advice while
preventing unintended impact on unrelated flows.

## QoS awareness
Quality of Service (QoS) may be enforced by networks through a variety of
mechanisms. In certain deployments, network operators may choose to apply distinct
QoS policies to SCONE-enabled flows. The SCONE Network Element
responsible for inserting SCONE advice is not required to interpret or
enforce QoS policies; its role is limited to the signaling of the advisory
throughput information. It is expected that network operators shall be able to identify
SCONE-enabled flows and, where appropriate, provide throughput advice in accordance
to their policy objectives.

## SCONE Hint to the Network
SCONE-aware applications ought to provide hints to the SCONE Network Elements,
enabling it to generate appropriate throughput advice for a given
UDP 4-tuple. Such hints prevent unnecessary default rate-limiting, allow the
network to signal the maximum allowable bit rate, and reduce CPU
overhead by eliminating additional classification steps.

## Retransmission of Advised Bit-Rate
Packet loss or non-delivery of SCONE advice reduces its effectiveness. Both
SCONE Network Elements and application endpoints should support retransmission or
periodic re-sending of SCONE packets to ensure reliable delivery.
Conformance depends on the behavior of both network and application endpoint.

## Frequency of Updates
The rate at which SCONE updates are issued depends on flow
characteristics and available computational resources. Excessively
frequent updates may increase CPU load, while infrequent updates may
reduce advisory effectiveness. Network Operators can define
adjustable update intervals based on application requirements, network
capacity, and operational constraints.

## Dynamic Updates
Dynamic rate limits updates can be enforced by the network during active
application sessions due to:

- Changes in access network type (requiring updated throughput advice)
- Changes in Subscriber policy (e.g., exceeding usage thresholds)

In such cases, the SCONE Network Elements need to be able to initiate SCONE
packets to provide updated advice, or applications should generate SCONE
packets frequently enough to trigger network responses.

## Monitoring and Logging
SCONE signaling can be integrated into existing operational and
management frameworks to enable monitoring, troubleshooting, and fault
isolation. Metrics of interest include:

- Rate of SCONE advisory messages issued per session
- Correlation between SCONE advisories and user-plane throughput changes
- Error conditions where SCONE signaling fails to reach the intended endpoints

## Conformance Monitoring
Networks providing SCONE throughput advice ought to implement
mechanisms to measure compliance, either per application flow or in
aggregate. This allows operators to validate advisory effectiveness and
adjust policies. Due to flow awareness, such mechanisms are typically
implemented in a SCONE Network Element but may also be implemented
elsewhere in the network.

## Standards Compliance
SCONE signaling is expected to traverse the existing data path associated
with the UDP 4-tuple flow for which the Network Element intends to send the advisory bit-rate.

## Interworking with Other Congestion Management Mechanisms
SCONE is distinct from transport-level congestion control mechanisms, such as
Explicit Congestion Notification (ECN) or Low Latency, Low Loss, and Scalable
Throughput (L4S). While congestion control operates on short timescales to manage
transient congestion caused by varying link conditions or instantaneous load, SCONE
provides throughput advice based on relatively stable network policies or capacity
management goals. ECN/L4S based congestion control works at transport level and SCONE
works at application level. SCONE does not replace the need for endpoints to perform
congestion control or network to provide explicit or implicit congestion signals; rather,
it complements these mechanisms by providing a variable range for application-level rate
adaptation. In environments where both are present, SCONE and congestion control mechanisms
co-exist: congestion control manages the immediate dynamics of the bottleneck link, while
SCONE informs the application of the maximum sustained rate allowed by policy. Network Operators
would benefit from harmonizing multiple congestion signaling methods by policy or scope
deployments to avoid conflicting feedback.


# Security Considerations
Security considerations are included separately in the SCONE protocol documents.

# IANA Considerations
This document has no IANA actions.

--- back

# Acknowledgments
{:numbered="false"}

The authors thank Wesley Eddy, Renjie Tang, Kevin Smith, Tina Tsou, Tianji Jiang, Lucas Pardue,
and Martin Thomson for their helpful comments and contributions to this document. The authors also
thank members of the SCONE Working Group for their review and support throughout the development of this document.
