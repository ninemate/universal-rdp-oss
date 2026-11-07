# 0001. Record architecture decisions as ADRs

- **Status:** Accepted
- **Date:** 2026-11-07
- **Deciders:** project owner (asked for ADRs to be written from the research)
- **Related:** [template](template.md), [research README](../research/README.md)

## Context

The research survey in [`docs/research/`](../research/README.md) ends with a list of decisions the project owner has to make, and the owner has now made several of them. Until now [`docs/architecture.md`](../architecture.md) recorded open questions as "TBD" with no place to keep the reasoning behind each answer. A remote-desktop stack has decisions that are expensive to reverse (language, reuse of IronRDP, codec licensing, server security defaults), and contributors will ask "why" long after the people who decided have moved on.

## Decision

We will record each significant architecture or project-policy decision as an Architecture Decision Record in `docs/adr/`, using the MADR-style [template](template.md): status, context, decision, consequences, alternatives considered.

- Files are named `NNNN-short-title.md` with a four-digit sequence number.
- **Accepted** is used only for decisions the project owner has made. Recommendations from research or from contributors stay **Proposed** until the owner accepts them.
- An Accepted ADR is not rewritten. A change of mind is a new ADR that supersedes the old one, and both status lines are updated.
- [`docs/architecture.md`](../architecture.md) and [`docs/ROADMAP.md`](../ROADMAP.md) describe the current state and link to the ADRs for the reasons. If they disagree with an Accepted ADR, the ADR wins until it is superseded.
- Items that are known to be open but are not yet decisions (for example the project name) are listed under "Open items" in the [ADR index](README.md), not given an ADR number.

## Consequences

- Reviewers can ask "which ADR covers this?" for any structural change.
- Writing an ADR adds a small cost to each decision. That is deliberate for decisions that are hard to reverse and unnecessary for local implementation choices.
- The index in [README.md](README.md) has to be kept current by whoever adds an ADR.

## Alternatives considered

- **Keep decisions in `architecture.md` only.** Rejected: the reasoning and rejected alternatives get lost as the document is edited.
- **Decisions in issues or PR threads.** Rejected as the primary record: hard to find, and tied to a single code host.
- **Nygard's original three-section format.** Rejected in favour of MADR because an explicit "alternatives" section matters for the licensing and reuse decisions here.
