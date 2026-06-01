# dotmds

A collection of Markdown guides for agentic software development.

## Guides

### Testing

#### [`testGuide.md`](./testGuide.md)

A practical testing guide for coding agents, inspired by SQLite's testing philosophy and current test suite.

Use it as an instruction file when an agent writes code, fixes bugs, reviews tests, or reports verification results. Its central rule is simple: every change must be tested. Risk determines how broad the verification should be, not whether verification happens.

The guide covers:
- Testing observable contracts instead of incidental implementation details.
- Using focused, fast, full, soak, and release test tiers.
- Injecting allocation, I/O, disk-full, interruption, and recovery failures.
- Verifying crash safety, atomicity, malformed input handling, and resource cleanup.
- Retaining fuzz discoveries as deterministic regression tests.
- Testing boundaries, configurations, concurrency, optimizations, and production-shaped builds.
- Using coverage, mutation testing, sanitizers, static analysis, and human release checklists with judgment.
- Surfacing assumptions, keeping edits surgical, avoiding speculative test infrastructure, and defining success criteria.
- Reporting what was tested, what passed, and what remains unverified.

Tradeoffs:
- More rigorous testing increases development time and CI cost.
- Fault injection, fuzzing, soak tests, and configuration matrices require dedicated tooling and maintenance.
- SQLite-level depth is valuable for critical paths but can be excessive for low-risk changes.
- Agents must scale the breadth of testing to the risk without skipping focused verification.
- Coverage metrics can create false confidence when they are treated as targets instead of signals.

#### [`testExamples.md`](./testExamples.md)

A companion document with concrete patterns adapted from SQLite's public tests.

Use it when principles alone are too abstract. Each section starts with a weak testing approach, explains what it misses, and shows a stronger pattern an agent can apply in other codebases.

The examples cover:
- Keeping a permanent regression test for every fixed bug.
- Sweeping I/O failures across an entire persistent-state operation.
- Testing both transient and persistent faults.
- Proving crash recovery with state signatures.
- Mutating valid stored data to test corruption handling.
- Preserving fuzz discoveries as normal CI tests.
- Comparing optimized behavior against a simpler baseline.
- Testing the last valid value and the first invalid value.
- Checking that failure paths release resources.

Tradeoffs:
- The examples are patterns, not drop-in test code for every project.
- Several examples depend on SQLite-specific Tcl helpers and test fixtures.
- Simplified examples omit some infrastructure details to keep the underlying idea clear.
- Applying every pattern to every change would create unnecessary test complexity.
- Teams still need project-specific examples for UI, API, service, and deployment behavior.

## Usage

Add the relevant Markdown file to your agent instructions or project documentation.

Start with [`testGuide.md`](./testGuide.md). Use [`testExamples.md`](./testExamples.md) when an agent needs concrete testing patterns.

## Sources

The testing documents reference SQLite's public documentation and source tree:

- [How SQLite Is Tested](https://sqlite.org/testing.html)
- [SQLite Source Repository](https://sqlite.org/src/)
