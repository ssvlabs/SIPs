# SIP Authoring Guidance

Use this guide when drafting or editing SSV Improvement Proposals in this repo.

## Start From The Repo Format

New SIPs SHOULD follow [template_sip.md](./template_sip.md) and the lifecycle in
[sips/sip0.md](./sips/sip0.md). Add the SIP under `sips/` and update the
relevant table of contents when the SIP is ready for review.

Each SIP SHOULD include:

- metadata table: author, title, category, status, dependencies, and date;
- summary: the problem and proposed change in a few sentences;
- rationale and design goals: why this change is needed and what constraints it
  optimizes for;
- specification: normative behavior, formats, algorithms, validation rules, and
  test expectations.

## Write For Compatible Implementations

A SIP MUST be precise enough that two developers, working independently in
different languages, can produce compatible implementations.

The specification SHOULD define:

- exact message, object, and wire formats;
- field names, types, byte order, encoding, limits, and default values;
- ordering rules for lists, maps, signatures, events, and hashes;
- domain separation, hashing, signing, and verification inputs;
- required validation, rejection, and fallback behavior;
- versioning and compatibility rules;
- test vectors or conformance cases for edge cases and invalid inputs.

If a reader must infer behavior from an implementation, issue, discussion, or
database layout, the SIP is underspecified.

## Style

Use RFC-style requirement words consistently:

- `MUST` and `MUST NOT` for required behavior;
- `SHOULD` and `SHOULD NOT` for recommended behavior with valid exceptions;
- `MAY` for optional behavior.

Keep SIPs concise. Prefer short normative statements, tables, and examples over
long prose. Include everything needed to implement and test the change, but
avoid implementation-specific commentary unless it affects compatibility.

When editing, remove ambiguity before adding detail. A concise SIP is not a
short SIP; it is a SIP where every sentence carries protocol meaning.
