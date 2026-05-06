# Internal Security Code Review Skills

**Internal snapshot:** 2026-05-06

This is a target-neutral checklist for reviewing company-owned repositories.
This version assumes you can inspect private deployment details, talk to
owners, create tickets, and land regression tests. It removes public-filing
concerns and focuses on finding, proving, fixing, and preventing security
defects in internal systems.

---

## How To Use This

- Start with production reachability, not cleverness.
- Map the trust boundaries before writing tests.
- Keep every claim tied to source, configuration, deployment evidence, logs, or
  a regression test.
- Prefer a small verified invariant over a broad unproven theory.
- End each review item with an owner, fix path, and regression test.

---

## Phase 0: Internal Scope And Ownership

### Asset And Data Classification

Before reading code, identify:

- What production asset this repo affects.
- What data it can read or mutate.
- What credentials, signing keys, tokens, or privileged roles it can reach.
- Whether it is internet-facing, partner-facing, employee-only, batch/offline,
  or library-only.
- Whether it is safety-critical, financial, identity, compliance, consensus,
  deployment, build, or observability infrastructure.

### Owner And Deployment Reality

Record:

- Service owner and on-call team.
- Production deployment unit: container, binary, package, function, mobile app,
  desktop app, smart contract, node plugin, or library.
- Current production version or commit.
- Feature flags and environment-specific differences.
- Whether staging meaningfully matches production.

Do not assume the repository root is what runs in production. Verify the
deployment artifact.

### Past-Incident And Owner Context

Before reading code, gather institutional context:

- Search the incident tracker, post-mortems, and security-review history for
  the repo / service / function names you plan to review.
- Search the change-management system for past deploys of the affected files.
- Skim recent merged PRs touching the security-relevant paths.
- Identify whether a prior review or audit covered this surface and what it
  closed.

Saves cycles when:

- A finding has already been reviewed and accepted as risk.
- A past incident touched the same code path and the fix is incomplete.
- The current behavior is a deliberate workaround for a known constraint.
- The surface is in active migration and the fix is already in flight.

Internal reviewers have access to this context. Skipping it produces tickets
that get closed "we know" or "this is being addressed in project X" — costing
owner trust.

### Code-Owner Conversation Before Drafting

For non-trivial findings, ask the code owner about intent before drafting the
ticket. Two questions are usually enough:

- "Why does this code do X?"
- "What threat were you defending against in line Y?"

The conversation often surfaces:

- Hidden invariants the code already enforces elsewhere.
- A defense at a different layer the reviewer missed.
- A documented operational policy that compensates.
- A confirmation that the gap is real, plus a faster fix path.

Talking to the owner is not adversarial. The goal is a correct fix shipped
quickly, not a flawless ticket.

### Threat Model

For every candidate, write the attacker preconditions:

- unauthenticated remote user,
- authenticated customer,
- tenant admin,
- malicious partner,
- compromised employee account,
- same-process plugin,
- local code execution,
- malicious integrator,
- misconfigured deployment,
- compromised dependency,
- operator error.

Then ask:

- Is this precondition inside the company's threat model?
- Does a real deployed path expose it?
- Which protected asset is harmed?
- Which compensating controls exist?

If the issue requires an already-compromised trusted operator or intentional
misuse of an internal helper API, record it as a hardening note unless it
defeats a defense-in-depth boundary the company explicitly relies on.

---

## Phase 1: System Mapping

### Trust Ladder

Draw the chain from input to privileged sink:

- ingress,
- parser,
- normalizer,
- validator,
- authorization check,
- dispatcher,
- state mutation,
- external call,
- persistence,
- event/log/audit output.

For each transition, record:

- intended invariant,
- actual enforcement mechanism,
- failure behavior,
- fallback path,
- owner.

Most internal findings appear where two adjacent layers disagree about the same
object or where one layer assumes another layer already enforced a property.

### Configuration-Gating Inventory

Search for knobs that change security behavior:

- feature flags,
- environment variables,
- tenant settings,
- allowlists / denylists,
- debug modes,
- "unsafe" / "trusted" / "skip verify" options,
- version fields,
- migration flags,
- kill switches,
- emergency bypasses.

For each:

- Who can change it?
- Is the change audited?
- Does it apply uniformly across workers, API nodes, background jobs, and
  migration paths?
- Can a caller downgrade enforcement by selecting a lower version or weaker
  mode?

### Artifact Inspection

Inspect the deployed artifact:

- container image,
- release tarball,
- package wheel/gem/jar/npm crate,
- compiled binary,
- generated config,
- infrastructure template,
- contract bytecode,
- mobile/desktop bundle.

Check for:

- files not present in source,
- generated code,
- bundled dependencies,
- debug symbols,
- unexpected config defaults,
- old versions,
- replacement/forked dependencies,
- startup scripts.

### Entry-Point Body Review

Read the actual body of each public/internal entry point before hypothesizing.
Do not infer behavior from interface names.

For each entry point:

- What type is accepted?
- What is copied vs borrowed?
- What is normalized?
- What is authenticated?
- What is authorized?
- What can fail after mutation?
- What external calls happen before commit?
- What cleanup is guaranteed?

### Layer N / Layer N-1 Mismatch Lens

Security checks often live at layer N while dangerous behavior lives at layer
N-1. The pattern is structural and recurs across protocol stacks, parsers,
runtime VMs, network frame layers, and middleware chains.

Pattern:

- Layer N validates a high-level representation (parsed object, decoded
  message, authorization principal).
- Layer N-1 has an alternate representation or control token (raw bytes,
  framing primitive, scheduling token, reset/error opcode).
- The lower layer handles something the upper filter never sees.

Concrete probes:

- Feed inputs that bypass normal constructors and reach a deserializer or
  framer directly.
- Exercise reset, abort, error, and re-send paths at the wire/frame layer.
- Compare what the validator sees against what the executor consumes.
- Compare admission-time check against execution-time mutation.
- Treat any framework that documents "validator verifies X" but where X is
  derived from a parsed view of bytes, not the bytes themselves, as a
  candidate.

When this lens fits, the assertion is "the security boundary is one layer too
high." Fixes typically move enforcement down a layer or strip the alternate
representation before validation.

### Internal Fan-Out Search

When a primitive shape appears in one service, search for it across the
monorepo and across sibling repos:

- copied source files,
- shared libraries with multiple wrappers,
- code generated from the same schema or IDL,
- forked modules,
- vendored / replaced dependencies,
- internal SDKs.

For each fan-out hit, verify whether the reachable impact is independent.
Document the fan-out in the original ticket so engineering coordinates the
fix across owners; do not file separate tickets that lose the connection.

Useful searches:

- function name and signature,
- error string,
- struct/field name,
- magic constant or sentinel value,
- IDL message name,
- middleware name.

---

## Hidden-Corner Lenses

Hidden corners are bugs the author did not name. They live in the code's
self-disclosure: where the author warned, where tools were silenced, where
cleanup is asymmetric, where the path is rarely taken, where the test does
not actually test. Each lens below is an attention pattern — what to grep
for, what to read twice, what to follow up.

These lenses are independent of any specific bug class. Apply them in passes
across the codebase before drilling into structural invariants. They surface
the surfaces the structural lenses should be applied to.

### Author Signals In Comments

Grep aggressively for author-disclosed risk:

```text
TODO  FIXME  XXX  HACK  HACKHACK  WTF  KLUDGE  BUG  GROSS  TEMP
"for now"  "temporary"  "should be"  "needs to"  "really should"
"don't touch"  "do not touch"  "magic"  "we should fix"
"this is wrong"  "this is broken"  "doesn't handle"  "ignores"
"placeholder"  "stub"  "dummy"  "until"  profanity
```

The age and density of these markers matters more than their content. A TODO
from years ago in a security path is either a real gap the team has lost
context on, or a closed concern nobody removed the marker for. Both states
deserve a follow-up.

Pair this with `git blame` on the marker line: if the comment author no
longer works on the project, the implicit promise to come back has lapsed.

### Suppressions And Workarounds

Each suppression marker is a place a tool flagged something and a human
overruled it:

```text
// nosec  # nosec  # noqa  // nolint  /* eslint-disable */  /* sonar-disable */
@SuppressWarnings  @SuppressLint  @SuppressFBWarnings  # type: ignore
unsafe { ... }  unsafe fn  pragma allow  pragma solidity-disable
trusted_  unchecked_  raw_  bypass_
```

For each suppression:

- What rule was suppressed?
- Why was it suppressed?
- Is the justification still valid?
- Did the suppression migrate to surrounding lines as the code evolved?

Findings often live one or two lines beyond the suppression marker — the
author silenced the warning on the line they intended to silence, but the
unsafe pattern leaked into adjacent code over later edits.

### Naming Smells

Names disclose intent. Read with skepticism:

```text
_internal  _private  _unsafe  _raw  _legacy  _old  _v1  _v2  _deprecated
_compat  _bypass  _trusted  _admin  _debug  _hack  _quick  _temp
doMagic  applyHack  unsafeFastPath  legacyCompat  trustMe  forceX
```

Specific shapes to investigate:

- Two functions with version suffixes side by side (`processV1`, `processV2`)
  — usually one path retains a less-restrictive check.
- Functions named after a person — single-author code paths often skip
  review.
- Inconsistent prefixes between siblings — the odd one out usually has odd
  semantics.
- Generic names (`util`, `helper`, `misc`, `common`) for files that contain
  security-significant logic — the file's classification disagrees with its
  contents.

### Git Archaeology

The history of a file is a search-light. Hot files are higher-yield:

- Files with many recent commits to a single function — recent churn around
  a security check usually leaves an inconsistency.
- Commits with messages like `fix`, `hotfix`, `patch`, `quick fix`, `revert`,
  `revert revert`, `band-aid` — incident artifacts often leave residue.
- Files where the last several commits are all "fix the fix" — the underlying
  invariant is unstable.
- Functions whose last edit was by a departed engineer — institutional
  memory has lapsed.
- Recent reverts in security-adjacent files — the original change may be
  partially still present.

Useful queries:

```bash
git log --since="6 months ago" --pretty=oneline -- path/to/security/file
git log --grep='hotfix\|emergency\|revert\|wtf' --pretty=oneline
git log --pretty=format:'%h %ae %s' -- path/ | sort -k2 | uniq -c -f1 | sort -rn
```

### Asymmetric Pairs

Many security bugs are pair imbalances. For each acquire/release-style pair,
verify the pairing on every code path including error paths:

| Acquire | Release |
|---|---|
| `open`/`Open` | `close`/`Close` |
| `acquire`/`Lock` | `release`/`Unlock` |
| `register` | `deregister`/`unregister` |
| `start`/`begin` | `stop`/`end` |
| `init`/`new` | `destroy`/`free`/`drop` |
| `commit`/begin | `rollback` |
| `enter` | `exit` |
| `addRef`/`incref` | `release`/`decref` |
| `subscribe` | `unsubscribe` |
| `set X = sentinel` | `clear X` |

Probes:

- Does every error return reach the release call?
- Does panic / exception unwinding reach it?
- Does cancellation, timeout, or signal-handler delivery reach it?
- Does the release run before the resource is observable to other actors,
  or after?

Cleanup that runs *after* the resource is observable is not cleanup — it is
a window.

### Default And Silent Fallback Smells

Fallbacks hide cases the author did not consider. Investigate every:

```text
||   ??   coalesce   getOrDefault   value_or   .unwrap_or   .or_else
try { ... } catch (...) { /* swallow */ }   except: pass   recover()
default: { /* nothing */ }   if !found { return ok }
.find(...)?.unwrap_or(DEFAULT)
```

For each:

- What does the fallback substitute, and does it have a security meaning?
  ("anonymous" replacing a missing principal is different from "0" replacing
  a missing count.)
- Does the fallback short-circuit a check the author thinks runs?
- Does the surrounding code distinguish "valid empty" from "absent"?

Especially scrutinize broad `catch` / `except` blocks in security-critical
code. A `catch (Exception)` around an authorization or signature-verification
call frequently masks the very error the verification was meant to surface.

### The "Almost Always" Branch

Rarely-executed paths receive disproportionately less testing. Hunt them:

```text
if rare_case   if (unlikely)   except SomeRareError
match … { _ => }   default: …   else { return STALE }
when feature_flag_off: …
when migration_complete: …
```

For each rare branch:

- How often does it actually fire in production logs?
- What state does it run in?
- Is the safe path or the dangerous path the rare one?

Operationally rare paths are also operationally untested paths. Bugs hide
where the test load did not.

### Forgotten Codepaths

Authentication and authorization tend to be carefully reviewed on the
request path and forgotten on adjacent paths:

- background workers and queue consumers,
- batch jobs, cron jobs, scheduled tasks,
- migration scripts and one-off database tools,
- admin and support tooling,
- internal dashboards and developer-only consoles,
- debug endpoints (`/debug`, `/internal`, `/_admin`),
- health-check, metrics, and profiling endpoints,
- import / export / data-export pipelines,
- replay tools, time-travel debugging, snapshot restores.

For each request-path enforcement found, locate and re-verify the
corresponding non-request-path. The most common finding is "the check exists
on the API, not on the queue worker that triggers the same action."

### Tests That Don't Test

A passing test gives the codebase implicit certification. Read the assertion:

```text
assertTrue(true)   assert 1 == 1
expect(result).not.toBeNull()    // without checking content
expect(...).toBeDefined()
catch (...) { /* test passes */ }
@Ignore   @Skip   @pytest.mark.skip
if (CI) { skip() }   #[ignore]
```

Watch for tests that:

- assert a return value is non-null but never inspect it,
- swallow exceptions and pass anyway,
- run only on platforms that the test environment never uses,
- compare against a hash that is committed alongside the test (so the test
  validates itself, not the production code),
- run a fuzzer for one iteration,
- mock the security-critical dependency (so the test certifies behavior
  the production code does not have).

A test whose name promises an invariant but whose body asserts something
weaker is a confidence hazard. Strengthen or delete.

### Generated, Vendored, And Templated Code

Generated code is rarely re-read after generation. If the generator template
contains a bug, the bug is everywhere the template is applied:

- IDL/protobuf/avro generated stubs,
- ORM-generated query builders,
- API-client SDKs,
- forms / validation generated from schemas,
- scaffold output from project generators,
- AI-assisted code suggestions never re-read by the author.

For vendored / forked / copy-pasted libraries:

- diff against upstream — what was changed and why?
- check whether security patches in upstream were ever pulled in,
- check whether the fork is documented, owned, and on a refresh schedule.

The "we vendored it three years ago and forgot" pattern is reliably
high-yield.

### Documentation Drift

Documentation is what the author thinks the code does. The two diverge over
time. For each load-bearing docstring, comment, README claim, or spec:

- read the code and verify the claim still holds,
- treat any deviation as either a bug in the code, a stale doc, or both,
- prefer fixing the doc only when the code is correct and the gap is
  cosmetic.

Particularly worth re-reading: security claims in module-level docstrings,
"this function MUST" comments, and error-handling promises in interface
documentation.

### Implicit State

Code that depends on state not visible at the call site is hard to review:

- mutable globals, package-level variables,
- thread-local / async-local / context-local storage,
- environment variables read at runtime,
- on-disk state files, lock files, marker files,
- in-process caches that outlive a request,
- singleton service objects with lifecycle managed elsewhere.

For each, the question is: *what assumes this state is in shape X, and what
sets it to shape X?* If the chain has gaps, the assumption is unverified.

### String-Boundary Concatenation

Any string built via concatenation that crosses a privileged boundary is a
candidate. Search for:

- shell command construction,
- SQL or NoSQL query construction,
- file-path construction (especially with user-supplied components),
- URL construction (especially scheme + host + path joining),
- HTML / XML / SVG construction,
- JSON / YAML built by string templates rather than serializers,
- log lines built with attacker-controlled fields and parsed downstream,
- email headers / MIME parts.

For each, verify that the boundary uses a structured builder (parameterized
query, `subprocess` arg list, URL builder, escaping serializer). Bare string
concatenation at a privileged boundary is a default-bad pattern.

### Reflection And Dynamic Dispatch

Authorization rarely tracks through dynamic dispatch:

- `eval`, `exec`, `Function(...)` constructors,
- `Class.forName`, `Method.invoke`, `getattr`, `__getattr__`,
- dynamic SQL, dynamic templates,
- `unmarshal` / `pickle.loads` / `Marshal.load` / `ObjectInputStream`,
- plugin loaders, hot-reload modules,
- callback registries that resolve names at call time.

For each dynamic call, the question is: *can attacker-controlled input
influence which method runs?* If yes, the call site is a candidate.

### Single-Caller Helpers

A function with exactly one caller is sometimes a wrapper hiding complexity
the caller does not want to look at:

- the wrapper performs a check the caller assumes ran;
- the wrapper performs a transformation the caller does not see;
- the wrapper has accumulated edits that the original caller never reviewed.

Read the body. If the wrapper is a thin pass-through, fold it mentally into
the caller. If the wrapper has logic the caller does not know about, that
delta is the corner.

### Magic Constants Without Provenance

Hardcoded numbers without a constant name and without a comment explaining
the rationale are a smell:

```text
return data[:8192]    // why 8192?
if retries > 3        // why 3?
sleep(150)            // why 150ms?
const TIMEOUT = 30
```

For each:

- What other code assumes this exact value?
- Was the value chosen, or copy-pasted, or derived from an external spec?
- Does increasing or decreasing it break a security property?

Constants that should match across systems often drift. Two services with
"the same" 30-second timeout can diverge at refactor and produce a
window-mismatch finding.

### The "Interesting Name" Heuristic

Some authors signal their own concern through naming. Open the file when you
see:

- `unsafeFastPath`, `_doMagic`, `applyHack`, `legacyCompat`, `_workaround`,
- file names like `quirks.py`, `compat.go`, `legacy_handlers.rs`,
- packages named after a constraint (`bypass`, `relaxed`, `trusted`).

The author has flagged the file. Follow the flag.

### Two-Implementation Differential

When the same logical protocol or contract has two implementations side by
side, the diff between them is high-yield:

- optimized vs reference implementation,
- assembly / hand-tuned vs straight-line version,
- v1 vs v2 of the same handler kept for compatibility,
- library code vs CLI wrapper of the same operation,
- service A vs service B both implementing the same RPC schema,
- reference test-vector implementation vs production implementation.

Probes:

- For every public entry point in the optimized version, find the
  corresponding reference function and read both bodies side by side.
- Note any check present in one but not the other. Either the missing check
  is implicit in a precondition or it is a real divergence.
- Note any input-shape the optimized version accepts that the reference
  rejects (or vice versa). Differential acceptance is often the corner.
- When the two implementations disagree on a single input, the question is:
  which one is the security oracle? Disagreement on edge inputs is a real
  finding even when neither implementation is "wrong" in isolation.

The two-implementation differential is the most reliable hidden-corner lens
on mature, audit-saturated targets. Auditors check each implementation
in isolation; the divergence between them is what gets missed.

### Self-Authorized Acceptance

When a candidate "exploit" turns out to require the attacker to also be the
protected party, the impact collapses. Detect this early:

- Trace every authorization check on the candidate path.
- For each, ask: does this check require `attacker == legitimate_principal`?
- If yes, the path is self-authorized — the attacker can already do whatever
  they like to their own resources.

A self-authorized acceptance is still worth a regression test (the path
exists, the check is non-obvious), but it is not bridge-class. Recognize the
shape early to avoid investing in a report whose impact is moot.

The dual lens — useful when triaging others' reports as well — is "would
the same outcome be reachable through a documented public action by this
principal?" If yes, no security boundary was crossed.

### Real-Path Bridge Enumeration

For any finding that depends on a representation split, an off-chain /
on-chain mismatch, or a layer-N / layer-N-1 disagreement, exhaustively
enumerate every real deployed component that could bridge the split before
declaring impact:

- routers, gateways, aggregators,
- indexers, search services, analytics consumers,
- simulators, dry-run helpers, suggested-call generators,
- off-chain signers, oracle services, attestation services,
- compliance, monitoring, alerting consumers,
- backup, restore, migration, replay tooling.

For each, verify whether it makes a security decision on the wrong side of
the split. If none does, the split is observation-only — keep the
regression test, do not file as impact.

The discipline is: before claiming "this raw-hash differs from this
semantic-hash, therefore X," map every consumer of both hashes and confirm
at least one consumer is *both* the security oracle *and* on the wrong
side.

### Encoding And Bytecode-Layer Lenses

For protocols that operate on raw bytes, decode-layer corners are
particularly under-covered. Run a separate pass focused on the bytes
themselves, not the decoded objects:

- **Dirty high bits.** Fixed-width words (256-bit, 64-bit) used to carry
  smaller-width values (160-bit address, 48-bit timestamp) often retain
  attacker-controllable high bits. The decoded value masks them; raw hashes
  preserve them.
- **Trailing data.** Canonical decoders frequently accept calldata, message
  bodies, or framed payloads with appended bytes after the last consumed
  field. The semantic interpretation is unchanged; the raw hash differs.
- **Overlapping or aliased offsets.** ABI-style decoders that follow
  pointer offsets into a payload may accept overlapping regions, where two
  decoded fields share underlying bytes.
- **Non-strict enum / discriminant decoding.** When an enum is encoded as
  a wider integer than its variant count, the high bits become free
  parameters that some implementations preserve and others zero.
- **Duplicate encodings.** Different byte sequences that decode to the same
  semantic content. Especially common in length-prefixed and tag-prefixed
  encodings.
- **Canonical vs non-canonical signatures.** Multiple valid encodings of
  the same logical signature (high-S vs low-S, multiple recovery IDs,
  malleable r/s pairs).

For each shape, ask the bridge question (above): does any deployed consumer
make a security decision based on raw bytes while another makes a decision
based on decoded semantics?

### Cross-Protocol Primitive Reuse

When a cryptographic primitive (commitment scheme, hash, signature scheme,
KDF, ZKP transcript) is used in two different protocol contexts within the
same codebase, verify domain separation:

- enumerate every protocol that calls the primitive,
- record the input format each protocol uses,
- check whether each input has a unique domain tag, label, or wrapping
  envelope,
- check whether a value valid in one protocol could be replayed as valid
  in another.

The corner is: the same cryptographic operation produces an output that
satisfies two protocols. An attacker who can cause one protocol to produce
the output (legitimately, in the protocol's normal flow) can replay it
into the second protocol's verification.

Particularly worth auditing:

- shared transcript prefixes between sign and verify protocols,
- shared randomness sources,
- shared commitment trees used for both proofs and audit logs,
- threshold protocols where a participant signs a message valid in two
  different ceremonies,
- legacy compatibility paths that re-use a primitive from an older
  protocol version without re-domain-separating.

### Natural Vs Synthetic Test Inputs

When probing diagnostic, error-handling, or symbol-processing code paths,
prefer inputs generated by realistic code execution over synthetic crafted
fixtures.

A crash from a hand-crafted symbol fixture is a different finding from a
crash from a symbol generated by the project's own type system in a normal
operation. The latter implies "real users hit this"; the former implies
"a malicious local actor crafting the fixture hits this."

For each diagnostic-path candidate:

- Can you trigger the crashing input via the normal API surface?
- What input shape does the natural code path produce?
- If only synthetic fixtures crash, the finding is local-misuse-only and
  classifies as hardening.

### Stripped Vs Unstripped Build Classification

Diagnostic-path crashes (stack-trace generation, symbol demangling, debug
formatting, panic hooks) may exist in debug-symbol builds but be absent in
stripped production builds.

For every diagnostic-path finding, verify:

- which build configuration retains the symbol/debug path,
- whether production deployments use that configuration,
- whether the crash is triggerable with stripped symbols (zero-byte symbol
  table) or only with non-empty symbol tables.

Without this classification, the finding is mis-scoped — claimed as
production-impacting when only debug builds are affected.

### Audit-Document Cross-Reference

When the target ships an internal audit document, threat model, security
review, or post-mortem, treat it as a treasure map:

- every "noted but not addressed" entry is a candidate,
- every "tracked under issue X" entry should be checked against issue X's
  actual status,
- every "mitigated by Y" claim should be verified against Y's current
  implementation,
- every accepted-risk entry should be re-verified against current
  deployment shape — accepted risks frequently outlive the deployment
  configuration that justified them.

The audit-document author has already done part of the work. The follow-up
is verifying that mitigations promised in the document are still in force.

### Test Name Vs Test Body Mismatch

When a test's name promises an invariant but its body asserts something
weaker or different, the test is certifying behavior the production code
does not have.

Examples:

- `test_rejects_invalid_input` whose body only checks that an exception was
  raised, not that no state was mutated.
- `test_authorization_required` whose body sends a request without auth
  and asserts the response code is non-200, without verifying *which*
  non-200 code (auth-failure vs validation-failure are different).
- `test_cleanup_runs_on_failure` whose body triggers failure but never
  checks that the cleanup side-effect occurred.
- A test whose body is a copy of an adjacent test with one renamed
  assertion that no longer matches the test name.

These mismatches are higher-yield than they look: a passing test in
security-critical code that does not actually test the named property is a
false certification, and the production code may have lost the invariant
silently.

### Cryptographic Staging-Buffer Audit

For cryptographic state machines that stage data before committing
(refresh, rotate, multi-round protocols, threshold ceremonies), audit every
stage/commit/rollback/cleanup transition:

- where is the staged data stored?
- what API commits it?
- what API discards it?
- what happens on partial failure between stage and commit?
- what happens if commit is called twice?
- what happens if the live state is consumed between stage and commit?
- what happens if a cleanup fails but the live state was already mutated?

Every staging buffer is a candidate for "stale state resurrection": the
buffer holds a value that should have been invalidated, and a later commit
restores it past the boundary that was supposed to consume it.

Common shapes:

- presignature or nonce material staged for a future signing operation,
- session token staged for a future authentication,
- approval staged for a future authorization,
- state snapshot staged for a future restore.

For each, the question is: can the staged value survive past the live
operation that should have consumed it?

### Library Test-Harness As PoC Delivery Surface

When the target ships an extensive test harness (Catch2, gtest, Foundry,
pytest, JUnit), the harness is a PoC delivery surface:

- the triager runs the existing test suite as part of triage anyway,
- a PoC delivered as a test fragment in the project's own harness has
  zero environment friction,
- the same harness is the regression-test landing site after the fix.

Probes when reviewing internally:

- where is the project's main test harness?
- are there existing tests that exercise the candidate path?
- can the regression test be modeled on an existing test that already
  passes?
- is there a fuzzer / property-test harness whose corpus could be extended?

The deliverable is a PR that adds the failing test alongside the fix. The
test fails before the fix, passes after. Reviewer and triager align on the
same artifact.

---

## Phase 2: Invariant Testing

### Name Tests By Invariant

Use names like:

- `RejectsDowngradedTenantPolicyBeforeDispatch`
- `NonceIsConsumedOnlyAfterSuccessfulCommit`
- `AuditLogMatchesExecutedPrincipal`
- `RefreshCannotRestoreConsumedToken`

### Passing Test Means Bug Demonstrated

For a vulnerability regression test:

- baseline valid behavior passes,
- direct invalid path rejects,
- exploit path reaches the sink,
- final state proves impact.

Avoid tests that only assert "parser accepts weird input" unless parser
acceptance is itself the security boundary.

### Assert The Sink

Assert the outcome that matters:

- money moved,
- data disclosed,
- permission granted,
- message dispatched,
- job executed,
- token minted,
- signature emitted,
- file written,
- audit record missing or false,
- policy bypassed,
- resource exhausted.

### Two-Tier Claims

Separate:

- What the test proves.
- What production exposure depends on.

Example:

> The test proves a stale staged record can be committed after the live record is
> consumed. Production impact depends on whether retry workers preserve this
> staging order.

### Negative Controls

Include controls that show:

- the intended check works on the direct path,
- a safe configuration blocks the attack,
- the exploit depends on the specific missing invariant,
- rollback happens when expected.

Controls turn "weird behavior" into a precise regression.

---

## Phase 3: High-Value Review Lenses

### Asymmetric Failure

Template:

> If component A fails, component B must fail closed.

Probe:

- state write succeeds then downstream validation fails,
- authorization succeeds then dispatch fails,
- cache update succeeds then persistence fails,
- audit write fails after privileged action,
- signature verification fails after partial decode,
- external service times out after local mutation.

Question:

> What state has already changed when the error is thrown?

### Transitional-State Races

For each multi-step workflow:

1. pause between phases,
2. fail between phases,
3. re-enter between phases,
4. retry from old state,
5. observe which state is durable.

Common phase gaps:

- stage → commit,
- authorize → execute,
- parse → validate,
- verify → persist,
- reserve → charge,
- create → initialize,
- burn → attest → mint,
- refresh → sign → finalize,
- lock → transfer → unlock.

### Representation Splits

Look for raw/semantic mismatches:

- raw bytes vs parsed object,
- encoded path vs normalized path,
- user-visible address vs canonical address,
- event state vs storage state,
- cache key vs authorization key,
- signature domain vs execution domain,
- audit log principal vs executed principal.

File internally when a real decision crosses the split:

- authorization uses one representation,
- execution uses another,
- audit logs a third,
- remediation relies on a fourth.

### Version And Mode Downgrades

Inventory caller-controlled:

- protocol version,
- API version,
- tenant mode,
- compatibility mode,
- migration state,
- "trusted" flag,
- verification level.

Risk patterns:

- upper-bound-only checks,
- no lower bound,
- mitigation gated by caller-selected version,
- version stored but not bound to object,
- parser accepts old shape but executor assumes new shape.

### Staging-Then-Commit Resurrection

Bug shape:

1. stage old state,
2. consume or mutate live state,
3. commit stale staged state.

High-impact targets:

- one-time tokens,
- nonces,
- presigned material,
- refresh state,
- approvals,
- locks,
- policy flags,
- replay caches.

### Reachability Bridge For Library Bugs

For internal libraries, prove:

- which service imports it,
- which entry point is called,
- what input reaches it,
- whether the unsafe precondition occurs in production,
- whether callers use safe wrappers.

If no deployed service crosses the boundary, file as hardening and add a
regression test in the library.

### Resource-Metering Asymmetry

For metered systems:

- compare charged cost vs actual CPU,
- compare accounted memory vs actual allocation,
- compare valid vs invalid input cost,
- compare static estimate vs runtime behavior,
- test cleanup after budget exhaustion.

Impact requires a real resource boundary: service DoS, cost bypass, queue
starvation, node liveness, or quota evasion.

### Diagnostic / Error-Path Bugs

Review:

- stack trace generation,
- symbol demangling,
- panic hooks,
- debug serializers,
- logging of attacker-controlled values,
- error-to-response conversion.

For internal risk, classify by deployment:

- production debug symbols retained,
- crash loop possible,
- request-triggered,
- worker-only,
- staging-only.

### Cryptographic State Machines

Prioritize:

- nonce reuse,
- single-use violation,
- stale presignature restore,
- refresh/rotate interleaving,
- transcript-binding gaps,
- version-gated mitigation downgrade,
- cross-protocol key confusion.

Avoid generic memory bugs in crypto code unless they reach key disclosure,
signature forgery, persistent DoS, or an accepted availability class.

### Authentication / Authorization Matrix

For each privileged operation, draw an explicit matrix:

| Principal | Authentication required | Authorization mechanism | Boundaries enforced | Logged |
|---|---|---|---|---|
| anonymous | — | — | — | — |
| authenticated user | session/token | role check | tenant, resource | y/n |
| service account | mTLS / signed JWT | scope check | namespace, action | y/n |
| internal admin | SSO + 2FA | RBAC | global | y/n |
| automation | API key / signed request | scope check | rate, action | y/n |

Bug shapes to probe:

- A high-privilege path that accepts a low-privilege authenticator (token
  shape collision).
- A role check that runs before normalization (path traversal, unicode
  confusables, case folding).
- An authorization decision derived from request-supplied identifiers rather
  than session-bound identity.
- A check that exists at one entry point but not at a sibling that reaches
  the same sink (gRPC vs REST, internal vs external API, batch vs stream).
- A re-entry path (callback, webhook, admin override, support tooling) that
  bypasses the standard check chain.

### Secret And Credential Flow

Trace each secret from issuance to use:

- where it is generated,
- where it is stored (env var, secrets manager, encrypted column, in-memory
  cache, build artifact),
- which processes can read it,
- which network paths it traverses,
- where it is logged or could be logged,
- what rotation procedure exists,
- what happens to in-flight uses during rotation,
- what kill-switch exists for compromise.

Common findings:

- Token in URL or query string (logged by load balancer).
- Long-lived credential where short-lived would suffice.
- Secret accessible to a sibling service with no need-to-know.
- Rotation procedure that leaves stale copies in caches.
- Recovery / break-glass credential without separate audit trail.
- Secret committed to source, build artifact, container image, or backup.

### Multi-Tenant Isolation

For multi-tenant services, every query, write, and side-effect must be bound
to the tenant. Inventory:

- request entry points and how tenant ID is derived,
- background jobs and how they pick up tenant context,
- shared caches and whether keys are tenant-scoped,
- shared databases and whether queries filter by tenant,
- shared object storage and whether object names include tenant prefix,
- shared message queues and whether consumers verify tenant on dequeue,
- usage metering and whether counters are tenant-scoped.

Bug shapes:

- Tenant ID from request body instead of session.
- Tenant ID from cache key but cache key constructed from raw request.
- Background worker that fetches "all pending" and processes without
  per-tenant authorization.
- Shared rate limiter keyed on global identifier.
- Migration script that touches all tenants without per-tenant guard.

### Logging And Audit-Trail Integrity

Audit logs are a security control. Probe whether:

- the logged principal matches the principal that performed the action,
- the logged action matches what actually executed (not what was requested),
- the log is written before or after commit, and what failure mode is chosen,
- the log captures both success and failure paths,
- the log is tamper-evident or tamper-resistant,
- the log retention satisfies compliance requirements,
- attacker-controlled input in the log can break parsers, injection
  downstream, or evade alerting,
- impersonation, sudo, support-tool, or break-glass paths are distinguished.

A finding here is real impact even when the underlying action is authorized:
incorrect audit trail blocks incident response and breaks compliance.

### Compliance Mapping

For each finding, identify which control framework section it touches:

- access control (e.g., SOC 2 CC6, ISO 27001 A.9, PCI 7-8),
- audit logging (SOC 2 CC7, PCI 10),
- encryption (PCI 3-4, HIPAA 164.312),
- change management (SOC 2 CC8, ISO A.12.1),
- availability (SOC 2 A1),
- data subject rights (GDPR Art. 17, 20).

The mapping does not change technical severity, but it changes who needs to
sign off on remediation timelines, whether a regression test must be retained
for audit, and whether the finding triggers external disclosure obligations.

---

## Phase 4: Production Reachability

### Verify Production Path

For every confirmed bug, answer:

- Is the vulnerable code deployed?
- Which environment has it enabled?
- Which feature flag controls it?
- Which customer/tenant/user path reaches it?
- What logs prove it is called?
- What metrics show frequency?
- What compensating controls exist?

### Deployment Matrix

Classify:

| Environment | Affected? | Notes |
|---|---|---|
| production | yes/no/unknown | exact version and flag state |
| staging | yes/no/unknown | parity with production |
| local/dev | yes/no/unknown | useful only for regression |
| worker/batch | yes/no/unknown | queue and retry impact |
| library consumers | yes/no/unknown | import graph required |

### Blast Radius

Estimate:

- affected tenants,
- affected assets,
- privilege needed,
- automation potential,
- rate limits,
- monitoring,
- rollback complexity,
- data recovery,
- customer-visible impact.

---

## Phase 5: Reporting Internally

### Internal Report Structure

1. Summary.
2. Affected service/repo/version.
3. Production reachability.
4. Impact.
5. Root cause.
6. Reproduction steps.
7. Regression test.
8. Recommended fix.
9. Owner and priority.
10. Rollout / backport notes.
11. Detection and monitoring.

### Severity

Use company risk criteria, but keep calibration honest:

| Shape | Typical internal priority |
|---|---|
| remote unauthenticated asset impact | urgent |
| authenticated cross-tenant impact | high |
| tenant-local privilege escalation | high/medium |
| production DoS with realistic trigger | high/medium |
| supported public API + deployed unsafe caller | medium/high |
| library bug + no deployed unsafe caller | hardening |
| local-only intentional misuse | hardening |
| debug-only crash absent in production | hardening |

### Remediation

Give the owner options:

- narrow invariant check,
- fail-closed reorder,
- immutable snapshot,
- bind metadata to signed/authenticated object,
- move check to sink layer,
- remove unsafe mode,
- add explicit documentation and type restrictions,
- add migration cleanup,
- add telemetry.

Always include the regression test that should fail before the fix and pass
after the fix.

### Working With Code Owners On The Fix

The ticket is a starting point, not a handoff:

- Pair on the first design discussion. The owner knows constraints the
  reviewer does not.
- If the reviewer can write the patch, write it. A working PR with the
  regression test attached gets merged faster than a ticket with a
  recommendation.
- For cross-service findings, identify all owners and convene one fix
  discussion rather than one ticket per service.
- If the fix is delayed, agree on a compensating control (feature flag off,
  monitoring alert, rate limit) and timebox it.
- After the fix lands, verify the deployed artifact, not just the merged PR.

### Multi-Pass Pre-Ticketing Verification

Before opening the ticket, run six checks against the writeup. Each pass has
a different question; verify against primary source.

1. **Core claim.** One-sentence boundary-violation description that matches
   the regression test.
2. **Primitives.** Every file path, line, function name, and commit hash
   exists in the cited source on the cited branch.
3. **Production reachability.** Deployment, flag state, and caller path are
   verified, not assumed.
4. **Severity.** Internal priority matches reachability and asset impact, not
   theoretical worst case.
5. **Owner and fix path.** Owner identified; fix-path option list is
   actionable; regression test exists.
6. **Concerns.** Push-back the owner is likely to make is addressed: "this
   is by design," "another layer enforces it," "this code is being deleted
   anyway."

A failed pass is a prompt to add a sentence or fix a fact, not a blocker.
Skipping the passes produces tickets that bounce back with "not enough
information."

### Round-By-Round Internal Draft Discipline

Reviews iterate. Plan for 3-4 rounds:

| Round | Typical correction |
|---|---|
| 1 | Pivot away from over-broad framing. State the standalone primitive. |
| 2 | Tighten claims to match what the test proves. Drop "arbitrary X". |
| 3 | Pin specifics: deployed version, flag state, owner, regression test. |
| 4 | Pre-empt owner pushback; cite primary source for each. |

The first draft is always wrong about something. The pattern-matching that
finds the bug is not the same skill as the verification that documents it
correctly.

---

## False Positives And Park Conditions

Park or downgrade when:

- No production caller reaches the path.
- The only trigger is intentional same-process misuse.
- The caller already has equivalent privileges.
- The issue requires a malicious service owner.
- The unsafe path exists only in tests or demo code.
- The default production caller uses a safe representation.
- The bug is a raw/semantic split with no decision crossing it.
- The bug requires debug-only build settings not used in production.
- The test edits internal state that no real workflow can edit.
- The behavior is documented and monitored as an accepted operational risk.

Still create a hardening ticket when the fix is cheap and prevents future
misuse.

---

## Cross-Cutting Internal Practice

### Comparable-Architecture Verification

When proposing "this is how our other services do X," read 2-3 comparable
services first. Cite the specific entry points or middleware. Distinguish:

- "All payment services in our org enforce idempotency keys somewhere" —
  durable claim.
- "All payment services use middleware M for idempotency" — narrower; fails
  if one uses a different mechanism.

When the comparison disconfirms an initial belief, the comparison strengthens
the finding: "Service A and Service B both bind tenant in middleware. Service
C binds in the handler, which is why this finding exists in C and not in A or
B." That framing is harder to dismiss than "service C should add the check."

### Detection And Monitoring As A Deliverable

For high-severity findings, the deliverable includes detection. Decide:

- can existing logs distinguish exploit attempts from normal traffic?
- if not, what log line or metric would?
- can an alert be authored on the new signal?
- does the SIEM rule already cover the signal under a different name?
- what false-positive rate is acceptable?

The fix prevents the bug; the detection catches the next variant. Both
should land in the same review cycle when severity warrants.

### Static-Analysis And Scanner Calibration Loop

Each finding is a calibration data point for the company's automated tooling:

- Did SAST flag this? If not, why? Is the rule missing, or is the pattern
  outside what the engine can match?
- Did the secret scanner catch the credential exposure?
- Did the dependency scanner catch the vulnerable transitive dep?
- Did the IaC scanner catch the misconfiguration?

Where automation missed the finding, file a separate (low-priority) ticket
to add the rule. Calibration is durable; manual re-reviews of the same
surface are not.

### Coverage Bookkeeping

Maintain a lightweight ledger of what was reviewed, when, by whom, against
what threat model. For each entry:

- repo / service / module,
- review date,
- review scope (what was in, what was deferred),
- threat model assumed,
- findings filed,
- closure (fixed, accepted, deferred),
- next review trigger (next major release, time-based, change-driven).

The ledger answers two recurring questions:

- "Has this been reviewed?" — yes/no/partial, with date.
- "What changed since last review?" — drives the trigger for re-review.

Without a ledger, the same surface gets re-reviewed by accident and adjacent
surfaces stay un-reviewed indefinitely.

### When A Review Closes Empty

A review that produces no findings is not a wasted review. Capture:

- the lenses applied,
- the surfaces examined,
- the corners that closed and why,
- the regression tests added along the way,
- any new threat-model assumptions surfaced.

The empty close is institutional knowledge: future reviewers know which
lenses have been tried, which assumptions are documented, and which
hypotheses were already falsified. Padding a low-quality finding to justify
the time spent is worse than recording the empty close honestly.

---

## Internal Anti-Patterns

1. **Starting with clever edge cases before mapping production reachability.**

2. **Confusing consequence with exploitability.** Proving "if this state is
   restored, bad things happen" is not the same as proving a real workflow
   restores that state.

3. **Filing local-only API misuse as a security incident.** If it requires
   local code execution or a malicious same-process caller, classify it as
   hardening unless it defeats a documented defense boundary.

4. **Stopping at parser acceptance.** Assert the privileged sink.

5. **Ignoring rollback and cleanup.** Many bugs are half-commits.

6. **Not checking background workers.** Queues, retries, cron jobs, and
   migration workers often bypass request-path checks.

7. **Assuming staging equals production.** Verify flags and artifacts.

8. **Writing a report without an owner or fix path.** Internal reviews should
   end in remediation, not just discovery.

9. **Letting tests flake.** Races need synchronization, retries, or narrower
   claims.

10. **Over-classifying hardening as urgent.** This burns owner trust. Keep
    severity tied to deployed reachability and asset impact.

11. **Filing without talking to the code owner first.** For non-trivial
    findings, a 10-minute conversation often reveals a defense the reviewer
    missed, a hidden invariant elsewhere, or a faster fix path. Adversarial
    discovery delivers worse outcomes than collaborative discovery.

12. **Skipping the incident-tracker and post-mortem search.** Past
    institutional knowledge is the single highest-leverage source for
    internal review. Tickets that ignore it get closed "we know" or "this is
    being addressed."

13. **Treating SAST output as ground truth without manual confirmation.**
    Static analyzers produce false positives at rates that flood ticket
    queues. Confirm reachability and impact manually before filing.

14. **Filing without a remediation owner.** A ticket without a code owner is
    a low-priority backlog item. Identify the owner before filing, even if
    that requires asking around.

15. **Ignoring background workers, batch jobs, migrations, and admin
    tooling.** Request-path checks routinely do not apply to these surfaces.
    Whenever a request-path enforcement is reviewed, the matching
    background-path enforcement should be reviewed at the same time.

16. **Letting one cross-service finding fragment into multiple disconnected
    tickets.** When a primitive recurs across services, file one ticket with
    all owners tagged and a coordinated fix plan. Separate tickets lose the
    structural connection and produce inconsistent fixes.

17. **No deployed-artifact verification after the fix merges.** A merged PR
    is not a deployed fix. Confirm the affected version is in production
    before closing the ticket.

---

## Regression Test Checklist

For every accepted issue:

- direct vulnerable path reproduces,
- safe path still works,
- invalid path rejects before mutation,
- state is unchanged on failure,
- audit/log output matches executed behavior,
- retry/replay path is covered,
- feature flag / version behavior is covered,
- test fails before fix and passes after fix.

---

## Closing Rule

Every reviewed candidate should end as one of:

- fixed,
- ticketed,
- hardening note,
- regression-only,
- accepted risk,
- false positive.

Record why. Future reviewers should not have to rediscover the same boundary.
