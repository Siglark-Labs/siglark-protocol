---
title: "Verifiable Action Provenance for Autonomous Physical Agents"
abbrev: "Physical Agent Provenance"
docname: draft-siglark-physical-agent-provenance-00
category: info
ipr: trust200902
area: Security
workgroup: Independent Submission
keyword:
  - robotics
  - provenance
  - attestation
  - delegation
  - audit
stand_alone: yes
pi: [toc, sortrefs, symrefs]
author:
  - ins: Siglark Labs
    name: Siglark Labs
    organization: Siglark, a Vorim AI Labs product
    email: team@vorim.ai
--- abstract

Autonomous physical agents, including robots, autonomous vehicles, drones, and
industrial machinery, take consequential and often irreversible actions in the
physical world. Existing standards secure the transport between these systems,
prove a device's identity at onboarding, or provide tamper evident logs for
public certificates, but no existing standard binds a non repudiable
cryptographic signature to each physical action, allows scoped authority to be
delegated and narrowed between agents, and produces an audit trail that an
independent party can verify offline without trusting the operator. This
document specifies a protocol that provides these three properties together. It
defines a per action signing envelope, an attenuated delegation token, and an
append only verifiable log, so that what a physical agent did, on whose
authority, and the integrity of the record can be checked by a counterparty on
their own equipment.

--- middle

# Introduction

Physical autonomy is moving from research into deployment. When an autonomous
machine takes an action with real world consequences, four questions follow:
which agent took the action, under what authority, what exactly did it do, and
can the record be trusted after the fact. Today these questions are difficult to
answer in a way that holds up to an independent reviewer such as a regulator, an
insurer, or a court.

This is not only an engineering concern. Regulation is beginning to require
tamper evidence for the actions of machines. The EU Machinery Regulation
2023/1230, which applies from 20 January 2027, requires machinery to collect
evidence of legitimate or illegitimate intervention in its software and to keep
a tracing log. EU Delegated Regulation 2024/2220 mandates tamper protected event
data recorders on new heavy vehicle types from 7 January 2026. UN Regulations
R155 and R156 make auditable software management a type approval prerequisite
across more than sixty countries. The obligation to produce a trustworthy record
of what a machine did is dated and real.

## The gap in existing work

A survey of the relevant standards, summarised in {{existing-work}}, finds that
the primitives needed to answer these questions exist, but only as disconnected
building blocks in unrelated domains. Transport security in robotics protects
messages in flight with symmetric keys, which cannot produce a record a third
party can verify. Device identity standards stop at authentication and
onboarding. Tamper evident logging is standardised, but for public TLS
certificates rather than arbitrary machine actions. No single specification, and
no surveyed working group, unifies per action signing, attenuated delegation,
and offline counterparty verifiable audit for autonomous physical agents.

This document specifies a protocol to close that gap.

## Goals

The protocol provides three properties:

1. Per action provenance: each consequential action by a physical agent is
   signed at the source with a key bound to that agent, so the action is
   attributable and non repudiable.
2. Attenuated delegation: authority can be granted from one agent to another, or
   from a human to a machine, such that the delegated authority is always a
   subset of the granting authority and can only narrow as it passes down a
   chain.
3. Offline counterparty verifiability: the record of what an agent did can be
   verified by an independent party on their own equipment, using inclusion and
   consistency proofs, without contacting or trusting the operator.

## Non goals

This document does not define a transport. It is intended to run over existing
robotics and vehicle transports. It does not replace device identity standards;
it binds to them. It does not mandate a specific hardware root of trust, though
it supports binding a signing key to one.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

Agent: an autonomous or semi autonomous physical system that takes actions,
including a robot, autonomous vehicle, drone, or item of industrial machinery.

Action: a discrete, consequential operation performed by an agent that is
recorded for provenance, for example moving a load, changing a trajectory, or
actuating an end effector.

Authority: a set of permissions describing what an agent is allowed to do,
expressed as caveats that constrain scope.

Capability token: a signed token that conveys authority and that can be
attenuated by adding caveats.

Verifier: an independent party that checks signatures, authority, and log
integrity. The verifier need not be the operator and need not be online.

# Architecture Overview

The protocol has three components.

1. An agent holds a long lived identity key, bound where possible to a hardware
   root of trust. This key is established through an existing device identity
   mechanism (see {{binding}}). The agent signs each action with this key or
   with a short lived key chained to it.

2. Authority is carried in capability tokens. A controller issues a token to an
   agent describing what it may do. An agent may delegate a strictly attenuated
   token to another agent. Each action references the token under which it was
   taken.

3. Each signed action is appended to a verifiable log structured as a Merkle
   tree. The log produces inclusion proofs (an action is in the log) and
   consistency proofs (a later log contains everything in an earlier one, in the
   same order). These proofs allow a verifier to confirm that the record has not
   been altered, reordered, or truncated, without trusting the log operator.

~~~
   controller                  agent A                 agent B
       |  issue capability         |                       |
       |-------------------------->|                       |
       |                           |  attenuated delegate  |
       |                           |---------------------->|
       |                           |                       | take action
       |                           |                       |  sign + append
       |                           |                       v
       |                     [ verifiable append only log ]
       |                                   |
       |                                   |  inclusion + consistency proofs
       v                                   v
                              independent verifier (offline)
~~~

# Agent Identity and Key Binding {#binding}

Each agent MUST have an identity key pair. Implementations SHOULD use Ed25519.
The public key is the agent's stable identifier within a trust domain.

The identity key SHOULD be bound to a hardware root of trust where the platform
provides one. The binding MAY be expressed by an existing device identity
credential, for example an IEEE 802.1AR DevID certificate or a TPM or DICE based
attestation. This document does not define a new device identity mechanism. It
treats the device identity as the root from which action signing keys derive
their trust.

An agent MAY use short lived signing keys for individual sessions or shifts,
provided each short lived key is itself signed by the identity key, so that a
verifier can chain an action back to the agent identity.

# Capability Tokens and Attenuated Delegation

Authority is expressed as a capability token. A token contains an issuer, a
subject (the agent the authority is granted to), a set of caveats that constrain
what may be done, and a signature.

A caveat is a predicate that narrows authority, for example a zone restriction,
a time window, an allowed action type, or a rate limit. Authority is the
intersection of all caveats on the token and its delegation chain.

Delegation MUST be attenuating. When agent A delegates to agent B, B's token
carries all of A's caveats plus any additional caveats A adds. A verifier
computing B's effective authority takes the intersection across the whole chain.
At no point can a delegated token grant more than the token it was derived from.
This mirrors the attenuation model of capability tokens such as macaroons and
biscuit, grounded here in physical agent identity.

A verifier checking an action MUST confirm that the action falls within the
effective authority of the token chain referenced by the action, and that every
token in the chain is validly signed by its issuer back to a trusted root.

# Action Signing Envelope

Each recorded action is wrapped in a signing envelope. The envelope binds the
action to the agent, the authority, and the log position.

An envelope contains:

- the agent identifier (the signing public key or a reference to it);
- a reference to the capability token chain under which the action was taken;
- the action descriptor (a typed, canonical representation of what was done);
- a monotonic sequence number and a timestamp;
- the hash of the previous envelope from this agent, forming a per agent hash
  chain;
- a signature over all of the above.

Signing at the source, with an asymmetric key, is what distinguishes this
protocol from transport security. A symmetric message authentication code, as
used by DDS Security and MAVLink signing, can be verified only by a holder of
the shared key and so cannot provide non repudiation or third party
verification. An asymmetric signature over the action can be verified by anyone
holding the agent's public key, including a party the operator does not control.

# Verifiable Log

Signed envelopes are appended to a verifiable log. The log is a Merkle tree in
which each appended envelope is a leaf. The log publishes a signed tree head.

The log MUST support:

- inclusion proofs, allowing a verifier to confirm that a specific action is
  recorded in the log corresponding to a given tree head;
- consistency proofs, allowing a verifier to confirm that a later tree head
  extends an earlier one without altering or removing earlier entries.

These mechanisms are those of standardised verifiable logs (see {{RFC9162}}).
This document applies them to physical agent action records rather than to
certificates. Because verification rests on the tree head and the proofs, a
verifier does not need to trust the operator of the log, and can verify offline
given the tree head and the relevant proofs.

# Offline Verification

A verifier checking what an agent did performs the following steps, all of which
can be done without contacting the operator:

1. Verify the agent's identity key against the trusted device identity root.
2. Verify each capability token in the chain and compute effective authority.
3. Verify the action signature in the envelope.
4. Confirm the action falls within effective authority.
5. Verify the per agent hash chain is intact across the envelopes of interest.
6. Verify inclusion of the envelopes in the log against a signed tree head, and
   verify consistency between tree heads if more than one is held.

If all steps pass, the verifier has established, on its own equipment, which
agent took the action, under what authority, what the action was, and that the
record has not been altered.

# Security Considerations

The protocol provides non repudiation of actions to the extent that an agent's
identity key is protected. Binding the key to a hardware root of trust raises the
cost of key extraction and is RECOMMENDED for agents whose actions carry safety
or legal weight.

The protocol detects tampering with the record; it does not prevent an agent from
taking an unauthorised physical action. It ensures that such an action, if taken,
is either signed and therefore attributable, or absent from a verifiable log and
therefore detectable as a gap by sequence number.

Time is a known difficulty for intermittently connected agents. Timestamps in
envelopes SHOULD be treated as agent asserted and corroborated against log entry
order and any available trusted time source.

Revocation of compromised keys and tokens is required and is expected to use
short lived keys and tokens with bounded validity, so that a verifier can reject
authority that has expired.

# IANA Considerations

This document has no IANA actions in this revision. A future revision is expected
to request registries for action descriptor types and caveat types.

# Existing Work and the Gap {#existing-work}

This section summarises the surveyed standards and the specific gap each leaves,
based on primary sources.

Robotics and drone transport security. OMG DDS Security, used by ROS 2 and
SROS2, secures messages at the transport layer with symmetric session keys, using
AES GCM for encryption and AES GMAC for authentication, scoped to a security
perimeter. MAVLink message signing uses a shared 32 byte secret and a truncated
SHA 256 message authentication code. Both protect messages in transit between
parties that share a key. Neither produces a non repudiable, third party
verifiable record of an action, neither offers attenuated delegation, and the
DDS Security logging plugin, which is optional and not implemented by ROS 2 and
SROS2, provides no tamper evident chaining.

Device identity. IEEE 802.1AR DevID, FIDO Device Onboard, and SPIFFE establish a
device or workload identity for authentication, onboarding, or provisioning. Their
scope stops at credential establishment. Peer reviewed analysis of SPIFFE notes
its identity document has no native support for delegation, attenuation, or
traceability, and that its scope is software workloads. FIDO Device Onboard
becomes dormant after onboarding, and its delegation feature is a provisioning
time mechanism rather than runtime attenuation of operational authority.

Tamper evident logging. RFC 9162 Certificate Transparency and the underlying
Merkle tree verifiable data structures provide inclusion and consistency proofs
that can be verified without trusting the log operator. The standardised scope is
public TLS certificates. The structures are domain agnostic, which is why this
protocol adopts them, but they carry no physical agent or per action semantics on
their own.

Capability tokens. Macaroons, biscuit, and UCAN provide attenuable authority in
software and web contexts. This protocol adopts their attenuation model and
grounds it in physical agent identity and action provenance, which those tokens
do not themselves address.

Adjacent and emerging work. The IETF has early activity on agent audit trails and
on supply chain transparency using COSE and verifiable logs. These are adjacent.
Physical agent action provenance, combining per action signing, attenuated
delegation, and offline counterparty verifiable audit for robots, vehicles,
drones, and industrial machinery, was not found in the charter of any surveyed
IETF, ISO, IEC, IEEE, or OMG working group. This protocol is positioned to fill
that intersection and to interoperate with the adjacent work rather than to
duplicate it.

--- back

# Acknowledgements

This draft was informed by a primary source survey of robotics security,
device identity, verifiable logging, and capability token standards. The
conclusion that the integrated space is open is an argument from the absence of a
unifying specification across the surveyed bodies, not a claim that every working
group charter was exhaustively enumerated.
