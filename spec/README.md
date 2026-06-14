# Siglark protocol specification

This folder holds the open specification for verifiable action provenance for autonomous physical agents.

## Current draft

- [draft-siglark-physical-agent-provenance-00.md](draft-siglark-physical-agent-provenance-00.md): an IETF-style Internet-Draft. It specifies three properties for autonomous physical agents (robots, autonomous vehicles, drones, industrial machinery):
  1. Per action provenance: each consequential action is signed at the source with a key bound to the agent, so it is attributable and non repudiable.
  2. Attenuated delegation: authority can be delegated agent to agent and can only narrow down the chain.
  3. Offline counterparty verifiability: a third party can verify what an agent did on their own equipment, using inclusion and consistency proofs, without trusting the operator.

## Why this exists

A primary-source survey of robotics security (DDS-Security / ROS 2 / SROS2, MAVLink), device identity (IEEE 802.1AR, FIDO Device Onboard, SPIFFE), tamper-evident logging (RFC 9162 Certificate Transparency, Merkle-tree verifiable logs), and capability tokens (macaroons, biscuit, UCAN) found that the primitives needed for physical-agent action provenance exist only as disconnected building blocks in unrelated domains. No surveyed IETF, ISO, IEC, IEEE, or OMG working group charter covers the integrated space. The draft cites the specific gap each existing standard leaves.

## Status and intent

This is an early draft (revision 00). The intent is to develop it in the open, interoperate with adjacent emerging work (agent audit trails, COSE and supply-chain transparency), and pursue a standards track when the design has been validated with implementers.

## Converting to IETF xml2rfc format

The draft is written in the kramdown-rfc markdown dialect. To produce the canonical IETF formats:

```bash
# install the tooling (one time)
gem install kramdown-rfc2629
pip install xml2rfc

# generate xml then txt/html
kramdown-rfc draft-siglark-physical-agent-provenance-00.md > draft-siglark-physical-agent-provenance-00.xml
xml2rfc draft-siglark-physical-agent-provenance-00.xml --text --html
```

## Contributing

Issues and proposals are welcome. The protocol is meant to be open and reviewable, not a single-vendor black box.
