# Supply Chain Security

End-to-end implementation of the seven layers in the design doc, plus one targeted addition — plpgsql security scanning for `scripts/psql/` — to close a coverage gap in the SAST layer.

## Scope

### Implementing as written

- **Renovate** — tiered cooldown policy + Dependency Dashboard
- **OSV-Scanner** — daily lockfile scan against vulnerability database
- **Socket.dev** — real-time malicious-package detection on PRs
- **Semgrep** — PR-time SAST on application code (TypeScript, Python, OWASP, AI packs)
- **Gitleaks** — broader secret scanning on PRs + weekly history scan
- **Trivy** — container image + IaC scanning in the GKE build pipeline
- **SBOM** — CycloneDX-format manifest per release tag
- **Alert routing** — Slack `#security-alerts` + auto-opened GitHub issues + weekly digest

### Adding (new)

- **plpgsql security scan** for `scripts/psql/` — audit + CI guardrail (~3 days)

## Why add plpgsql security scanning

### Gap

- Semgrep covers application code via built-in rule packs (TypeScript, Python, OWASP, AI) but does not parse Postgres procedural language (plpgsql)
- No equivalent open-source SAST tool exists for plpgsql
- The `scripts/psql/` tree is invisible to the SAST layer as currently designed

### Why it matters

- `scripts/psql/` contains 2,373 `EXECUTE` statements and 1,498 `SECURITY DEFINER` declarations
- `SECURITY DEFINER` functions run with the function owner's privileges and bypass Row-Level Security
- An injection bug in such a function is a privilege-escalation bug, not a contained one

### Concrete unsafe patterns already in the codebase

- `copy_and_anonymized_data.sql` — concatenates `new_table_name`, `view_name`, `insert_query`, `batch_size`, `offset_count` directly into `EXECUTE` statements with no quoting
- `chat_bot_process_query.sql:253`, `chat_bot_process_query_async.sql:245`, `main_query_processing.sql:124` — `EXECUTE ('SELECT ' || api_call_query)` adjacent to chatbot user-input flow

## Deliverable

Four artifacts get created in the repo:

1. **Audit document** — a list of every plpgsql function in `scripts/psql/` that builds queries at runtime, with file path, line number, and one of three tags: ✅ safe, 🟡 needs review, 🔴 has injection risk
2. **CI workflow** — runs automatically on every PR that touches a `.sql` file
3. **Scanner script** — the small custom code that does the pattern matching (Semgrep doesn't parse plpgsql, so this is a ~50-line shell/python script)
4. **Allowlist file** — known-safe patterns to skip, each with an inline comment explaining why

The CI check flags three specific patterns:

- `EXECUTE` statements built by string concatenation (e.g. `EXECUTE 'SELECT ' || user_input`)
- `format()` calls used for SQL that skip the safe placeholders — `%I` for table/column names, `%L` for values
- New functions declared `SECURITY DEFINER` (gets routed for explicit reviewer approval, since these run with elevated privileges)

When the CI check fires, the developer sees: which file, which line, which pattern matched, and how to fix it. The PR cannot merge until the issue is resolved or added to the allowlist with justification.

## Out of scope

Fixing the existing issues that the audit document surfaces. That is a separate follow-up project — the same pattern the design doc uses for gitleaks history-scan findings (find first, fix as a separate workstream).

---

# Secret Scanning Hardening

End-to-end implementation of the secret-scanning layer in the design doc — broader-coverage detection on new commits and weekly visibility into historical leaks.

## Scope

Implementing as written:

- **Gitleaks CI workflow** — runs on every PR against the full diff, scans all file types
- **Weekly history scan** — scheduled gitleaks run against the entire main branch history
- **`.gitleaks.toml` allowlist** — suppresses known false positives identified in the initial history scan
- **Existing `auto_env_secret_scan.yml` preserved** — LLM-based redundant check for `.env`-specific nuance (real value vs placeholder)

---

# Phased Timeline

| Phase | Timeline | Deliverables |
|-------|----------|--------------|
| 0 |  | Dependabot alerts enabled; Slack channel + webhook confirmed; reviewer team confirmed |
| 1 |  | OSV-Scanner cron + Slack digest + issue automation + weekly posture digest + Socket.dev install |
| 2 |  | Renovate setup with tiered cooldown; one-week dry-run; flip to PR mode |
| 3 |  | Gitleaks + Semgrep + plpgsql audit + CI check |
| 4 |  | Trivy in build pipeline; SBOM per release |
| 5 |  | Documentation in `docs/specs/observability/`; SOC2 evidence bundling; retrospective |
| 6 |  | Land `.github/workflows/gitleaks.yml` (PR + history scan modes); run initial full-history scan |
| 7 |  | Triage initial history scan output → populate `.gitleaks.toml` allowlist; flip CI from warning-only to blocking |
