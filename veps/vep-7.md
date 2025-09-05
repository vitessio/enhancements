# VEP-7 AI Assistance Disclosure

```
VEP: 7
Title: AI Assistance Disclosure
Author: Nick Van Wiggeren
Reviewers: Dirkjan Bussink, Harshit Gangal, Arthur Schreiber, Matt Lord, Derek Perkins, Florent Poinsard
Status: Approved
Created: 2025-08-25
```

## Abstract

As AI tools become common in software development, establishing transparency about their use in Vitess contributions helps maintainers understand the context of pull requests and fosters respectful collaboration.

## Motivation

AI assistance varies widely in scope and quality. When contributors disclose how AI tools were used, it provides valuable context that helps reviewers understand the work and offer more targeted feedback. This transparency is fundamentally about respect for the humans reviewing the code and creating an environment of open collaboration.

There are many members of the team that use these tools daily, and the motivation is not to make it more difficult to do so. At this time, the quality produced by AI tooling can vary heavily, so it's important to acknowledge that and apply the appropriate level of scrutiny.

## Disclosure Guidelines

### When to Disclose

Contributors should disclose AI assistance when it was used for:

- Code generation or substantial modification
- Writing documentation or comments
- Creating tests
- Drafting pull request descriptions or responses

### How to Disclose

A simple note in the pull request description is sufficient. Examples:

```
AI Assistance: This PR was written primarily by Claude Code.
```

```
AI Assistance: I used ChatGPT to understand the codebase structure, but implemented the solution manually.
```

```
AI Assistance: GitHub Copilot suggested completions for standard SQL patterns in the test files.
```

### What Doesn't Need Disclosure

Basic IDE features like single-word autocomplete, spell check, and standard linting suggestions don't require disclosure. Using AI to review human-generated code in the process of writing a PR does not require disclosure.

## Rationale

It's required that PR authors understand the code that they're proposing, and take responsibility for responding to feedback. Using AI tools does not absolve humans from that expectation.

By encouraging disclosure, we aim to create mutual understanding between contributors and maintainers. Contributors can use AI tools productively while maintainers gain context about the work they're reviewing. The goal is building a system that makes code review more effective and collaborative.

## Implementation

This guideline applies to new pull requests. Failure to disclose will not automatically invalidate a pull request, but maintainers reserve the right to request this context.
