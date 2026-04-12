# Core Framework: Layer N / Layer N-1 Invariant Analysis

The foundational technique: identify two adjacent processing layers where
Layer N has a security check and Layer N-1 processes input before or after
that check. The invariant gap is the space where the check at Layer N does
not cover the processing at Layer N-1.

## Application Pattern

```
1. Map the processing pipeline for a given input
2. Identify each layer's security responsibility
3. Find the boundary where one layer's assumption about
   another layer's behavior is wrong
4. Prove the gap with three-part evidence:
   a. Baseline: the check works in the normal path
   b. Control: the check fails in the alternative path
   c. Impact: the failure has measurable consequences

## Technique : Incomplete Fix Detection via Version Bisection

When a CVE fix exists, systematically determine whether YOUR specific
attack vector was covered by the fix.

### Method

```
1. Read the CVE advisory for the specific code path that was fixed
2. Identify what the fix validates:
   - URL validation? (initial URL only, or also redirect targets?)
   - Input sanitization? (which fields? all entry points?)
   - Size limits? (which layer? all frame types?)
3. Test on the FIXED version:
   - If your vector still works → fix bypass → new CVE
   - If your vector is blocked → duplicate → do not file
4. Check sibling code paths:
   - If the fix was applied to module A, was it also applied to module B?
   - If not → incomplete fix propagation → new finding
5. Check git blame:
   - Find the fix commit
   - List all files modified
   - If the affected file was NOT modified → confirmed gap
```

### Key Indicators

- Fix commit touches one module but not a structurally identical sibling
- Fix validates one code path but not an alternative entry point
- CVE advisory mentions "variant" or "related" issues found later

---

## Technique : Heap Dump Information Disclosure

When an OOM or crash produces a heap dump, analyze it for sensitive
data that was in memory at crash time.

### Method

```
1. Trigger a controlled OOM/crash that produces a heap dump
2. Locate the heap dump file (default: data directory, *.hprof)
3. Search for sensitive strings:
   - `strings heap_dump.hprof | grep -i "password\|api_key\|secret"`
4. Document: the heap dump contains ALL in-memory data in plaintext
5. Assess: who has access to the heap dump file?
   - Same-host filesystem → requires local access
   - Shared storage (K8s PVC, NFS) → broader exposure
   - The dump is NOT accessible via API — file-level access only
6. This is a SECONDARY impact, not the primary finding
```

---

## Anti-Patterns to Avoid

```
- Inferring impact without measurement ("this COULD crash the node")
- Claiming "gas-free" without verifying the gas metering configuration
- Claiming "chain halt" without verifying the timeout semantics
- Testing on in-memory stores and claiming production equivalence
- Filing the same vulnerability class twice against different modules
  without acknowledging the relationship
- Ignoring "by design" comments in the source code
- Conflating order-dependent behavior with commutativity bugs
- Reporting truncation residuals without economic viability analysis
- Claiming "no protection" without reading the close/error frame
- Assuming ping flood is a vulnerability (it's per-spec behavior)
- Claiming deserialization filter bypass when the filter is re-triggered
- Testing on EOL versions when current versions are available
- Filing duplicates of existing CVEs without checking NVD first
- Reporting timeout-based findings without waiting 2x the expected timeout
- Using protocol libraries for tests that require malformed frames
- Presenting messagesReceived=0 as proof without diagnosing the cause
```

```
- Inferring impact without measurement ("this COULD crash the node")
- Claiming "gas-free" without verifying the gas metering configuration
- Claiming "chain halt" without verifying the timeout semantics
- Testing on in-memory stores and claiming production equivalence
- Filing the same vulnerability class twice against different modules
  without acknowledging the relationship
- Ignoring "by design" comments in the source code
- Conflating order-dependent behavior with commutativity bugs
- Reporting truncation residuals without economic viability analysis
```
