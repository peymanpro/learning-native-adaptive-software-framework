# Learning-Native Adaptive Software Framework (LNASF)

**Technical Concept and Architecture Specification - Version 0.1**

**Author:** Peyman Salimi  
**Date:** 2026-08-28  
**Status:** Technical disclosure / concept specification

## About

LNASF is a proposed architectural framework for building reusable software components that can observe runtime behavior, learn from experience, make constrained predictions and decisions, and adapt selected aspects of their behavior while retaining a deterministic fallback.

The central loop is:

`Observe -> Learn -> Predict -> Decide -> Adapt -> Measure -> Learn`

The framework is intended to be applicable to libraries, frameworks, runtimes, developer tools, frontend systems, backend systems, and infrastructure.

## Important Positioning

LNASF does **not** claim to invent machine learning, self-adaptive software, online learning, runtime adaptation, MAPE-K, predictive prefetching, or learning-enabled components. Those areas have substantial prior art.

The document records a specific proposed architectural formulation centered on:

- learning as an explicit reusable software-component capability;
- native learning within the host ecosystem;
- scratch implementations of learning mechanisms for reference implementations;
- explicit separation of prediction and decision;
- confidence and utility-aware adaptation;
- deterministic fallbacks;
- constrained runtime adaptation;
- applicability across multiple software layers and programming languages.

## Reference Implementation

The [LNASF Adaptive Admin Console](https://github.com/peymanpro/angular-standalone-admin-panel) is an application-level reference implementation of a **limited subset** of these principles. It uses a native TypeScript transition-frequency model, an explicit decision policy, constrained predictive route preloading, a deterministic fallback, and runtime counters.

This implementation is an engineering demonstration—not a claim that the full LNASF framework is complete or that adaptive preloading has already produced a measured performance gain. Its README documents the architecture, tests, evaluation protocol, and current limitations.

## Real-Time Communication Examples

The [LNASF real-time communication examples](REALTIME-COMMUNICATION-EXAMPLES.md) document five limited integrations across Express, NestJS, ASP.NET Core, Next.js and React. They make host-native learning, explicit prediction/decision separation, bounded adaptation, deterministic fallback, and runtime measurement executable. The document links the relevant code/test areas through each repository's README and records verified CI evidence. It distinguishes synthetic unit-level checks from real-world performance claims; these are selected reference examples, not complete LNASF implementations.

## Files

- `SPECIFICATION.md` - editable specification with LaTeX math.
- `LNASF-v0.1.tex` - LaTeX source used to produce the PDF.
- `LNASF-v0.1-Technical-Specification.pdf` - rendered publication version.
- `references/` - bibliography source.
- `metadata/CITATION.cff` - citation metadata.
- `metadata/SHA256SUMS.txt` - integrity hashes for the published files.
- `CHANGELOG.md` - version history.

## Citation

Salimi, Peyman. (2026). *Learning-Native Adaptive Software Framework (LNASF): Technical Concept and Architecture Specification*, Version 0.1.
