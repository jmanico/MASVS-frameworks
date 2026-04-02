# Requirements Style Guide

This guide defines how requirements in this repository should be written so they remain auditable, consistent, and easy to maintain.

## Purpose

Use this guide for both Android and iOS chapters.

Goals:

- distinguish normative requirements from explanatory guidance
- keep requirements testable
- scope version-specific behavior explicitly
- avoid mixing platform controls with policy or compliance obligations without clear labeling

## Required Chapter Structure

Each control-group chapter should keep this high-level shape:

1. Overview
2. Control sections
3. Training-aligned requirements

The internal formatting of training-aligned requirements should be normalized over time, but the content must still remain clearly separable from the main requirements.

## Main Requirement Format

Each main sub-requirement should contain:

1. Stable requirement ID
2. Short title
3. Normative requirement text
4. Rationale when the requirement is not self-evident
5. Platform references where they materially improve verification or implementation clarity

Preferred pattern:

```md
#### MASVS-AREA-X.Y - Short Title

Normative requirement text.

**Rationale:** Why this matters.

**Platform References:**
- API, framework, entitlement, manifest key, or vendor doc anchor
```

## Training-Aligned Requirement Format

Training-aligned requirements must supplement the main requirements, not redefine the control taxonomy.

Rules:

- keep a stable training ID such as `AREA-ANDROID-X.Y` or `AREA-IOS-X.Y`
- include one normative statement
- include one testable verification statement
- do not silently change the meaning of `MASVS-*` control IDs
- if grouped by implementation topic instead of control ID, say so explicitly

Preferred pattern:

```md
#### AREA-PLATFORM-X.Y: Short Title

Normative requirement text.

**Testable:** Concrete verification approach.
```

Tables are acceptable when they remain readable and each row still contains:

- stable ID
- requirement
- verification

## Normative Language

Use normative terms intentionally:

- `MUST`: required baseline behavior
- `SHOULD`: recommended default, but exceptions may be valid
- `MAY`: optional technique

Use `MUST` only when all of the following are true:

- the requirement is broadly applicable
- it is realistically implementable
- it is testable
- exceptions are rare enough that they should be treated as deviations, not normal cases

Use `SHOULD` when:

- the pattern is preferred but not universal
- platform, product, or operational constraints may justify exceptions
- the requirement is best treated as default guidance rather than a flat pass/fail rule

Use `MAY` for optional hardening or situational techniques.

Avoid vague pseudo-normative phrasing such as:

- "make sure"
- "best practice" without a requirement statement
- "consider" when a stronger or weaker term is actually intended

## Testability Rules

Every normative requirement should be verifiable by:

- static review
- dynamic testing
- configuration inspection
- server-side validation

Weak verification patterns to avoid:

- "verify awareness"
- "confirm adoption plan"
- "ensure strong security"

Prefer concrete verification statements such as:

- inspect a manifest, plist, entitlement, or config file
- capture network traffic and verify a condition
- exercise a user flow and confirm a security boundary
- inspect on-device storage or Keychain / Keystore behavior

## Version-Specific Guidance

When behavior depends on OS version:

- name the version explicitly
- scope the claim narrowly
- avoid projecting future platform behavior as if it were already stable baseline guidance

Preferred pattern:

```md
On Android 16+ ...
On iOS 17+ ...
```

If the behavior is future-facing or roadmap-oriented, label it as guidance rather than baseline requirement text.

## Baseline vs High-Assurance Scope

Do not present high-assurance or commercial-hardening techniques as universal baseline requirements unless that is truly intended.

Examples that often need explicit scoping:

- RASP integration
- commercial obfuscation
- runtime self-integrity checks
- aggressive anti-instrumentation
- certificate pinning
- post-quantum transition planning

Preferred scoping phrases:

- "For high-risk apps ..."
- "For high-value apps ..."
- "Where the threat model justifies it ..."

## Platform vs Policy vs Compliance

Where a requirement is driven by:

- platform security behavior
- app-store policy
- legal or regulatory obligation

say so clearly.

Do not blur these into a single undifferentiated platform requirement.

Preferred pattern:

```md
Where App Store policy or applicable regulation requires it ...
Where the product relies on consent-based collection ...
```

## References

References should be:

- platform-primary where possible
- directly relevant
- used to support specific claims, not to pad the document

When a claim is likely to change across releases, prefer current vendor documentation over memory or secondary summaries.

## Normalization Priorities

When normalizing an existing chapter, fix items in this order:

1. internal contradictions
2. stale or unsupported platform claims
3. over-broad `MUST` statements
4. weak or non-testable verification text
5. formatting consistency
