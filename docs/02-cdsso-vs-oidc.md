# CDSSO vs. OIDC

## Why migrate

CDSSO and eCSSO were deprecated for new installations in [ISVA 10.0.2](https://www.ibm.com/support/pages/cdsso-deprecated-10020),
and are completely removed as of IVIA 11. There is no automated migration
tool — the two mechanisms are architecturally different (see
[01-concepts.md](01-concepts.md)), so this is a redesign, not an upgrade.

## Do you actually need OIDC?

Not every CDSSO deployment needs a full OIDC redesign. CDSSO solved one
specific problem: sharing an authenticated session across applications in
**different DNS domains**. If your reverse proxy instances are all under a
single DNS domain (e.g. everything under `*.example.com`), DSC or Redis
alone can give you seamless session sharing after the initial login,
without needing OIDC at all.

You need OIDC (or another federated SSO mechanism) specifically when your
applications span **separate DNS domains** — a session cookie from one
domain can't be read by another, which is the exact gap CDSSO was closing.

## Architecture comparison

| | CDSSO | OIDC |
|---|---|---|
| Topology | Peer-to-peer — each domain pushes an encrypted token to the next | Hub-and-spoke — one OP, consumed by one or more RPs |
| Kickoff | `GET /pkmscdsso?<target-url>` | `GET /pkmsoidc?iss=<op-id>&Target=<encoded-target-url>` |
| Session sharing across domains | Built into CDSSO itself | Requires OIDC *plus* DSC/Redis — OIDC propagates identity across the domain boundary, DSC/Redis shares the resulting session within each domain |

## The kickoff URL swap

The direct replacement for `/pkmscdsso?<target-url>` is:

```
GET /pkmsoidc?iss=<op-id>&Target=<encoded-target-url>
```

`iss` selects which OP to use — it can be omitted if the RP has a
`default-op` configured and only ever talks to one OP.

**Behavioral difference to plan for:** CDSSO's `/pkmscdsso` was a
transparent redirect the browser was pushed through automatically. The
WebSEAL OIDC RP presents `/pkmsoidc` as a login *mechanism* — a button on
the login page, or a link you build — rather than something invoked
silently. If your CDSSO deployment relied on invisible redirection between
peers, plan to explicitly link to `/pkmsoidc` (see
[04-rp-setup.md](04-rp-setup.md) for the exact link format) rather than
assuming it fires automatically.

## If you need closer CDSSO-like push behavior

If the login-mechanism behavior above doesn't fit — you need something
closer to a transparent push — the **Federation Module OIDC RP** kickoff is
a closer behavioral match:

```
GET /sps/oidc/rp/<federation-name>/kickoff/<partner-name>?Target=<encoded-target-url>
```

This requires the Federation module rather than (or alongside) AAC. This
guide covers the AAC + Web Reverse Proxy OIDC RP path; if you need the
Federation path instead, the OP-side setup differs but the underlying
OIDC concepts in [01-concepts.md](01-concepts.md) still apply.

## Multi-instance / load-balanced reverse proxies

If your RP has multiple reverse proxy instances behind a non-sticky load
balancer, the in-progress OIDC handshake state needs to be shared across
instances (it lives in the local session cache by default). Add DSC or
Redis, plus:

```
[session]
create-unauth-sessions = yes
```
