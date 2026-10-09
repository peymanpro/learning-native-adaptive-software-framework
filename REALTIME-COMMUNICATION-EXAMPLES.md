# LNASF Real-Time Communication Examples

**Status:** Verified reference examples of selected LNASF principles  
**Reviewed:** 2026-10-09

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

Configure NEXT_PUBLIC_LNASF_MODE for Next.js/SignalR and REACT_APP_LNASF_MODE for React/Socket.IO. Both default to Passive. Their learned state is held in memory for the current page session.

The tests inject deterministic timestamps or synthetic outcome counts to establish model updates, the separation between prediction and action, mode behavior, fallback, feedback and retry limits. They do not simulate a real network and do not support a real-world reconnect-performance claim.

## Verified GitHub Actions and test coverage

The following runs were verified as successful on the current source changes or their documentation-only successors.

| Repository | Verified checks | Workflow |
| --- | --- | --- |
| Express + Socket.IO | Syntax checks and 10 Node tests, including validation, health and metrics endpoint, model update, operating modes, fallback and a deterministic baseline comparison | [Successful CI](https://github.com/peymanpro/socketio-express/actions/runs/37986262861) |
| NestJS + Socket.IO | TypeScript build and 8 Node tests, including model learning, action separation, fallback and same-trace baseline comparison | [Successful CI](https://github.com/peymanpro/socketio-nestjs/actions/runs/37986269231) |
| ASP.NET Core + SignalR | .NET 8 Release build and 19 xUnit tests, including validation, LNASF modes, fallback and baseline comparison | [Successful CI](https://github.com/peymanpro/signalr-aspnetcore/actions/runs/37986010145) |
| Next.js + SignalR | Lint, syntax checks, 15 Node tests and production build | [Successful CI](https://github.com/peymanpro/signalr-nextjs/actions/runs/37986826784) |
| React + Socket.IO | 2 Jest suites, 10 tests and production build | [Successful CI](https://github.com/peymanpro/socketio-react/actions/runs/37986790513) |

These links are snapshots from GitHub Actions. Later commits or dependency updates can change the current status.

## Dependency security audit snapshot

The JavaScript CI workflows run a non-blocking, package-level npm audit report. Snapshot recorded on 2026-10-09:

| Repository | Audit summary | Finding to prioritize | Interpretation |
| --- | --- | --- | --- |
| Express + Socket.IO | 11 findings: 1 critical, 7 high, 3 moderate | Transitive proxy-addr reports GHSA-jqcg-44mw-7w3h; Engine.IO and Socket.IO parser findings also appear | The report marks fixes available for the listed transitive packages. A dependency update still needs a lockfile change plus tests |
| NestJS + Socket.IO | 34 findings: 1 critical, 16 high, 13 moderate, 4 low | Transitive proxy-addr is critical; the report also flags the current Nest 10 dependency line and old build tooling | Several suggested fixes require a major Nest upgrade, so this is not a safe one-step audit fix |
| Next.js + SignalR | 16 findings: 1 critical, 13 high, 1 moderate, 1 low | The direct next dependency is pinned to 16.2.4 and falls into multiple reported advisory ranges, including GHSA-8h8q-6873-q5fj | npm audit reports next 16.4.0 as a same-major candidate; the manifest and lockfile still require a controlled update and a full CI run |
| React + Socket.IO | 94 findings: 3 critical, 74 high, 12 moderate, 5 low | The direct react-scripts 5.0.1 toolchain has a broad legacy dependency tree; critical transitive findings include proxy-addr, shell-quote and websocket-driver | This likely needs a controlled build-tool migration or a separately reviewed dependency plan, not an automatic force upgrade |

These are advisory database findings, not a claim that every package is exploitable in this application's runtime. The audit workflow logs record each package's direct/transitive status and suggested fix metadata. The repositories remain useful as engineering samples, but **do not treat the JavaScript dependency posture as production-release clearance** until the findings are triaged, fixes are applied deliberately, and tests are rerun. No automated dependency fix or major migration was applied as part of the LNASF integration.

References: [proxy-addr advisory](https://github.com/advisories/GHSA-jqcg-44mw-7w3h), [Engine.IO advisory](https://github.com/advisories/GHSA-2gc4-cqfq-p2gv), [Socket.IO parser advisory](https://github.com/advisories/GHSA-2m8v-j782-fhvr), [Next.js advisory](https://github.com/advisories/GHSA-8h8q-6873-q5fj).

## Limitations

- Learning is local and ephemeral. The examples do not persist models across restarts or implement shared multi-instance learning.
- The server-side typing model optimizes a proxy—the number of duplicate start notifications—not measured user-perceived quality or latency.
- The reconnect model observes outcomes after selected delays; success/failure is not proof that a delay caused the result. Network conditions and timing can confound observations.
- The current suites are unit-level tests. They are not live multi-client integration tests, load tests, chaos tests or statistically valid performance benchmarks.
- Candidate delays, cooldown bounds and policy thresholds are fixed. The model does not learn arbitrary parameters and cannot override the policy.
- Autonomous mode is intentionally not implemented in these examples.
- Dependency audit summaries still report advisories in the JavaScript projects. The counts are documented separately in the repository CI logs; each finding needs package-level triage before production release.

These repositories demonstrate real native learning-to-decision paths and bounded actions for selected runtime problems. They do not represent the full LNASF framework, and no performance improvement is claimed without an appropriate benchmark.
