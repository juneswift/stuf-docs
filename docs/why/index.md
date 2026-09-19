!!! warning "Early-stage software"
    STUF is not production-ready. It has not undergone independent security review and should not be used in production update paths without one. See the [GitHub repo](https://github.com/juneswift/stuf) for current status.

# Why STUF?

Software and firmware updates increasingly touch the physical world — industrial controllers, automotive ECUs, medical devices, building infrastructure, and cloud-connected fleets. The security of the update mechanism is now a first-order concern, not an afterthought.

## Secure Updates are Hard

Conventional update systems can inherit too much trust from the environment they run in: the repository, mirror, transport, local filesystem, or signing workflow. In practice, real-world incidents have shown that each part of that chain can fail.

STUF starts from the opposite assumption: important parts of the update infrastructure may be compromised, unavailable, stale, or operationally constrained. The verification layer should remain explicit about what it has proven and what it has not.

The following are selected risks that stuf can address, not an exhaustive threat model:

| Risk | Example failure mode | STUF concern |
|---|---|---|
| Repository compromise | attacker publishes malicious update material | authorization verification |
| Mirror or staging compromise | stale or substituted artifacts | artifact hash and length checks |
| Key compromise | attacker signs with limited authority | authority separation and threshold rules |
| Device constraint | verifier cannot parse large metadata or manifests | profile-specific limits |
| Fleet staging | staging layer becomes operational bottleneck | staged but untrusted distribution |

## Where STUF fits

TUF is the foundation, not the problem. The specification is mature, well-studied, and deliberately flexible. Existing implementations such as aws-tough, go-tuf, and python-tuf are valuable parts of that ecosystem.

STUF explores a different implementation boundary: a small Rust trust kernel that can be shared across constrained embedded targets, fleet-staging systems, and cloud release pipelines, while keeping crypto, transport, encoding, and profile assumptions explicit.

## Design goals

STUF's design is built around five principles.

**Small trust kernel.** The code that makes security decisions should be small enough that a single reviewer can hold it in their head. Complexity belongs in profiles and environment bindings, not in the kernel.

**Explicit trust transitions.** Every step from untrusted bytes to verified artifact should be visible in types and code paths. There is no implicit trust promotion.

```rust
// An unverified value and a verified value are different types.
// You cannot pass one where the other is expected.
let unverified: Unverified<Metadata> = fetch_metadata();
let verified: Verified<Metadata> = verifier.verify(unverified)?;
```

**Profile-driven assumptions.** Different environments need genuinely different constraints. Embedded profiles can use bounded buffers and static trust anchors. Cloud profiles can support audit logging, policy hooks, and CI/CD integration. The goal is for both to share the same trust kernel while making their assumptions explicit.

**Fail closed.** If verification cannot prove authorization, the artifact is not accepted.

**Composable by design.** Feature flags and crate boundaries are not just conveniences — they are part of the interface. They keep crypto, transport, encoding, and environment choices explicit, and help minimize what compiles into a given binary.

## Threat model

For any protocol profile STUF implements, basic specification compliance is the floor, not the ceiling. A profile should reject inputs that satisfy a narrow reading of a protocol but violate the profile's safety assumptions, resource limits, trust boundaries, threat model, encoding and parsing rules, or fail-closed behavior.

