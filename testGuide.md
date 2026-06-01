# SQLite-Inspired Testing Guidelines

Agent testing rules. Based on SQLite testing philosophy + current suite.

**Tradeoff:** Evidence over speed. Test every change, including small UI label changes. Scale breadth/depth by risk: label change needs focused verification; storage engine, migration, parser, auth flow, payment path may need full matrix.

See [`testExamples.md`](./testExamples.md): concrete SQLite public-suite patterns.

## 1. Test the Contract, Not the Implementation

**Verify observable behavior. Do not echo implementation.**

Before writing tests:
- Identify public contract: inputs, outputs, errors, state changes, invariants.
- Test through published interfaces where practical.
- Assert durable behavior, not private call order/incidental structure.
- Keep white-box tests when they expose important internal invariants or unreachable branches.
- For critical correctness, use second implementation, reference system, or independent oracle.

SQLite: independent harnesses + differential testing. SQL Logic Test compares results across database engines. Independence: implementation must not grade itself.

**The test:** Useful after correct refactor? If not, likely implementation-coupled.

## 2. Use Multiple Layers of Evidence

**No single test kind suffices.**

Build a portfolio:
- Example tests for known behavior.
- Boundary tests for limits and off-by-one errors.
- Regression tests for every fixed bug.
- Fault injection for non-happy paths.
- Fuzz/property tests for unimagined cases.
- Differential tests against an independent oracle.
- Concurrency and stress tests for timing-sensitive behavior.
- Performance tests for important workloads.
- Static/dynamic analysis for defect classes assertions cannot express.
- Release checklists for checks that require judgment.

SQLite uses separately maintained harnesses, specialist programs, runtime instrumentation, fuzzers, manual release checklists. Layers find different failures.

**The test:** If serious bug survives this layer, which layer catches it?

## 3. Make the Fast Loop Strong

**Run useful tests often. Run costly tests deliberately.**

Use tiers:
- **Tiny:** the narrow test that reproduces the change.
- **Fast:** default pre-commit suite; fast enough for habitual use.
- **Full:** slower integration, fault, fuzz-corpus, and configuration tests.
- **Soak:** long-running fuzz, stress, concurrency, and performance checks.
- **Release:** full matrix plus a human-reviewed checklist.

SQLite has `veryquick`, `quick`, `full`, soak-style runs. `veryquick` omits costly malloc, I/O error, fault, soak cases: meaningful pre-check-in subset.

For agents:
- Run the narrowest relevant test first.
- Run the fast suite before reporting completion.
- Run broader suites when the blast radius is large.
- State skipped tiers + reason.

**The test:** Is the default suite fast enough that people will actually run it?

## 4. Test Failures as First-Class Behavior

**Correctness includes broken environments.**

Test:
- Allocation failures.
- Disk-full conditions.
- Read/write/sync/open/shared-memory failures.
- Network failures/timeouts.
- Interruptions.
- Dependency failures.
- Partial writes/truncation.
- Process crashes/power-loss equivalents.
- Cleanup/recovery failures.

Verify:
- Operation reports controlled error.
- State remains valid.
- Retry/restart behaves correctly.
- Partial work rolls back or safely resumes.
- Resources are released.
- Persistent data is not corrupted.

**The test:** What happens if the next external operation fails?

## 5. Sweep Every Failure Point

**Do not inject one convenient failure. Walk whole operation.**

For an operation with external steps:
- Fail step 1. Verify recovery.
- Fail step 2. Verify recovery.
- Continue until operation completes without injected failure.
- Run transient failure.
- Run persistent failure after first fault.

SQLite applies pattern to allocation + I/O errors. Advance fault point one operation each run until success without simulated failure.

For agents:
- Prefer reusable fault-injection helper over hand-written mocks.
- Record seed/fault index for generated failures.
- Check integrity after the fault is removed.

**The test:** Have you exercised every place this operation can be interrupted?

## 6. Stack Failures

**Recovery paths can fail too.**

Add compound cases:
- Allocation failure during I/O-error handling.
- I/O error during crash recovery.
- Timeout during dependency retry.
- Cancellation during rollback.
- Shutdown during background cleanup.

Use selectively. Compound failures costly; critical recovery code deserves them.

**The test:** Does the error handler assume that everything else works?

## 7. Prove Atomicity After Crashes

**Crash leaves old state or new state. Never broken middle state.**

For durable writes:
- Crash across update points.
- Restart from simulated disk state.
- Verify full commit or full rollback.
- Integrity-check recovery.
- Simulate reordered/buffered/truncated/unsynced writes where storage model allows.

SQLite uses alternative VFS implementations + separate-process crash simulation. Checks journal ordering: database pages must not reach disk before corresponding rollback journal content written + synced.

Also apply to:
- Database migrations.
- File replacement.
- Queue acknowledgements.
- Checkpointing.
- Cache snapshots.
- Distributed workflow state.

**The test:** After a kill at the worst possible moment, is the next startup boring?

## 8. Fuzz Inputs and Persist Discoveries

**Humans write examples. Fuzzers find surprises.**

Fuzz:
- Structured text/binary inputs.
- Serialized data.
- API request bodies.
- Query languages/expressions.
- File formats.
- Protocol messages.
- Related-input combinations when one changes interpretation of another.

Use:
- Grammar-aware generation: syntactically valid nonsense.
- Mutation-based fuzzing for malformed inputs.
- Coverage-guided fuzzing: retain inputs reaching new behavior.
- Property-based testing where invariants are clear.
- Differential testing where an oracle exists.

For interesting fuzz case:
- Minimize it when practical.
- Save seed/corpus entry.
- Add deterministic regression test.
- Rerun retained corpus in normal CI.

SQLite retains historic fuzz cases; reruns curated corpus via `fuzzcheck`. `dbsqlfuzz` mutates SQL + database content together: combined inputs reach states single-input mutation misses.

**The test:** Will this fuzz finding still run after the fuzzer process exits?

## 9. Treat Malformed Persistent Data as Hostile Input

**Files/stored records may corrupt even when your code wrote them.**

Test:
- Truncated files.
- Invalid lengths/offsets.
- Corrupt headers/metadata.
- Modified structural bytes.
- Unknown versions.
- Duplicate/missing/reordered records.
- Valid structure with unexpected content.
- Empty and oversized payloads.

Verify:
- Parsing fails safely.
- Error controlled + useful.
- No buffer overflow, null dereference, infinite loop, runaway allocation.
- Read-only inspection does not make corruption worse.
- Recovery preserves possible data; invents none.

**The test:** Can an attacker, bit flip, or interrupted write turn bad data into unsafe behavior?

## 10. Push Boundaries From Both Sides

**Test edge, one below, one beyond.**

For each limit:
- Test min allowed.
- Test min+1.
- Test typical.
- Test max-1.
- Test max allowed.
- Test max+1.
- Test zero, empty, null, negative, overflow-adjacent where meaningful.

For boolean expressions and bitmasks:
- Make each condition independently alter result.
- Exercise each branch-affecting bit.
- Cover each shared switch arm explicitly.

SQLite uses `testcase()` instrumentation: mark boundaries + boolean-vector cases needing both outcomes.

**The test:** Did you test the last valid value and the first invalid value?

## 11. Turn Every Bug Into a Permanent Test

**Bug not fixed until reproducer stays in suite.**

For every defect:
- Write failing test demonstrating bug.
- Make the smallest code change that fixes it.
- Keep the test after the fix.
- Include issue ID when useful.
- Add case to fuzz corpus if malformed/generated input exposed it.
- Broaden test when same root cause crosses interfaces.

SQLite keeps thousands of regression cases, including tests named after historical ticket identifiers.

**The test:** What prevents this exact bug from returning next year?

## 12. Check Invariants During Every Test Run

**Passing assertion stronger when system checks own assumptions.**

Add runtime checks for:
- Preconditions/postconditions.
- Loop invariants.
- State transitions.
- Lock ownership.
- Resource counts.
- Data-structure integrity.
- Transaction invariants.
- Impossible branches.

Use assertions in test/debug builds. Keep production checks where hostile inputs/data loss make failure practical.

SQLite uses extensive `assert()`, mutex ownership assertions, integrity checks, defensive `ALWAYS()` / `NEVER()`. Defensive conditions document assumptions; become assertion failures during tests.

**The test:** Which assumption would you want to fail loudly during development?

## 13. Detect Resource Leaks Automatically

**Every test leaves environment clean.**

Track:
- Memory.
- File descriptors.
- Sockets.
- Threads/processes.
- Locks/mutexes.
- Transactions.
- Temporary files.
- Timers.
- Background tasks.
- Test DB state.

Make leak checks automatic:
- Run after each test/suite where practical.
- Verify cleanup after success + failure.
- Use sanitizers/leak detectors in CI.
- Treat leaked resources as test failures.

SQLite harnesses auto-detect leaks, including after allocation + disk I/O errors.

**The test:** Does the failure path release everything the success path releases?

## 14. Measure Branches, Not Just Lines

**Executed code != tested code.**

Prefer stronger coverage signals:
- Statement coverage: line executed?
- Branch coverage: decision both ways?
- Condition coverage: did each compound-decision part matter?
- MC/DC where the risk justifies the cost.

Do not chase a number blindly:
- Use coverage to find missing tests.
- Inspect uncovered branches.
- Preserve defensive code protecting assumptions.
- Mark truly unreachable branches explicitly.
- Exclude generated code/irrelevant modules deliberately.

SQLite measures 100% branch coverage + MC/DC for core code in as-deployed configuration. Cost justified: widely deployed infrastructure. Typical apps choose risk-based targets.

**The test:** Did the suite prove each decision, or merely execute each line?

## 15. Test the Tests

**Coverage = meta-test. Mutation testing goes further.**

Use meta-tests:
- Measure branch and condition coverage.
- Break branch; confirm test failure.
- Remove validation; confirm test failure.
- Change boundary; confirm test failure.
- Verify harness reports intentional failure.

Mutation testing asks: if branch wrong, does suite notice?

Be practical:
- Exclude performance-only mutations.
- Review surviving mutations.
- Add tests when surviving mutation reveals missing behavior.
- Simplify when surviving mutation reveals irrelevant branch.

SQLite mutates compiled branch instructions; checks suite catches change. Annotates optimization-only branches: performance branches do not create misleading mutation failures.

**The test:** If the implementation were subtly wrong, would this test suite complain?

## 16. Run Production-Shaped Builds

**Instrumented builds test suite. Production-shaped builds test product.**

Run:
- Debug + assertions.
- Sanitized.
- Coverage.
- Optimized release.
- Actual deployment config.

Compare results:
- Instrumented/release behavior must agree.
- Coverage instrumentation != production verification.
- Debug checks must not hide release-only failures.

SQLite runs coverage builds to validate tests, then recompiles/reruns with delivery options. Both runs must match output.

**The test:** Did you test the artifact you intend to ship?

## 17. Vary Configuration and Platform

**Default config = one point in larger space.**

Test relevant combinations:
- OSes.
- CPU architectures.
- 32-bit/64-bit.
- Endianness where relevant.
- Compilers/versions.
- Feature flags.
- Optional extensions.
- Storage backends.
- Encodings/locales.
- Threading modes.
- Allocation strategies.
- Debug/release settings.

Choose risk-based matrix:
- Run representative subset each change.
- Run the broad matrix before release.
- Add a configuration when a bug exposes a gap.

SQLite exercises platforms, compile options, memory subsystems, mutex modes, journal modes, VFS implementations, encodings, other permutations.

**The test:** Which configuration would be most likely to embarrass this change?

## 18. Disable Optimizations and Compare Answers

**Optimization changes speed, not meaning.**

When possible:
- Run correctness tests with optimizations on.
- Rerun with selected optimizations off.
- Compare outputs.
- Keep separate performance assertions for tests measuring work avoided.

Apply this to:
- Query planners.
- Compilers.
- Caches.
- Memoization.
- Indexes.
- Batch paths.
- Fast paths.

SQLite test-control disables selected query optimizations; verifies unchanged answers.

**The test:** Does the slow path produce the same result?

## 19. Use Dynamic Analysis

**Run tools noticing what assertions miss.**

Use relevant tools:
- Address sanitizers.
- Undefined-behavior sanitizers.
- Thread sanitizers.
- Memory sanitizers.
- Leak detectors.
- Valgrind-like tools.
- Runtime debug allocators.
- Race detectors.

Look for:
- Out-of-bounds.
- Use-after-free.
- Uninitialized reads.
- Double frees.
- Memory leaks.
- Integer overflow.
- Invalid shifts.
- Data races.
- Lock misuse.

SQLite runs Valgrind-style analysis, debug allocation wrappers, mutex assertions, undefined-behavior checks. Also varies implementation-defined compiler behavior: signed vs unsigned `char`.

**The test:** What class of bug can the runtime tool see that your assertions cannot?

## 20. Test Concurrency, Stress, and Performance Separately

**One quiet request proves little.**

Add specialized tests for:
- Multiple threads.
- Multiple processes.
- Repeated open/close.
- Concurrent readers/writers.
- Lock contention.
- Cancellation/shutdown races.
- Long-running workloads.
- Representative-operation performance regressions.

Keep the goals distinct:
- Stress: races/state corruption.
- Soak: rare failures over time.
- Performance: slowdowns.
- Correctness: answers.

SQLite uses dedicated `mptester.c`, `threadtest3.c`, `speedtest1.c` beside primary suites.

**The test:** Does the system remain correct when timing stops being convenient?

## 21. Use Static Analysis With Judgment

**Warnings = signals. Static-analysis pass != proof.**

Run:
- Compiler warnings.
- Type checking.
- Linters.
- Static analyzers.
- Dependency/security checks where relevant.

Then:
- Fix valid findings.
- Investigate suspicious findings.
- Suppress false positives narrowly; explain.
- Do not distort correct code to silence weak warning.
- Static analysis does not replace runtime tests.

SQLite compiles cleanly with strict warnings + static analysis. Developers explicitly treat intensive testing as stronger evidence.

**The test:** Are you fixing a real defect or negotiating with a tool?

## 22. Keep a Human in the Release Loop

**Automation catches failures. Humans notice wrongness.**

Maintain a release checklist:
- Run fast/full suites.
- Run platform/config matrix.
- Run sanitizers/leak checks.
- Run retained fuzz corpora.
- Review unresolved failures/flakes.
- Review performance changes.
- Review migration/rollback.
- Inspect the final artifact.
- Read output, even when every command exits zero.

Evolve the checklist:
- Add item for new failure class.
- Keep historical checklists when release traceability matters.
- Assign long-check ownership.

SQLite uses evolving human-reviewed release checklist: roughly 200 items. Intentionally not fully automated.

**The test:** Did a person ask, "Is this really right?"

## 23. Agent Discipline

**Think first. Keep scope tight. Define proof. Loop until verified.**

Before testing:
- State assumptions. If uncertain, ask.
- Surface multiple interpretations; do not pick silently.
- Name confusion. Stop if ambiguity risks wrong tests.
- Prefer simpler approach. Push back on needless complexity.

While editing:
- Add minimum tests proving requested behavior.
- No speculative helpers, fixtures, abstractions, flexibility, configurability.
- Touch only required tests/code. No adjacent refactors, comment rewrites, formatting churn.
- Match existing test style.
- Remove only orphans created by your change.
- Mention unrelated gaps; do not fix unless asked.

Define success criteria:
- "Add validation" -> invalid-input tests pass.
- "Fix bug" -> reproducer fails before fix, passes after.
- "Refactor" -> relevant tests pass before + after.
- Multi-step task: state `[step] -> verify: [check]`.
- Loop independently until criteria pass.

**The test:** Does each changed line trace to request, and does evidence prove success?

## 24. Agent Workflow

**Turn code changes into verifiable goals.**

When fixing a bug:
```
1. Reproduce the bug with a failing test.
2. Make the smallest fix.
3. Run the reproducer.
4. Run the relevant fast suite.
5. Run broader layers based on risk.
6. Report what ran, what passed, and what was not run.
```

When adding a feature:
```
1. State the contract and invariants.
2. Add normal, boundary, and invalid-input tests.
3. Add failure-path tests where the feature touches external state.
4. Add configuration, concurrency, or fuzz coverage when risk warrants it.
5. Run the relevant fast suite.
6. Report the evidence.
```

When reviewing tests:
- Ask what contract is protected.
- Look for missing boundaries.
- Find failure-path gaps.
- Check test catches plausible wrong implementation.
- Check bug fix has permanent reproducer.
- Check production-shaped code exercised.
- Check if release needs broader tests.

**The test:** Can another engineer see why the change is correct from the evidence you produced?

## How to Know It's Working

Working signals:
- Bugs become permanent regression tests.
- Fast tests run on every meaningful change.
- Fault paths receive happy-path attention.
- Fuzz findings become retained corpus entries.
- Coverage reports lead to better tests, not cosmetic numbers.
- Release builds are tested directly.
- Configuration-specific bugs become rarer.
- Test output reviewed, not collected only.
- Agents report evidence and remaining risk clearly.

## SQLite Patterns Referenced

SQLite testing exceeds public tree:
- **Tcl tests:** primary public dev suite.
- **TH3:** proprietary C harness: embedded targets, branch coverage, MC/DC.
- **SQL Logic Test:** differential queries checked across database engines.
- **`dbsqlfuzz`:** proprietary structure-aware fuzzer mutating SQL + DB content together.

Public-tree examples:
- `test/permutations.test`: fast/full/subsystem/allocator/mutex/encoding/journal/VFS suites.
- `test/malloc_common.tcl`: transient/persistent OOM, I/O, disk-full, shared-memory, open, interrupt fault simulation.
- `test/fuzzcheck.c` reruns retained fuzz cases.
- `test/ossfuzz.c` integrates SQLite with OSS-Fuzz.
- `mptest/mptest.c`, `test/threadtest3.c`, `test/speedtest1.c`: multi-process, multi-thread, performance tests.
- Core: `assert()`, `ALWAYS()`, `NEVER()`, `testcase()` expose assumptions/coverage gaps.

## Source Notes

Reusable agent rules from SQLite practices. SQLite examples checked against:
- [How SQLite Is Tested](https://sqlite.org/testing.html)
- [`test/permutations.test`](https://sqlite.org/src/file?name=test/permutations.test&ci=trunk)
- [`test/malloc_common.tcl`](https://sqlite.org/src/file?name=test/malloc_common.tcl&ci=trunk)
- [`test/fuzzcheck.c`](https://sqlite.org/src/file?name=test/fuzzcheck.c&ci=trunk)
- [`test/ossfuzz.c`](https://sqlite.org/src/file?name=test/ossfuzz.c&ci=trunk)
- [`main.mk`](https://sqlite.org/src/file?name=main.mk&ci=trunk)
