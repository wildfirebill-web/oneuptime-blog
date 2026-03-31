# How to Understand Dapr Component Certification Tiers

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Component, Certification, Stability, Production

Description: Learn the difference between Dapr component certification tiers - Alpha, Beta, and Stable - and what each tier means for production use and support guarantees.

---

Not all Dapr components are equally mature. Dapr uses a certification tier system to communicate the production-readiness, test coverage, and community support level of each component implementation.

## The Three Certification Tiers

### Alpha

Alpha components are newly contributed or experimental. They:

- May have incomplete feature implementation
- Have minimal automated test coverage
- Can change spec fields or behavior between releases
- Are not recommended for production use
- May be removed if maintenance is abandoned

### Beta

Beta components have passed basic quality gates and are approaching stable. They:

- Have functional automated tests including conformance tests
- Are reasonably stable but may still have minor API changes
- Can be used in non-critical production workloads with caution
- Have at least one maintainer actively responding to issues

### Stable

Stable components are production-ready. They:

- Pass the full Dapr component conformance test suite
- Have comprehensive documentation including all metadata fields
- Guarantee no breaking spec changes without deprecation notices
- Have multiple maintainers and active community support
- Are safe for production use in business-critical systems

## Finding Component Certification Status

The certification status is listed on the component reference page in the Dapr docs. Look for the certification badge at the top of each component page.

You can also browse the component registry:

```bash
# Browse components in the Dapr components-contrib repository
git clone https://github.com/dapr/components-contrib.git
ls components-contrib/state/
```

The README for each component folder indicates its certification tier.

## Conformance Tests

Stable components pass the Dapr conformance test suite, which validates:

```bash
# Run conformance tests locally (from components-contrib)
cd components-contrib
go test ./tests/conformance/... -tags=<component>
```

Conformance tests verify behaviors like:
- State store CRUD operations and ETags
- Pub/sub publish/subscribe message delivery
- Binding input/output behavior
- Secret store read operations

## Choosing Components for Production

When evaluating a component for production use:

```yaml
# Check these before committing to a component:
# 1. Certification tier (Stable preferred)
# 2. Last commit date on components-contrib GitHub
# 3. Open issues and pull request activity
# 4. Whether the backing service is self-hosted or cloud-managed
```

For example, Redis state store (Stable) vs. an Alpha NoSQL state store:

```yaml
# Stable - safe for production
spec:
  type: state.redis
  version: v1

# Alpha - evaluate carefully
spec:
  type: state.cassandra
  version: v1
```

## Summary

Dapr components use three certification tiers: Alpha (experimental), Beta (approaching stable), and Stable (production-ready and conformance-tested). Always check the certification tier before using a component in production, and prefer Stable components for business-critical workloads to avoid unexpected breaking changes.
