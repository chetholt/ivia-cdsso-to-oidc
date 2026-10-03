# IVIA CDSSO → OIDC Migration Guide

A field-tested guide for replacing IBM Verify Identity Access (IVIA) CDSSO with
AAC Module + Web Reverse Proxy OIDC Relying Party authentication.

CDSSO and eCSSO were deprecated for new installations in
[ISVA 10.0.2](https://www.ibm.com/support/pages/cdsso-deprecated-10020),
and are completely removed as of IVIA 11, with no automated migration path
— the two mechanisms are architecturally different (peer-to-peer token
passing vs. hub-and-spoke OIDC). This repo
documents a working reference implementation, built and debugged end-to-end
in a lab environment, along with every real issue hit along the way and how
each was diagnosed and fixed.

## Who this is for

Anyone running IVIA WebSEAL reverse proxies that currently use CDSSO for
cross-domain SSO between applications, and who needs to move to OIDC using
the AAC module rather than the Federation module or IBM Application Gateway
RP paths.

## The one thing to know before you start

**The OIDC OP and OIDC RP must run on separate Web Reverse Proxy instances**
— even in the simplest possible setup with only one application. Collapsing
OP and RP onto the same reverse proxy instance causes a session cookie
collision between the OP's own login challenge and the RP's pending
transaction state, which manifests as confusing, hard-to-diagnose failures
(session/token correlation errors, silent redirect loss) well downstream of
the actual cause. See [05-topology.md](docs/05-topology.md) for the full
explanation — this is the single most important decision in the whole setup
and is easy to get wrong if you're starting from "let's keep this simple."

## Contents

1. [Concepts](docs/01-concepts.md) — what OIDC is, and how the OP, RP, and
   tokens fit together
2. [CDSSO vs. OIDC](docs/02-cdsso-vs-oidc.md) — why this migration, what
   changes conceptually, and the kickoff-URL equivalent
3. [Setting up the OP](docs/03-op-setup.md) — AAC module, the OAuth/OIDC
   Provider Configuration wizard, the API Protection/OIDC definition, and
   client registration
4. [Setting up an RP](docs/04-rp-setup.md) — the `[oidc]` stanza, linking
   guidance, and everything a pure RP instance does and does not need (no
   `login.html` changes required)
5. [Topology: why OP and RP need separate instances](docs/05-topology.md) —
   the cookie collision explained, with the evidence
6. [Troubleshooting](docs/06-troubleshooting.md) — symptom → cause → fix,
   covering every real issue hit while building this out

## Example configs

- [`examples/webseald-oidc-stanza.conf`](examples/webseald-oidc-stanza.conf)
  — a fully annotated, working `[oidc]`/`[oidc:default]` stanza for an RP
  instance

## Lab environment this was built against

- IBM Verify Identity Access, single-appliance lab deployment
- One shared AAC runtime, two Web Reverse Proxy instances:
  `webseal-op` (OIDC Provider) and `webseal-rp1` (OIDC Relying Party)
- Real-world deployments replacing multi-domain CDSSO would have one
  `webseal-op` and one `webseal-rpN` per application domain that previously
  participated in CDSSO

## Status

This guide reflects a verified, working end-to-end flow (unauthenticated
request → OIDC login → landing back on the originally-requested resource)
as of the date in the most recent commit. Screenshots and version-specific
UI labels may drift across IVIA point releases — if something on your LMI
doesn't match what's described here, treat this guide's underlying
*mechanism* explanations as more durable than the exact screen/field names.
