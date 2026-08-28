---
name: Validate Nissan integration
description: "Validate current NissanConnect integration changes with the narrowest relevant check, the complete test suite, editor diagnostics, and diff checks. Use before finishing or reviewing an implementation."
argument-hint: "Optional changed file, platform, or pytest target"
agent: agent
---

Validate the current NissanConnect integration changes. Treat invocation arguments as an optional file, platform, behavior, or pytest target that narrows the scope.

1. Read the repository [guidelines](../../AGENTS.md). Inspect the explicit target when provided; otherwise inspect `git status --short` and the focused diff to determine the affected platform or shared component. Preserve existing user changes and keep any repairs within the validated behavior.
2. Choose and run the cheapest check that can falsify the changed behavior:
   - Map a platform change to `python -m pytest tests/test_<platform>.py`.
   - Map config-flow behavior to `python -m pytest tests/test_config_flow.py`.
   - For a shared component or unclear blast radius, select the smallest defensible test set; use the full suite directly only when no narrower executable check exists.
   - For documentation or customization-only changes, run relevant editor diagnostics and structural checks instead of an unrelated platform test.
3. If the narrow check cannot start because test dependencies are missing, install them with the command documented in the repository guidelines and rerun the exact same check.
4. When the narrow check fails, diagnose the local root cause, make the smallest repair consistent with existing patterns, and rerun that same check before widening scope. Continue until it passes or a genuine external blocker or clearly unrelated pre-existing failure is proven.
5. Once the narrow check passes, run `python -m pytest tests`. Repair regressions caused by the current changes, rerun their narrow check, and then rerun the complete suite until it passes or a genuine blocker remains.
6. Check editor diagnostics for every changed text file. Run `git diff --check` and explicitly include untracked text files because the normal diff omits them. Inspect the focused diff for unrelated changes, accidental identity changes, and polling-versus-update regressions. Repair relevant findings and repeat the affected checks.
7. For entity changes, also apply the completion criteria from the [add-nissan-entity skill](../skills/add-nissan-entity/SKILL.md), including capability gating, translations, README coverage, stable unique IDs, and absence of network calls from state properties.
8. Do not claim that Hassfest or the HACS validator passed unless they were actually run. Their presence in CI is not a local validation result.

Report only:

```text
Validation: PASS | FAIL | BLOCKED
Scope: <target or inferred changed area>
Narrow check: <command or diagnostic> - <result>
Full suite: <command> - <result, skipped, or not reached>
Diagnostics/diff: <result>
Repairs: <files or behavior repaired, or none>
Notes: <actionable failures, environment mismatch, or omitted CI-only checks>
```