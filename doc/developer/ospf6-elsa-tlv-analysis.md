# Post-Mortem: OSPFv3 E-LSA Implementation (RFC 8362)

## Summary

Three PRs attempted to add RFC 8362 Extended LSA support to FRR's ospf6d over 8 months. The effort ended without the feature being merged. The core tension: maintainers were concerned about callback patterns affecting code maintainability; the contributor was concerned about code duplication. Neither side found the other's approach acceptable.

**PRs:**
- [#16199](https://github.com/FRRouting/frr/pull/16199) — RFC v1 (June 2024)
- [#16532](https://github.com/FRRouting/frr/pull/16532) — RFC v2 (August 2024, self-closed after contributor identified technical issues)
- [#17343](https://github.com/FRRouting/frr/pull/17343) — RFC v3 (November 2024)

---

## The Technical Problem

RFC 8362 defines new LSA types that carry the same semantic data as existing LSAs, but wrapped in TLV (Type-Length-Value) headers:

```
Legacy LSA:     [LSA Header][Fixed Struct]
E-LSA:          [LSA Header][TLV Header][TLV Body]
```

ospf6d's algorithms operate directly on wire-format structs via pointer casts:

```c
struct ospf6_inter_prefix_lsa *prefix_lsa = lsa_after_header(lsa->header);
cost = prefix_lsa->metric;  // Direct field access
```

Adding E-LSA means algorithms must handle two memory layouts for the same semantic data.

---

## The Implementation Approach

The contributor introduced iterators/handlers to abstract over both formats:

```c
static const struct tlv_handler handlers[] = {
    { OSPF6_TLV_ROUTER_LINK, handle_router_link },
    { 0 }
};
foreach_lsdesc(lsa->header, handlers, &context);
```

The goal: keep algorithms format-agnostic while handlers dealt with wire format differences.

---

## Maintainer Feedback

**aceelindem (PR #16199):**
> "Callbacks...make logic harder to read, maintain, and debug"

Argued callbacks should remain "external interfaces between components" not internal iteration logic. Expressed concern about disrupting existing OSPFv3 deployments.

**eqvinox (PR #16199):**
Raised a security concern about function pointers stored on stack—potential RCE vector. Suggested two alternatives:
- `frr_each`-style iterators without callbacks
- Loading function pointers as constants rather than runtime-constructed structs

**eqvinox (PR #17343, speaking for TSC):**
> "The cost of making everything be callbacks with void pointers is too high... code maintainability concerns and safety against bugs."

**riw777 (PR #16199):**
Approved the approach but requested memory leak testing given the large changeset.

---

## Contributor Position

From PR #16199:
> Duplicating code for TLV-based processing would be unmaintainable long-term.

From PR #17343:
> "Without specific advice... we honestly don't know how to make this change more acceptable."

The contributor felt the suggested alternatives (frr_each-style, const pointers) had been explored but didn't resolve the fundamental need to iterate over format-varying structures.

---

## Why It Stalled

| Party | Primary Concern |
|-------|-----------------|
| Maintainers | Callback patterns make code harder to maintain and debug |
| Contributor | Code duplication leads to bugs and semantic drift |

Both positions have merit:
- Maintainers know their codebase and what patterns cause problems for their contributors
- The contributor correctly identified that duplication creates long-term maintenance burden

The gap: maintainers suggested alternatives at a conceptual level, but no one produced a concrete implementation that satisfied both concerns. The contributor didn't see a viable path with the suggested approaches; the maintainers didn't accept the callback approach.

---

## Comparison: ospfd TLV Handling

FRR's ospfd (IPv4 OSPF) handles TLV-encoded LSAs for Traffic Engineering, Segment Routing, and Extended Link/Prefix. The patterns differ:

| Aspect | ospfd (IPv4) | ospf6d E-LSA (proposed) |
|--------|--------------|-------------------------|
| TLV parsing | Switch-case in functions | Callback handlers |
| Code style | Inline, direct | Abstracted iteration |
| How it evolved | Feature-by-feature over years | Large change all at once |

### Trade-offs

| Criterion | Switch-case | Callbacks |
|-----------|-------------|-----------|
| Adding new format | Touch multiple switch statements | Add one handler |
| Bug fixes | Apply in multiple places | Apply once |
| Code duplication | Higher | Lower |
| Control flow | Direct, easy to trace | Indirect |
| Type safety | Direct struct access | Void pointers lose type info |
| Testability | Test whole function | Test handlers in isolation |
| Familiarity | Matches existing ospf6d style | New pattern for codebase |

Neither approach is objectively "right." The callback approach reduces duplication but adds indirection. The switch-case approach is familiar but scales poorly. Maintainers prioritized familiarity and debuggability; the contributor prioritized DRY principles and extensibility.

### Context

ospfd's TLV code arrived incrementally over years—smaller changes, less scrutiny per change. The ospf6d E-LSA work was a larger change touching many files at once, which naturally invites more resistance.

---

## The Underlying Architecture

The deeper issue predates E-LSA: ospf6d uses wire-format structs as internal data representation.

```
Wire bytes → cast to struct → algorithms operate directly on fields
```

This works fine with one wire format. Adding a second format forces a choice:
1. Branch on format everywhere (duplication)
2. Abstract over formats (callbacks/handlers)
3. Deserialize to internal types first (refactoring)

Option 3 would be cleanest but requires significant refactoring—likely out of scope and potentially even more contentious.

---

## Outcome

- **Users:** No RFC 8362 E-LSA support in FRR
- **Contributor:** Significant work not merged
- **Maintainers:** Avoided patterns they consider problematic
- **Codebase:** Status quo maintained

---

## Lessons

1. **Large architectural changes face more resistance** — Incremental changes over time may be more acceptable than a single large PR, even if the end result is the same.

2. **"Try X instead" needs follow-through** — Conceptual alternatives are hard to evaluate without concrete implementations. When suggesting alternatives, specificity helps.

3. **Maintainability is subjective** — What's maintainable depends on who's maintaining it. Patterns unfamiliar to current maintainers have real costs, even if they're "better" in the abstract.

4. **Impasse is a valid outcome** — Sometimes there's no solution that satisfies everyone. Walking away, while costly, is legitimate.

---

## Alternatives for Users

- **[Holo Routing](https://github.com/holo-routing/holo)** — Rust-based routing suite with RFC 8362 support
- **IS-IS** — Often better suited for datacenter/spine-leaf topologies
- **Babel** — Simpler protocol, modern design

---

## References

- [RFC 8362: OSPFv3 Link State Advertisement Extensibility](https://datatracker.ietf.org/doc/html/rfc8362)
- [PR #16199](https://github.com/FRRouting/frr/pull/16199) — RFC v1
- [PR #16532](https://github.com/FRRouting/frr/pull/16532) — RFC v2
- [PR #17343](https://github.com/FRRouting/frr/pull/17343) — RFC v3
