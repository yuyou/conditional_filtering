---
title: "Conditional Range Filters for Media over QUIC Transport"
abbrev: "Conditional Range Filters"
category: std
docname: draft-yuyou-conditional-filtering-latest
ipr: trust200902
submissiontype: IETF
consensus: true
v: 3
area: "Web and Internet Transport"
workgroup: "Media Over QUIC"
keyword:
  - media over quic
  - range filter
  - abr
venue:
  github: yuyou/conditional_filtering
  latest: https://yuyou.github.io/conditional_filtering/
author:
  -
    ins: S. Gül
    name: Serhan Gül
    organization: Nokia
    email: serhan.guel@nokia.com
  -
    ins: Y. You
    name: Yu You
    organization: Nokia
    email: yu.you@nokia.com
  -
    ins: A. Begen
    name: Ali Cengiz Begen
    organization: Networked Media
    email: ali.begen@networked.media
  -
    ins: A. Sarker
    name: ANM Zaheduzzaman Sarker
    organization: Nokia
    email: zaheduzzaman.sarker@nokia.com

normative:
  I-D.ietf-moq-transport:

informative:
  SSTS:
    title: Sender-Side Track Switching for Media over QUIC Transport
    target: https://github.com/wilaw/moqt-ssts/blob/main/draft-wilaw-moq-moqt-ssts.md

...

--- abstract

In Media over QUIC Transport (MOQT), subscribers can use Range Filters to select specific subgroups, objects, or priorities within a subscribed track. However, these subscription filters are static once established and can only be modified through explicit subscriber control signaling. This document proposes an extension to the Range Filter design that binds conditional evaluation logic directly to specific Range Filter sets. By introducing dynamic conditions to Range Filter configurations, a relay can autonomously adapt the intra-track forwarding behavior based on real-time network conditions, avoiding the round-trip delay of explicit subscriber update signaling.

--- middle

# Introduction

Media over QUIC Transport (MOQT) supports subscription filters in SUBSCRIBE requests to indicate which objects within a track are to be forwarded. Range Filters allow subscribers to restrict delivery based on subgroup identifiers, object identifiers, publisher priorities, or specific metadata properties.

However, current MOQT subscription filters are static once established. If a subscriber wishes to change the adaptation policy (e.g., dropping an enhancement subgroup due to network congestion), it requires a closed-loop procedure: detecting the throughput drop, computing a new filter, signaling via REQUEST_UPDATE, and waiting for the relay to apply it. This reaction delay introduces a risk of temporary overdelivery and queue build-up.

This document extends Range Filters with conditional logic that allows relays to evaluate pre-authorized conditions and determine which SetIDs remain active at runtime. It defines a new RANGE_FILTER_CONDITION parameter that explicitly binds a specific Range Filter SetID to a condition. If the condition for a particular SetID is met, the relay keeps that SetID active in the Range Filter expression; otherwise, that SetID is ignored. This enables autonomous, relay-side intra-track adaptation without requiring further signaling from the subscriber.

# Conventions and Definitions
{::boilerplate bcp14-tagged}

# The RANGE_FILTER_CONDITION Parameter

## Relationship to Existing SetID

Range Filters in MOQT (Section 5.1.3 of {{!I-D.ietf-moq-transport}}) already define a SetID field (8 bits) within each filter parameter (SUBGROUP_FILTER, OBJECTID_FILTER, PRIORITY_FILTER, OBJECT_PROPERTY_FILTER, TRACK_PROPERTY_FILTER). Filter parameters sharing the same SetID are combined with logical AND, and different SetID groups are combined with logical OR.

This document does not redefine SetID. RANGE_FILTER_CONDITION references an existing SetID value to bind conditional evaluation logic to a specific filter group that was already declared in the same message.

## Parameter Definition

To enable conditional functionality, this document introduces a new Message Parameter: RANGE_FILTER_CONDITION. This parameter applies conditional activation rules to an intra-track filter set identified by a SetID.

The parameter MAY appear in SUBSCRIBE, PUBLISH_OK, or REQUEST_UPDATE messages. Multiple instances MAY be present, each referencing a different SetID. The same SetID MUST NOT appear in more than one RANGE_FILTER_CONDITION within the same message.

The RANGE_FILTER_CONDITION parameter has the following wire format:

~~~
RANGE_FILTER_CONDITION {
  Set ID (8),
  Algorithm ID (vi64),
  Throughput threshold (vi64),
  Set throughput fraction (vi64),
  Activate switching (vi64),
  Set rank (8)
}
~~~

The fields are defined as follows:

Set ID (8 bits):
: Identifies the Range Filter set being made conditional. This value MUST match a SetID used in at least one Range Filter parameter in the same message. The field is 8 bits, consistent with the SetID encoding defined in MOQT Range Filters.

Algorithm ID (vi64):
: Identifies the conditional evaluation algorithm to be used. This document defines Algorithm 0. Other algorithms MAY be defined in future specifications.

Throughput threshold (vi64):
: Algorithm 0 specific. The minimum estimated downstream throughput in kbps required for the relay to keep this SetID active in the Range Filter OR-combination. When bandwidth is constrained, the relay determines which conditional SetIDs remain active according to the bandwidth allocation algorithm ({{bandwidth-allocation}}).

Set throughput fraction (vi64):
: Algorithm 0 specific. Relative weight for bandwidth allocation, expressed as an integer 1 <= N <= 10. Each conditional SetID receives bandwidth proportional to its fraction: `target = B_total x fraction / sum_F`. Fractions are relative weights, not absolute percentages. For example, fractions of 6, 4, 3 (sum_F = 13) allocate approximately 46%, 31%, and 23% respectively. This allows conditional SetIDs to be added or removed without requiring other conditional SetIDs to update their fractions.

Activate switching (vi64):
: Controls whether conditional evaluation is active for this SetID. When set to 0, the relay ignores the conditional state for this SetID and includes it in the base MOQT OR-combination exactly as ordinary Range Filters do. When set to a non-zero value N, the relay activates condition-based admission for conditionally bound SetIDs as soon as at least N RANGE_FILTER_CONDITION entries for this subscription have been received. Activation takes effect at the next Group boundary.

Set rank (8 bits):
: Degradation priority when estimated bandwidth is constrained, expressed as an 8-bit unsigned integer in the range 1-255. The default value is 1. Values of 0 and outside this range MUST result in a REQUEST_ERROR with error code INVALID_FILTER. Lower values indicate higher priority (protected from degradation). When bandwidth is sufficient, all conditional SetIDs remain active. When bandwidth is constrained, lower-priority conditional SetIDs (higher numeric rank) are deactivated first, while higher-priority conditional SetIDs maintain their target allocation.


## Alternative: Reference-Based Design with CONDITIONAL-SET-ASSIGNMENT {#conditional-set-assignment-design}

The inline parameter design described in {{parameter-definition}} embeds all algorithm fields directly in RANGE_FILTER_CONDITION. This produces a self-contained parameter but duplicates the algorithm parameter structure that Sender-Side Track Switching (SSTS) {{SSTS}} defines in its SWITCHING-SET-ASSIGNMENT parameter, which has an identical set of fields (Algorithm ID, Throughput threshold, Set throughput fraction, Activate switching, Set rank).

A more composable design separates the algorithm configuration from the filter binding. Under this design, a single common parameter — CONDITIONAL-SET-ASSIGNMENT — carries the algorithm fields and is shared by both intra-track conditional filtering and inter-track SSTS switching. RANGE_FILTER_CONDITION is then reduced to a binding that maps a Range Filter SetID to a Conditional set ID:

~~~
CONDITIONAL-SET-ASSIGNMENT {
  Conditional set ID (vi64),
  Algorithm ID (vi64),
  Throughput threshold (vi64),
  Set throughput fraction (vi64),
  Activate switching (vi64),
  Set rank (8)
}

RANGE_FILTER_CONDITION {
  Set ID (8),
  Conditional set ID (vi64)
}
~~~

CONDITIONAL-SET-ASSIGNMENT replaces the SWITCHING-SET-ASSIGNMENT parameter currently defined in SSTS. The fields and semantics are identical; only the name and scope change. For SSTS use, the subscriber assigns a subscription to a conditional set by including CONDITIONAL-SET-ASSIGNMENT in SUBSCRIBE or PUBLISH_OK, exactly as it would have used SWITCHING-SET-ASSIGNMENT. For intra-track conditional filtering, the subscriber additionally includes one RANGE_FILTER_CONDITION per SetID to bind each Range Filter group to a Conditional set ID.

This design has the following properties:

- Algorithm parameters are defined once in CONDITIONAL-SET-ASSIGNMENT and shared across use cases. Multiple RANGE_FILTER_CONDITION entries can reference the same Conditional set ID if they share algorithm configuration, or different IDs if they require different thresholds or ranks.
- RANGE_FILTER_CONDITION becomes a lightweight binding with a fixed, minimal wire footprint (one 8-bit SetID and one varint).
- A single relay implementation handles both conditional intra-track filtering and inter-track SSTS switching through the same CONDITIONAL-SET-ASSIGNMENT evaluation logic.
- Future algorithms defined for SSTS are automatically available for conditional intra-track filtering without any changes to this specification.

The trade-off is that the subscriber must include two parameter types — CONDITIONAL-SET-ASSIGNMENT and RANGE_FILTER_CONDITION — rather than one, and the relay must correlate them. This is a minor implementation cost relative to the elimination of duplicated parameter definitions across two specifications.

This document presents both designs. The reference-based design above is RECOMMENDED when SSTS and conditional intra-track filtering are deployed together. The inline design ({{parameter-definition}}) is suitable for deployments that use only intra-track conditional filtering without SSTS.

# Relay Evaluation and Bandwidth Allocation

Most of the text in this section is adapted from Sender-Side Track Switching for Media over QUIC Transport {{SSTS}}, specifically the "Bandwidth Allocation Algorithm" material at <https://github.com/wilaw/moq-transport/blob/patch-1/draft-ietf-moq-transport.md#bandwidth-allocation-allocation-algorithm>.

## General Behavior

With this parameter, the subscriber pre-authorizes the relay to autonomously adjust intra-track forwarding within the subscribed track.

When RANGE_FILTER_CONDITION entries are present and activate switching is non-zero, the relay evaluates each conditionally bound SetID at each permitted switching point (Group boundary) and determines whether that SetID remains active. The relay then applies the standard MOQT OR-combination across all active conditional SetIDs together with any unconditional SetIDs. In other words, RANGE_FILTER_CONDITION changes whether a SetID participates in the OR expression; it does not change the meaning of SetID itself.

When activate switching is 0 for all conditional SetIDs, or no RANGE_FILTER_CONDITION is present, the relay applies the standard MOQT Range Filter OR-combination behavior unchanged.

## Algorithm 0: Throughput-Based Conditional Set Admission

### Relay State

The relay maintains the following state for each subscription that carries one or more RANGE_FILTER_CONDITION parameters with Algorithm ID 0:

- `B_total`: Estimated downstream bandwidth capacity for the subscriber connection. The relay MUST maintain a bandwidth estimate per downstream subscriber. The measurement interval SHOULD be at least the Group duration of the subscribed track. The estimate is obtained from the QUIC congestion control state (e.g., pacing rate, congestion window, smoothed RTT) and MAY be supplemented by external sources. The exact mechanism is implementation-specific.
- `sum_F`: Sum of all Set throughput fraction values across all registered RANGE_FILTER_CONDITION entries for this subscription. Updated when entries are added or removed.
- `set.threshold`: The Throughput threshold for each conditional SetID.
- `set.fraction`: The Set throughput fraction for each conditional SetID.
- `set.rank`: The Set rank for each conditional SetID.
- `active_set_ids`: The set of conditional SetIDs currently admitted into the MOQT OR-combination for this subscription.

### Bandwidth Allocation Algorithm {#bandwidth-allocation}

On a periodic update interval or at a minimum at each Group boundary, the relay executes the following algorithm to determine which conditional SetIDs remain active:

~~~
active_set_ids = {}
reserved_bw = 0
B_available = B_total
for each set in ascending rank order:
  set.target     = B_total x set.fraction / sum_F
  B_available    = max(0, B_total - reserved_bw)
  if set.threshold <= min(set.target, B_available):
    active_set_ids += set.set_id
    reserved_bw += set.threshold
~~~

The relay then forwards Objects that pass the Range Filters associated with any SetID in `active_set_ids`, combined using the standard MOQT OR semantics, together with any unconditional SetIDs. If no conditional SetID satisfies its threshold, the relay forwards only the unconditional SetIDs, if any. If the subscription contains no unconditional SetIDs, the relay MAY forward no Objects; this behavior SHOULD be documented by the relay implementation.

The rank ordering ensures higher-priority conditional SetIDs (lower rank value) are admitted first. The use of `reserved_bw` makes admission monotonic: once a higher-priority conditional SetID has been admitted, lower-priority SetIDs are evaluated only against the bandwidth that remains after honoring those earlier admissions. Lower-priority conditional SetIDs therefore absorb any shortfall by being deactivated from the OR-combination.

### Relay Procedure

When the relay receives a subscription with one or more RANGE_FILTER_CONDITION parameters:

1. Register each conditional SetID and its associated Algorithm parameters.
2. Store `B_total`, `sum_F`, `set.threshold`, `set.fraction`, `set.rank`, and `active_set_ids` as subscription state.
3. If activate switching is non-zero and the number of registered RANGE_FILTER_CONDITION entries for this subscription is >= the activate switching value, begin active SetID admission by applying the bandwidth allocation algorithm ({{bandwidth-allocation}}) at the next Group boundary.

When a REQUEST_UPDATE is received:

- A RANGE_FILTER_CONDITION with Length = 0 removes the conditional state for the referenced SetID.
- A RANGE_FILTER_CONDITION with non-zero Length replaces the entire conditional state for the referenced SetID.
- Conditional SetIDs not referenced in REQUEST_UPDATE remain unchanged.

When the subscription is terminated (PUBLISH_DONE or cancellation), the relay removes all associated RANGE_FILTER_CONDITION state.

# Example: OR-Preserving Conditional Layer Admission

A subscriber receives a scalable video track containing three subgroups: Subgroup 0 (base layer), Subgroup 1 (enhancement layer 1), and Subgroup 2 (enhancement layer 2).

The subscriber defines three SetID groups that remain OR-combined in the normal MOQT sense. The base layer is always included. Conditional evaluation determines which enhancement layers stay active at any given time.

## Step 1: Define the Range Filters

Using the standard MOQT SUBGROUP_FILTER parameter (SetID is 8 bits per MOQT Section 5.1.3):

- SUBGROUP_FILTER (SetID = 0): Range 0-0. Delivers the base layer. This SetID is unconditional.
- SUBGROUP_FILTER (SetID = 1): Range 1-1. Delivers enhancement layer 1.
- SUBGROUP_FILTER (SetID = 2): Range 2-2. Delivers enhancement layer 2.

## Step 2: Bind Conditions with RANGE_FILTER_CONDITION (Algorithm 0)

~~~
RANGE_FILTER_CONDITION (SetID=1):
  Algorithm ID           = 0
  Throughput threshold   = 2000  (kbps)
  Set throughput fraction = 6
  Activate switching     = 2
  Set rank               = 1

RANGE_FILTER_CONDITION (SetID=2):
  Algorithm ID           = 0
  Throughput threshold   = 2500  (kbps)
  Set throughput fraction = 4
  Activate switching     = 2
  Set rank               = 2
~~~

Activate switching = 2 means condition-based admission activates once both conditional RANGE_FILTER_CONDITION entries are registered. sum_F = 6 + 4 = 10.

## Result

The relay applies the bandwidth allocation algorithm at each Group boundary:

- B_total = 7000 kbps: SetID=1 is allocated `7000 * 6 / 10 = 4200`, which satisfies its 2000 kbps threshold. The remaining budget is 5000 kbps. SetID=2 is then allocated `7000 * 4 / 10 = 2800`, which satisfies its 2500 kbps threshold. The relay therefore forwards SetID 0, SetID 1, and SetID 2.

Note: The subscriber should set fractions and thresholds such that, at the expected B_total, each desired layer's threshold is within its allocated fraction. The example above illustrates the mechanics; real deployments may calibrate threshold and fraction values so the intended set of layers remains active at expected bandwidth levels.

In a typical deployment:

- B_total = 7000 kbps: The relay keeps SetID=1 and SetID=2 active. Combined with unconditional SetID=0, subgroups 0, 1, and 2 are forwarded.
- B_total = 4000 kbps: The relay keeps SetID=1 active but not SetID=2. Combined with unconditional SetID=0, subgroups 0 and 1 are forwarded.
- B_total = 1500 kbps: No conditional enhancement SetID is admitted. The relay still forwards the unconditional base layer in SetID=0.

By structuring the SetIDs as independently admitted OR terms evaluated against bandwidth thresholds, the subscriber eliminates round-trip signaling delays while preserving the existing SetID combination semantics.


# Call flow

The following call flows illustrate three workflows:
1. A typical SUBSCRIBE workflow for conditional range-filter evaluation.
2. A publisher-initiated subscription workflow based on SUBSCRIBE_TRACKS and Namespace Prefix Matching.
3. A publisher-initiated subscription workflow where RANGE_FILTER_CONDITION is carried in PUBLISH_OK.

## Typical SUBSCRIBE workflow

~~~
+------------+                                   +-------+
| Subscriber |                                   | Relay |
+------------+                                   +-------+
  |                                              |
  |    SUBSCRIBE (Standard Range Filters,        |
  |               RANGE_FILTER_CONDITIONs)       |
  |--------------------------------------------->|
  |                                              |
  |                  SUBSCRIBE_OK                |
  |<---------------------------------------------|
  |                                              |
  |                              +-----------------------------+
  |                              | 1. Measure B_total          |
  |                              | 2. Evaluate Conditions      |
  |                              | 3. Admit Active SetIDs      |
  |                              +-----------------------------+
  |                                              |
  |    Forward Objects (Active SetIDs)           |
  |<=============================================|
  |                                              |
  ~             Bandwidth Fluctuates             ~
  |                                              |
  |                              +-----------------------------+
  |                              | Group Boundary Re-eval:     |
  |                              | Update active SetIDs        |
  |                              +-----------------------------+
  |                                              |
  |    Forward Objects (Updated Active SetIDs)   |
  |    (w/ PRIOR_SUBGROUP_ID_GAP if skipped)     |
  |<=============================================|
  |                                              |
~~~

* **Initial setup**: The subscriber sends SUBSCRIBE with multiple SetID groups (using standard Range Filters) and binds them using RANGE_FILTER_CONDITION.

* **Continuous evaluation**: The relay autonomously executes the evaluation loop (measuring throughput, comparing thresholds, and admitting or deactivating conditional SetIDs).

* **Dynamic adaptation**: At the next group boundary, the relay can deactivate lower-priority conditional SetIDs and, when subgroups are skipped, uses PRIOR_SUBGROUP_ID_GAP to make intentional omissions explicit.

## Publisher-initiated SUBSCRIBE_TRACKS workflow

The following call flow illustrates a publisher-initiated subscription workflow based on SUBSCRIBE_TRACKS and Namespace Prefix Matching.

~~~
+------------+                        +-------+                            +-----------+
| Subscriber |                        | Relay |                            | Publisher |
+------------+                        +-------+                            +-----------+
  |                                      |                                      |
  | SUBSCRIBE_TRACKS                     |                                      |
  | (Track Namespace Prefix,             |                                      |
  |  optional Range/Conditional Filter)  |                                      |
  |------------------------------------->|                                      |
  |                                      | Namespace Prefix Matching            |
  |                                      | against authorized tracks            |
  |                                      |------------------------------------->|
  |                                      |<-------------------------------------|
  |                                      | Matched Track List                   |
  |                                      |                                      |
  |                                      | PUBLISH (Track A) on new stream      |
  |<=====================================|======================================|
  | PUBLISH_OK (or REQUEST_OK)           |                                      |
  |=====================================>|======================================|
  |                                      | PUBLISH (Track B) on new stream      |
  |<=====================================|======================================|
  | PUBLISH_OK (or REQUEST_OK)           |                                      |
  |=====================================>|======================================|
  |                                      |                                      |
  |<==== Objects for accepted tracks, constrained by active filters ==========> |
  |                                      |                                      |
~~~

* **Discovery and matching**: The subscriber requests a Track Namespace Prefix on a bidirectional stream using SUBSCRIBE_TRACKS. The relay or publisher performs Namespace Prefix Matching across known authorized tracks.

* **Publisher-initiated delivery**: For each matched track, the publisher (directly or through the relay) initiates a new stream and sends PUBLISH. The subscriber accepts each stream with PUBLISH_OK (or REQUEST_OK), establishing the subscription.

* **Filter refinement**: In this workflow, SUBSCRIBE_TRACKS MAY carry optional Range Filter and/or Conditional Filter parameters to limit delivered objects, either statically (fixed ranges) or dynamically (runtime condition evaluation).

## Publisher-initiated workflow with RANGE_FILTER_CONDITION in PUBLISH_OK

The following call flow illustrates a publisher-initiated workflow where the subscriber returns RANGE_FILTER_CONDITION in PUBLISH_OK.

~~~
+------------+                        +-------+                            +-----------+
| Subscriber |                        | Relay |                            | Publisher |
+------------+                        +-------+                            +-----------+
  |                                      |                                      |
  | SUBSCRIBE_TRACKS                     |                                      |
  | (Track Namespace Prefix)             |                                      |
  |------------------------------------->|                                      |
  |                                      | Namespace Prefix Matching            |
  |                                      | against authorized tracks            |
  |                                      |------------------------------------->|
  |                                      |<-------------------------------------|
  |                                      | Matched Track List                   |
  |                                      |                                      |
  |                                      | PUBLISH (Track A) on new stream      |
  |<=====================================|======================================|
  | PUBLISH_OK + RANGE_FILTER_CONDITION  |                                      |
  |=====================================>|======================================|
  |                                      | Apply returned conditions            |
  |                                      | before forwarding objects            |
  |<==== Objects for Track A constrained by returned condition ================>|
  |                                      |                                      |
  |                                      | PUBLISH (Track B) on new stream      |
  |<=====================================|======================================|
  | PUBLISH_OK + RANGE_FILTER_CONDITION  |                                      |
  |=====================================>|======================================|
  |                                      | Apply returned conditions            |
  |                                      | before forwarding objects            |
  |<==== Objects for Track B constrained by returned condition ================>|
  |                                      |                                      |
~~~

* **No filter parameters in SUBSCRIBE_TRACKS**: The subscriber requests only a Track Namespace Prefix.

* **Condition conveyed in acceptance**: For each received PUBLISH stream, the subscriber responds with PUBLISH_OK carrying RANGE_FILTER_CONDITION.

* **Per-track activation**: The relay applies the returned conditions to that accepted stream and forwards only objects that satisfy the resulting active SetID combination.

# Design Considerations

## OR-Preserving Admission Versus Set Replacement

Two high-level designs are possible for conditional intra-track filtering.

In a set-replacement design, the relay interprets conditional SetIDs as mutually exclusive alternatives and selects exactly one active SetID. That model can be convenient for layered examples, but it changes the base MOQT meaning of SetID because SetIDs no longer behave as OR-combined filter groups once conditional logic is enabled.

This document instead uses an OR-preserving admission design. Under this model, conditional logic determines only whether a SetID participates in the existing OR expression at a given switching point. The AND semantics within a SetID and the OR semantics across SetIDs remain unchanged. This preserves compatibility with existing Range Filter processing, avoids introducing a special-case interpretation of SetID, and lets unconditional and conditional SetIDs coexist naturally in the same subscription.

The main trade-off is that subscribers describe adaptation as incremental admission or removal of OR terms, rather than as a single explicit "current SetID". That requires examples and algorithms to be phrased in terms of active SetID combinations. This document prefers that trade-off because it keeps the extension additive and avoids redefining core MOQT Range Filter behavior.

## Alternative: Embedding Conditions in Existing Range Filter Parameters

An alternative design would embed the conditional evaluation fields (Algorithm ID, Throughput threshold, Set throughput fraction, Activate switching, Set rank) directly into the existing Range Filter parameters SUBGROUP_FILTER, OBJECTID_FILTER, PRIORITY_FILTER, OBJECT_PROPERTY_FILTER, and TRACK_PROPERTY_FILTER, rather than introducing a separate RANGE_FILTER_CONDITION parameter.

Under this approach, a subscriber would signal the condition inline with the filter that it governs, for example:

~~~
SUBGROUP_FILTER {
  Type=0x25, Length, [SetID], Range...,
  [Algorithm ID (vi64)],
  [Throughput threshold (vi64)],
  ...
}
~~~

This approach is not adopted for the following reasons.

**SetID is a grouping mechanism, not a per-filter attribute.**
A single SetID typically comprises multiple Range Filter parameters that are AND-combined (e.g., a SUBGROUP_FILTER and an OBJECTID_FILTER with the same SetID). The conditional activation logic applies to the entire SetID group, not to any individual filter within it. Embedding condition fields in each Range Filter parameter would require either repeating the fields identically across every parameter sharing the same SetID, or designating one parameter as the canonical carrier while others are implicitly governed by it. Both approaches introduce ambiguity and implementation complexity without benefit.

**Modifying base MOQT parameters raises the change bar.**
SUBGROUP_FILTER and related parameters are defined and registered in the MOQT base specification ({{!I-D.ietf-moq-transport}}, Section 15.7). Extending their wire format would require a change to or an update of that specification rather than a new parameter registration. A standalone parameter type, by contrast, is a purely additive change.

**Separation of concerns is architecturally preferable.**
Range Filter parameters define the *content selection criterion* (which subgroup IDs, object IDs, priorities, or property values pass). The RANGE_FILTER_CONDITION parameter defines the *activation condition* (under what network state the SetID is active). Keeping these concerns in separate parameters preserves the ability to use Range Filters independently of any conditional logic, and allows future algorithms to be introduced without touching the base filter format.

For these reasons, RANGE_FILTER_CONDITION is defined as a standalone parameter that references an existing SetID by value.

## Handling Filtered Subgroups: Prior Subgroup ID Gap

When a relay dynamically filters out a subgroup due to network conditions, a subscriber receiving the stream might notice a numerical leap in Group IDs or Object IDs. To explicitly communicate that these missing objects were intentionally skipped by a conditional filter rather than lost to network congestion, we propose a new PRIOR_SUBGROUP_ID_GAP Object Property.

*   **Prior Subgroup ID Gap (Property Type: 0x3F):** A variable-length integer containing the number of Subgroups prior to the current Subgroup ID (within the current Group) that do not and will never exist because they were conditionally filtered out.

If a relay drops Subgroup 1 but continues forwarding Subgroup 0 and Subgroup 2, it injects PRIOR_SUBGROUP_ID_GAP = 1 into the first Object of Subgroup 2. Relays MUST be permitted to modify this specific gap property to support dynamic conditional filtering.

## Extension negotiation

Per Section 10.3 of {{!I-D.ietf-moq-transport}}, the SETUP message allows
endpoints to agree on the initial configuration before any other control
messages are exchanged. Extensions that modify control message semantics,
such as RANGE_FILTER_CONDITION, MUST be negotiated through Setup Options
before use.

This document defines two new Setup Options: RANGE_FILTER_CONDITION and
MAX_CONDITIONAL_FILTERS.

RANGE_FILTER_CONDITION (Type 0x09):
: Declares support for the RANGE_FILTER_CONDITION message parameter defined
  in {{parameter-definition}}. The value is a variable-length integer
  identifying the version of the conditional filtering algorithm set the
  endpoint is willing to use. The value indicates the highest Algorithm ID
  the endpoint is capable of evaluating. An endpoint that includes this
  option can process RANGE_FILTER_CONDITION parameters for any Algorithm ID
  up to and including the advertised value. If not present, the endpoint does
  not support this extension.

MAX_CONDITIONAL_FILTERS (Type 0x0A):
: Limits the peer's total number of RANGE_FILTER_CONDITION parameters allowed
  concurrently for a given subscription or fetch. The default value is 0, so
  if not specified, the peer MUST NOT send any RANGE_FILTER_CONDITION
  parameters. If this limit is exceeded, an endpoint MUST reject this with
  REQUEST_ERROR with error code INVALID_FILTER.

An endpoint that wishes to send RANGE_FILTER_CONDITION parameters MUST verify
that the peer's SETUP message includes a RANGE_FILTER_CONDITION Setup Option
with a value greater than or equal to the Algorithm ID it intends to use, and
a MAX_CONDITIONAL_FILTERS value greater than zero.

Because RANGE_FILTER_CONDITION binds to Range Filter SetIDs, an endpoint that
sends RANGE_FILTER_CONDITION MUST also ensure that the MAX_FILTER_RANGES Setup
Option (Section 10.3.1.6 of {{!I-D.ietf-moq-transport}}) has been negotiated
with a non-zero value; otherwise Range Filters themselves would not be
permitted and the referenced SetIDs could not be established.

Endpoints that do not support this extension will not include the
RANGE_FILTER_CONDITION Setup Option in their SETUP message. As required by
Section 10.3 of {{!I-D.ietf-moq-transport}}, endpoints MUST ignore unknown
Setup Options, so a peer that receives these options from an endpoint that
does not recognize them simply ignores them and does not send
RANGE_FILTER_CONDITION parameters in that session.

# Security Considerations

This extension relies on the existing security framework of MOQT. Conditional filter evaluation must be computationally lightweight to prevent denial-of-service attacks against relays. A malicious subscriber could attempt to overwhelm a relay with highly complex RANGE_FILTER_CONDITION thresholds or excessive conditional SetIDs. Relays SHOULD implement rate limiting on the maximum number of conditionally bound SetIDs a subscriber can declare in a single session.

# IANA Considerations

This document requests the following registrations:
*  A new Setup Option Type for RANGE_FILTER_CONDITION (suggested value: 0x09) in the "MOQ Setup Options" registry (Section 15.4 of {{!I-D.ietf-moq-transport}}), with Specification Required policy.
*  A new Setup Option Type for MAX_CONDITIONAL_FILTERS (suggested value: 0x0A) in the "MOQ Setup Options" registry (Section 15.4 of {{!I-D.ietf-moq-transport}}), with Specification Required policy.
*  A new Object Property Type for PRIOR_SUBGROUP_ID_GAP (suggested value: 0x3F) in the "MOQ Properties" registry.

--- back

# Acknowledgments
{:numbered="false"}

TODO
