# LNASF Real-Time Communication Examples

**Status:** Verified reference examples of selected LNASF principles  
**Reviewed:** 2026-10-10

This document records five intentionally limited integrations of the Learning-Native Adaptive Software Framework (LNASF) in real-time client and server repositories. These are reference examples of selected principles, not complete implementations of every framework mode or capability.

## Shared design rule

The components make this loop explicit:

Observe → Learn → Predict → Decide → Adapt → Measure → Learn

Prediction is separated from the decision to act. Passive mode learns from observations without changing behavior. Advisory mode exposes a recommendation but does not apply it. Adaptive mode may apply only the action allowed by a deterministic policy. All examples default to Passive mode.

Authentication, room membership checks, input validation, delivery of primary chat messages, and other security-sensitive behavior remain deterministic. Learned predictions cannot authorize or bypass them.

## Implementation matrix

| Repository | Host runtime and role | Observation and native model | Bounded adaptive action | Fallback and measurement |
| --- | --- | --- | --- | --- |
| [socketio-express](https://github.com/peymanpro/socketio-express) | JavaScript, Express and Socket.IO server | Learns inter-arrival gaps between typing-start events and estimates the probability of a fast burst | In Adaptive mode, suppresses duplicate typing-start notifications inside a policy-selected 150–500 ms window | Cold start or weak evidence broadcasts normally. GET /lnasf/metrics reports observations, prediction, decision, event counts and suppression rate |
| [socketio-nestjs](https://github.com/peymanpro/socketio-nestjs) | TypeScript, NestJS Socket.IO gateway | Uses a host-native frequency model over typing-start gaps | Only repeated typing-start notifications are eligible for suppression | Typing-stop and chat messages always pass through. GET /lnasf/metrics exposes model and action counters |
| [signalr-aspnetcore](https://github.com/peymanpro/signalr-aspnetcore) | C#, ASP.NET Core 8 and SignalR Hub | Uses a native C# online gap-frequency model with an explicit decision policy | Only duplicate typing-start notifications are eligible for suppression | Thread-safe service state, deterministic broadcast fallback, and GET /lnasf/metrics; primary chat messages remain unaffected |
| [signalr-nextjs](https://github.com/peymanpro/signalr-nextjs) | TypeScript learning module in a Next.js SignalR client | Records retry success/failure by selected delay, then estimates smoothed success probability, confidence and delay-penalized utility | Adaptive mode can choose a better-supported delay from the fixed set 0, 2,000, 5,000 and 10,000 ms | At most four retries or 30 seconds per episode. Passive and Advisory retain the deterministic schedule. The UI shows runtime diagnostics |
| [socketio-react](https://github.com/peymanpro/socketio-react) | JavaScript, React and Socket.IO client | Learns retry outcomes per delay and compares candidate utility with the deterministic schedule | A bounded retry scheduler chooses a learned delay only when the policy's evidence and utility gates pass | Initial connect is immediate; at most four retries or 30 seconds per episode. Passive and Advisory do not change the baseline schedule |

## Learning model and policy

### Server-side typing burst model

The three backend examples observe the time gaps between typing-start notifications. Their smoothed estimate is:

P_fast = (N_fast + 1) / (N + 2)

Here N is the number of observed gaps and N_fast counts gaps no longer than 500 ms. The confidence value N / (N + 3) is an evidence heuristic, not a calibrated statistical confidence interval.

The policy requires at least five gap observations and confidence of at least 0.60. It selects a 150, 300 or 500 ms candidate cooldown from the predicted burst probability. Adaptive mode may suppress a duplicate start only when the same connection is already typing and its previous forwarded start falls within the selected window. Typing-stop is always forwarded.

The test suites run an identical deterministic trace through Passive and Adaptive modes and verify that Adaptive mode emits fewer start notifications for this synthetic burst. This is a count-level unit test. It does not establish better typing accuracy, lower perceived latency or improved end-user experience.

Configuration is LNASF_MODE=passive, advisory or adaptive. Passive is the default. Server model state is process-local and resets on restart.

### Client-side reconnect outcome model

The reconnect examples record a successful retry outcome when a retry connects and record failure evidence when another retry is requested. For each permitted delay, the model estimates:

- Smoothed success probability: (successes + 1) / (attempts + 2)
- Evidence heuristic: attempts / (attempts + 2)
- Utility: predicted success probability minus a small normalized penalty for longer waits

A candidate is eligible only with at least three observations, confidence of at least 0.60, estimated success probability of at least 0.55, and at least 0.05 estimated utility gain over the current deterministic baseline. These are explicit experimental defaults, not statistically optimized thresholds.

The only candidate delays are 0, 2,000, 5,000 and 10,000 milliseconds. Retry scheduling is capped at four attempts or 30 seconds. In Passive mode, the model learns while leaving the schedule unchanged. Advisory reports the recommendation without applying it. Adaptive may select a candidate only when the policy gates pass. An intentional leave or unmount is not treated as evidence that a retry failed.

Configure NEXT_PUBLIC_LNASF_MODE for Next.js/SignalR and VITE_LNASF_MODE for React/Socket.IO. Both default to Passive. Their learned state is held in memory for the current page session.

The tests inject deterministic timestamps or synthetic outcome counts to establish model updates, the separation between prediction and action, mode behavior, fallback, feedback and retry limits. They do not simulate a real network and do not support a real-world reconnect-performance claim.

The retry classifiers inspect both direct status fields and common nested transport shapes (`data`, `description`, and `response`). Known permanent HTTP statuses stop the episode without being recorded as a delay-specific failure. In the React/Socket.IO client, `io server disconnect` and `io client disconnect` are also classified as non-retryable disconnect reasons; a server-forced disconnect is surfaced to the user and requires an explicit manual retry. Terminal diagnostics retain the actual retry index and elapsed episode duration for troubleshooting.

## Verified GitHub Actions and test coverage

The following runs were verified as successful on the current source changes or their documentation-only successors.

| Repository | Verified checks | Workflow |
| --- | --- | --- |
| Express + Socket.IO | Syntax checks and 12 Node tests, including model/policy tests and a live two-client Engine.IO polling integration that verifies adaptive duplicate-start suppression while typing-stop and primary chat messages still pass through | [Successful CI after dependency remediation](https://github.com/peymanpro/socketio-express/actions/runs/37996981426) |
| NestJS + Socket.IO | NestJS 12/TypeScript 6 build and 9 Node tests, including model/policy checks and live two-client Socket.IO transport integration for adaptive typing suppression, typing-stop, and primary chat delivery | [Successful CI after dependency remediation](https://github.com/peymanpro/socketio-nestjs/actions/runs/37997132144) |
| ASP.NET Core + SignalR | .NET 8 Release build (0 warnings/errors) and 20 xUnit tests, including a live two-client WebSocket/SignalR Hub integration verifying adaptive typing suppression while stop and primary messages remain deliverable | [Successful CI](https://github.com/peymanpro/signalr-aspnetcore/actions/runs/37991401270) |
| Next.js + SignalR | Next.js 16.4.0 / React 19.3.0; independent ESLint 9; 15 unit tests plus a live two-connection integration with the actual ASP.NET Core Hub covering join, message, and typing events | [Successful main-branch live-Hub CI](https://github.com/peymanpro/signalr-nextjs/actions/runs/37997037738) |
| React + Socket.IO | Migrated from Create React App to Vite/Vitest; ESLint 9, 15 passing unit tests (one environment-gated integration test is skipped in the unit run), bounded 1,000-message history, plus a separate passing live test with two rendered clients integrated with the actual Express/Socket.IO backend for join, typing, and message delivery | [Successful live-client CI](https://github.com/peymanpro/socketio-react/actions/runs/37997301550) |

These links are snapshots from GitHub Actions. Later commits or dependency updates can change the current status.

## Dependency security audit snapshot

The JavaScript CI workflows emit a package-level npm audit report and then enforce `npm audit --audit-level=high`; high and critical findings fail CI, while lower-severity findings remain visible in the report. The original snapshot was recorded on 2026-10-09; the table below records post-remediation `main` CI snapshots verified on 2026-10-09/10.

| Repository | Audit summary | Finding to prioritize | Interpretation |
| --- | --- | --- | --- |
| Express + Socket.IO | 0 findings | Lockfile refreshed; `nodemon` removed in favor of native Node watch mode; `qs` pinned to a compatible patched range | Clean npm audit snapshot, syntax checks and 12 tests including live transport passed |
| NestJS + Socket.IO | 0 findings | Upgraded to NestJS 12 and TypeScript 6; refreshed lockfile and made compiler output settings explicit | Clean npm audit snapshot, build and all 9 tests including live transport passed |
| Next.js + SignalR | 0 findings | Next.js 16.4.0 and React/React DOM 19.3.0; independent ESLint 9 with React, Hooks and JSX accessibility rules replaces the retired `eslint-config-next` chain | Remediation CI passed clean install, lint, syntax checks, 15 tests, production build and npm audit. Trade-off: Next-specific `@next/eslint-plugin-next` rules are not enabled; the general React/accessibility rule set remains |
| React + Socket.IO | 0 findings | Replaced Create React App / `react-scripts@5` with Vite/Vitest and regenerated the lockfile; message history is capped at 1,000 entries | Clean npm audit snapshot, ESLint, 13 tests and production build passed |

These are advisory database findings, not a claim that every package is exploitable in this application's runtime. After updates, all four JavaScript projects have clean current npm audit snapshots. The Next.js-specific lint preset and its vulnerable transitive chain were removed in favor of explicit React/Hooks/accessibility linting. The dependency changes were merged only after remediation workflows and main-branch CI passed; no `--force` upgrade was used. React/Next-specific lint coverage is narrower without `@next/eslint-plugin-next`, a deliberate trade-off recorded in the project README.

References: [Historical proxy-addr advisory](https://github.com/advisories/GHSA-jqcg-44mw-7w3h), [Historical Engine.IO advisory](https://github.com/advisories/GHSA-2gc4-cqfq-p2gv), [Historical Socket.IO parser advisory](https://github.com/advisories/GHSA-2m8v-j782-fhvr), [Historical Next.js advisory](https://github.com/advisories/GHSA-8h8q-6873-q5fj), [Historical `braces` advisory in the retired lint chain](https://github.com/advisories/GHSA-vfj7-8cjw-p6xm).

## Limitations

- Learning is local and ephemeral. The examples do not persist models across restarts or implement shared multi-instance learning.
- The server-side typing model optimizes a proxy—the number of duplicate start notifications—not measured user-perceived quality or latency.
- The reconnect model observes outcomes after selected delays; success/failure is not proof that a delay caused the result. Network conditions and timing can confound observations.
- Express and NestJS include live two-client Socket.IO transport integration tests; the ASP.NET Core Hub also has a live Hub integration suite. The Next.js client now connects two real SignalR clients to that Hub, and the React client test mounts two React instances against the live Express backend through Socket.IO. These are transport/component integration tests, not full browser automation. The family has no load tests, chaos tests, or statistically valid real-network performance benchmark.
- Candidate delays, cooldown bounds and policy thresholds are fixed. The model does not learn arbitrary parameters and cannot override the policy.
- Autonomous mode is intentionally not implemented in these examples.
- Current npm audit snapshots are clean for Express, NestJS, Next.js and React. Both frontend clients now have live transport integration tests, while full browser-driven UI tests, load tests, chaos tests, and statistically valid real-network performance benchmarks remain outstanding.

These repositories demonstrate real native learning-to-decision paths and bounded actions for selected runtime problems. They do not represent the full LNASF framework, and no performance improvement is claimed without an appropriate benchmark.
