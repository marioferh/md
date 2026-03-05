# PR Comments Summary: Prometheus Config API

**Scope:** Comments extracted and organized from the following pull requests:

| PR | State | Link |
|----|--------|------|
| [#2653](https://github.com/openshift/api/pull/2653) | Open | Add prometheusConfig API |
| [#2463](https://github.com/openshift/api/pull/2463) | Closed | MON-4030: prometheus k8s config API |
| [#2630](https://github.com/openshift/api/pull/2630) | Closed | Add prometheusConfig API |
| [#2652](https://github.com/openshift/api/pull/2652) | Closed | Add prometheusConfig API |

**Saved PR content (agent-tools):** Raw timeline/content used for this summary is stored under the project's `agent-tools` folder so it can be re-used without re-fetching:

| PR | File (relative to agent-tools) | Notes |
|----|--------------------------------|-------|
| #2653 | `57b01a20-853e-486e-8ff0-9e4fc86a3056.txt` | Open PR; full timeline. |
| #2463 | `73a6375c-b120-4767-83fc-f362a6c75db6.txt` | Closed; largest (6308 lines). |
| #2630 | `57fa43aa-02c1-4fce-a426-3dc504fb404d.txt` | Closed; 4034 lines (171.5 KB). |
| #2652 | `pr-2652-openshift-api.txt` | Closed; short timeline (~170 lines), saved after retry. |

---

## [Executive summary](#executive-summary)

The high number of comments comes from (1) **very large PRs** (size/XXL, thousands of lines across types, CRDs, OpenAPI, and tests), (2) **multiple review cycles** across four PRs (2463 → 2630 → 2652 → 2653) with 2463 and 2630 becoming hard to load, (3) **heavy automated feedback** from CodeRabbit (actionable comments, nitpicks, and many "Additional comments" / LGTM-style items), and (4) **many small topics** (validation, docs, types, defaults, remote write, relabel, TLS, volumeClaimTemplate, etc.). Human review from **everettraven** (and JoelSpeed, danielmellado, simonpasquier) drove most of the design decisions; CodeRabbit repeated similar points across CRD variants and files. This document groups feedback **by theme** so a follow-up AI or human can see what was asked per topic and what was resolved.

---

## [1. volumeClaimTemplate / PVC type](#1-volumeclaimtemplate-pvc-type)

**Status:** Resolved (in current code).

**Summary:** Whether to use a custom `VolumeClaimConfig` (sizeInGiB + storageClassName) or `*v1.PersistentVolumeClaim`. If v1alpha1 had already shipped with one type, changing it would be a **serialization-breaking change** requiring v1alpha2.

- **[PR #2630]** **everettraven** (Dec 17, 2025): If changing the type changes serialization, it is *not* OK in v1alpha1 if the API has shipped. Older clients would fail to serialize. The correct approach is to up-version from v1alpha1 to v1alpha2. Re: splitting into smaller changes, leaving that to the author; this PR is large but manageable.
- **[PR #2630]** **danielmellado** (Dec 17, 2025): verify-crd-schema and verify-crdify fail because we intentionally changed volumeClaimTemplate from full `corev1.PersistentVolumeClaim` to `VolumeClaimConfig` for better admission-time validation and to avoid exposing ~40 PVC fields. As TechPreview v1alpha1, breaking changes are allowed; both tools say override is appropriate for intentional changes. Asking everettraven for another proposal; plan to split into smaller changes.
- **[PR #2630]** CodeRabbit (multiple): Noted that VolumeClaimConfig is well-designed and consistent across Alertmanager and Prometheus; some comments later clarified that this is a *new* API introduction (no prior shipped type), so no version bump needed.
- **Resolution (from conversation summary):** In PR #2653, `volumeClaimTemplate` was reverted to `*v1.PersistentVolumeClaim` to avoid breaking v1alpha1 and stay consistent with existing API surfaces.

---

## [2. RelabelConfig / RelabelAction](#2-relabelconfig-relabelaction)

**Status:** Resolved (enum with 11 actions; RelabelActionConfig with type + configs per action). Optional: document that RE2 regex is validated at runtime.

**Summary:** RelabelAction enum must include all Prometheus actions referenced in XValidation and docs (Replace, Keep, Drop, HashMod, LabelMap, LabelDrop, LabelKeep, **Lowercase, Uppercase, KeepEqual, DropEqual**). CEL rule had a malformed string (non-ASCII quote). OpenAPI/CRD enum and descriptions must align.

- **[PR #2653]** **coderabbitai[bot]** (Jan 16, 2026): RelabelAction enum is missing Lowercase, Uppercase, KeepEqual, DropEqual; update enum and RelabelConfig.action/targetLabel docs so enum and XValidation align in types_cluster_monitoring.go, OpenAPI, and all three payload CRDs.
- **[PR #2630]** **coderabbitai[bot]** (Jan 9, 2026): LogLevel constant comments (LogLevelWarn, LogLevelInfo, LogLevelDebug) should be full sentences without leading commas; CEL rule for targetLabel had malformed quoting (e.g. `self.targetLabel != "`); fix with ASCII empty-string check and correct YAML quoting in DevPreviewNoUpgrade and TechPreviewNoUpgrade CRDs.
- **[PR #2630]** CodeRabbit: RelabelConfig regex field is not validated at admission time (invalid RE2 only fails at runtime); consider documenting that regex validation is at runtime or adding operator-level validation.
- **[PR #2463]** CodeRabbit: Add validation for Prometheus label names in `sourceLabels` (e.g. `^[a-zA-Z_][a-zA-Z0-9_]*$`); same for `Label.key` with CEL or pattern.

---

## [3. AuthorizationConfig / TLSConfig / SecretKeySelector](#3-authorizationconfig-tlsconfig-secretkeyselector)

**Status:** Resolved (TLS cert/key CEL; union with discriminator; SecretKeySelector has no optional field in this API).

**Summary:** Discriminated union for auth (BearerToken, etc.), cert/key both present or both absent for mTLS, enum for certificateVerification and SecretKeySelector.optional. Schema and docs should enforce these.

- **[PR #2630]** CodeRabbit (Dec 17): TLS config has minProperties: 1 but doesn't enforce cert and key together; suggest CEL rule `(has(self.cert) && has(self.key)) || (!has(self.cert) && !has(self.key))`.
- **[PR #2463]** CodeRabbit: Document the `optional` field in SecretKeySelector (Required/Optional); consider enum + fail-closed handling for SecretKeySelector.optional and TLSConfig.certificateVerification in OpenAPI.
- **[PR #2653]** CodeRabbit (LGTM-style): TLSConfig XValidation and MinProperties correctly enforce mTLS; AuthorizationConfig uses proper union pattern with discriminator and CEL.

---

## [4. RemoteWrite](#4-remotewrite)

**Status:** Resolved (Headers as `[]PrometheusRemoteWriteHeader`; listMapKey=name; auth/TLS in RemoteWriteSpec; name documented when omitted).

**Summary:** Headers (map vs `[]PrometheusRemoteWriteHeader`), queue/duration fields (explicit units e.g. seconds vs duration strings), URL uniqueness (listMapKey: url vs name), metadataConfig and queue-parameter docs. RemoteWriteSpec auth/TLS was missing in early feedback; later added.

- **[PR #2630]** CodeRabbit (Jan 9): Consider whether **url** is the right listMapKey for remoteWrite: same URL with different configs (e.g. relabeling, auth) would be merged by SSA. Suggest **name** as listMapKey with url required but not unique; document if one-config-per-URL is intentional.
- **[PR #2463]** CodeRabbit: RemoteWriteSpec description says "including URL, authentication, and relabeling" but schema only had URL, name, timeout, relabel configs—no auth/TLS; add auth/TLS or clarify.
- **[PR #2463]** CodeRabbit: Clarify RemoteWriteSpec **name** when optional: what happens when omitted, whether system auto-generates, how to troubleshoot without names.
- **[PR #2463]** everettraven (documented in summary): Prefer headers as `[]struct { Name, Value string }` for validation of header names (e.g. reserved); duration strings replaced with explicit units (e.g. batchSendDeadlineSeconds); listMapKey changed from url to name.
- **[PR #2653]** marioferh (Feb 18): Linked to Prometheus remote write docs in response to a discussion comment.

---

## [5. Validation (general)](#5-validation-general)

**Status:** Mostly resolved (validations in types). Pending: ensure generated OpenAPI/CRDs reflect all markers after `make update`.

**Summary:** CEL, enum, MinLength/MaxLength, MinItems/MaxItems, pattern, minProperties; align schema with documented constraints. OpenAPI/CRD should reflect kubebuilder markers.

- **[PR #2653]** **coderabbitai[bot]**: Add schema validation for scheme/pathPrefix/staticConfigs (Enum, Pattern, MinItems/MaxItems); align schema with documented constraints for collectionProfile, logLevel, externalLabels, remoteWrite, tolerations, topologySpreadConstraints.
- **[PR #2463]** CodeRabbit: Many OpenAPI descriptions document enums/size limits that are not in the schema; add kubebuilder validations so OpenAPI has enum, minItems/maxItems, min/max.
- **[PR #2463]** CodeRabbit: AdditionalAlertmanagerConfig—staticConfigs min/max, name DNS subdomain, pathPrefix pattern, scheme enum, timeoutSeconds range; RelabelConfig action enum and field constraints; RemoteWriteSpec and Retention min/max.
- **[PR #2630]** CodeRabbit: Add negative tests for relabel targetLabel (action Replace/HashMod without targetLabel) and for malformed staticConfigs.
- **[PR #2463]** CodeRabbit: CEL for retention duration format; pattern for enforcedBodySizeLimit and retention; duration/quantity pattern validation.

---

## [6. Documentation](#6-documentation)

**Status:** Partial. PrometheusConfig.logLevel already says "Prometheus"; no external links. **Pending:** LogLevel type (generic), LogLevel constants (full sentences), pathPrefix v2 example.

**Summary:** GoDoc and CRD descriptions: avoid external links, use "no opinion, platform chooses reasonable default," fix copy-paste (Alertmanager vs Prometheus for logLevel), pathPrefix example (v1 vs v2), scheme "default" vs required.

- **[PR #2463]** CodeRabbit (repeated): **logLevel** description says "logs emitted by Alertmanager" but field is under prometheusK8sConfig; change to "Prometheus" in types_cluster_monitoring.go and regenerate.
- **[PR #2463]** CodeRabbit: **LogLevel** type is shared by Alertmanager and Prometheus; godoc should be generic, e.g. "monitoring components (such as Alertmanager and Prometheus)."
- **[PR #2463]** CodeRabbit: Remove **external documentation links** from PrometheusConfig; API should be self-contained.
- **[PR #2463]** CodeRabbit: Use standard defaulting phrase: "When omitted, this means the user has no opinion and the platform is left to choose reasonable defaults, which are subject to change over time."
- **[PR #2463]** CodeRabbit: **pathPrefix** example uses `/api/v1/alerts` but apiVersion only allows v2; use `/api/v2/alerts`. **scheme** comment says "default value is http" but field is +required with no +kubebuilder:default; either add default or update doc (e.g. "Possible values are `http` or `https`").
- **[PR #2463]** CodeRabbit: Remove "This is a custom type to allow for admission time validations" from user-facing bearerToken doc. Align volumeClaimTemplate description between Prometheus and Alertmanager (e.g. "data will not persist").
- **[PR #2463]** CodeRabbit: Documentation says "remote write/read" but only RemoteWrite exists; say "remote write" only or note remote read may be added later.

---

## [7. Defaults and API contract](#7-defaults-and-api-contract)

**Status:** Resolved (no fixed default for scheme in schema; "no opinion" wording on optional fields).

**Summary:** Avoid `+kubebuilder:default` in alpha for flexibility; controller-side defaulting preferred. "No opinion" wording for optional fields.

- **[PR #2630]** (from conversation summary) **everettraven:** `+kubebuilder:default=HTTP` for scheme in v1alpha1 locks in a default; prefer controller defaulting.
- **[PR #2630]** CodeRabbit: spec.prometheusConfig default `{}` in schema—if "unset vs set empty" should be distinguishable for SSA, omit default in source and regenerate.

---

## [8. pathPrefix / scheme / staticConfigs](#8-pathprefix-scheme-staticconfigs)

**Status:** Validations and scheme OK. **Pending:** in pathPrefix, change example from `/api/v1/alerts` to `/api/v2/alerts` if Alertmanager API only exposes v2.

**Summary:** Allowed values (scheme HTTP/HTTPS), pathPrefix format (trailing slash, no query/fragment), staticConfigs host:port validation and min/max items.

- **[PR #2463]** CodeRabbit: pathPrefix validation—disallow pathPrefix `/` producing `//api/v1/alerts` unless explicitly allowed; scheme enum; staticConfigs minItems/maxItems and host:port pattern.
- **[PR #2630]** CodeRabbit: staticConfigs validation using `isURL('http://' + self)` accepts "host:port/path" or "host:port?query"; description says only host:port—optional stricter validation or document.
- **[PR #2630]** CodeRabbit: pathPrefix example should use /api/v2/alerts; scheme doc vs required/default.

---

## [9. LogLevel](#9-loglevel)

**Status:** Pending. In `types_cluster_monitoring.go`: (1) `LogLevel` type (line ~300) still says "emitted by Alertmanager" → use generic wording; (2) LogLevelWarn, LogLevelInfo, LogLevelDebug constants (lines ~308-312) use leading comma → use full sentence (e.g. "LogLevelWarn logs warnings and errors.").

**Summary:** Comment wording (e.g. "LogLevelWarn means…" not ", both warnings…"); shared type doc should not say "Alertmanager" only.

- **[PR #2630]** **coderabbitai[bot]** (Jan 9): LogLevel constant comments (LogLevelWarn, LogLevelInfo, LogLevelDebug) — remove leading commas, make full sentences matching LogLevelError (e.g. "LogLevelWarn logs warnings and errors.").
- **[PR #2463]** Multiple: logLevel field and LogLevel type must reference "Prometheus" in prometheusK8sConfig context; shared type doc should be component-agnostic.

---

## [10. Process / PR management](#10-process-pr-management)

**Summary:** PR size, splitting, closing in favor of newer PRs, CI failures (verify-crd-schema, verify-crdify), rebase, conflict markers.

- **[PR #2463]** **JoelSpeed** (Sep 16): Conflict markers in the PR; please fix and push correct content.
- **[PR #2463]** **marioferh**: Fixed mixed commit; force-pushed.
- **[PR #2463]** **danielmellado** (Dec 2): Addressed comments, asking everettraven for a fresh review.
- **[PR #2463]** **everettraven** (Jan 8): Closing in favor of #2630. /close
- **[PR #2463]** **everettraven** (Jan 12): danielmellado spun this into #2630 because the review was so large the PR took a long time to load.
- **[PR #2653]** Description: "This pr replaces https://github.com/openshift/api/pull/2630 because is broken."
- **[PR #2630]** **danielmellado**: Plan to split into smaller changes; asking everettraven re volumeClaimTemplate and overrides.
- **[PR #2653]** **everettraven** (Jan 28): Diff since last review was bigger than expected; didn't finish full review; will return next morning.
- **[PR #2653]** needs-rebase, verify-deps, verify-crd-schema, lint, etc. (Mar 5); marioferh force-pushed and pushed multiple fix commits.

---

## [11. Bots / automation](#11-bots-automation)

**Summary:** CodeRabbit actionable vs nitpick vs "Additional comments"; openshift-ci; docstring coverage; kubeapilinter.

| Source | Summary |
|--------|--------|
| **CodeRabbit** | Multiple review passes per PR; "Actionable comments" (2–8+), "Nitpick" (1–5+), "Duplicate comments" (2–11), "Additional comments" (15–54). Many LGTM-style or "no action needed" items. Focus on schema vs doc alignment, validation in OpenAPI/CRD, and consistency across CRD variants. |
| **openshift-ci[bot]** | API conventions reminder, approval notifier, size/XXL label, assignees (everettraven, JoelSpeed). |
| **Docstring coverage** | 0.00% reported; threshold 80%. Resolution: write docstrings for missing functions. |
| **golangci-lint / kubeapilinter** | Error: "unable to load custom analyzer kubeapilinter: plugin: not implemented" — tool/config issue, not fixed in PRs. |

---

## [12. Other / uncategorized](#12-other-uncategorized)

- **[PR #2463]** **simonpasquier**: Regarding max values, some may be too low; consider leveraging Insights data for current practices.
- **[PR #2463]** CodeRabbit: additionalAlertmanagerConfigs listMapKey (e.g. apiVersion vs name); verify minProperties with empty prometheusK8sConfig; listType=atomic vs map for multi-owner (e.g. ACM + cluster admin).
- **[PR #2463]** CodeRabbit: retentionSize vs volumeClaimTemplate storage—operator check to warn/error if retention exceeds PVC size.
- **[PR #2653]** **marioferh**: @coderabbit ignore (reviews paused).

---

## [Comments remaining to address](#comments-remaining-to-address)

Summary of what is still pending in code or docs according to the current repo state:

1. **LogLevel (type and constants)** — `config/v1alpha1/types_cluster_monitoring.go`
   - **`LogLevel` type** (approx. line 300): The godoc says "verbosity of logs emitted by **Alertmanager**". It should be generic, e.g.: "verbosity of logs emitted by monitoring components (such as Alertmanager and Prometheus)."
   - **Constants** (approx. lines 308-312): LogLevelWarn, LogLevelInfo, and LogLevelDebug use a leading comma ("LogLevelWarn, both warnings…"). Change to full sentences, e.g.: "LogLevelWarn logs warnings and errors.", "LogLevelInfo logs general information, warnings, and errors.", "LogLevelDebug logs detailed debugging information."

2. **pathPrefix (example in docs)** — `config/v1alpha1/types_cluster_monitoring.go` (approx. lines 782, 784)
   - The example uses `/api/v1/alerts`. If the exposed Alertmanager API is v2 only, change to `/api/v2/alerts` in both places (pathPrefix and the "double slashes" note).

3. **Docstring coverage**
   - CodeRabbit reports 0% docstring coverage (80% threshold). Add docstrings to functions that still lack them (per the repo linter).

4. **Optional — RelabelConfig regex validation**
   - Document in the RelabelConfig `regex` field that RE2 syntax is validated at **runtime** by Prometheus, not at admission, so it is clear to users.

5. **Optional — Negative tests**
   - Add tests that fail when `action` is Replace or HashMod but `targetLabel` is missing, and when `staticConfigs` has invalid format (host:port), to ensure CEL rules are applied correctly.

6. **CI / verify**
   - After changes, run `make update` (without `API_GROUP_VERSIONS`) and `make verify` to ensure OpenAPI and CRDs are up to date and there are no regressions.

---

## [Per-PR index (theme anchors)](#per-pr-index-theme-anchors)

- **#2653 (open):** 2 (RelabelConfig), 3 (TLS/Authorization/SecretKeySelector), 4 (RemoteWrite), 5 (Validation), 10 (Process), 11 (Bots), 12 (Other).
- **#2463 (closed):** 1 (volumeClaimTemplate refs), 2 (RelabelConfig), 4 (RemoteWrite), 5 (Validation), 6 (Documentation), 8 (pathPrefix/scheme/staticConfigs), 9 (LogLevel), 10 (Process), 11 (Bots), 12 (Other).
- **#2630 (closed):** 1 (volumeClaimTemplate/PVC), 2 (RelabelConfig), 3 (TLS/Authorization), 5 (Validation), 6 (Documentation), 7 (Defaults), 8 (pathPrefix/scheme), 9 (LogLevel), 10 (Process), 11 (Bots).
- **#2652 (closed):** Short-lived; mostly bot timeline (CodeRabbit review failed—PR closed), openshift-ci, no substantive human comments extracted.

---

## [Data and limitations](#data-and-limitations)

- **#2653** and **#2463**: Content from existing agent-tools text files (full PR page timeline). Paths: see "Saved PR content (agent-tools)" above.
- **#2630**: Fetched via mcp_web_fetch (re-tried and re-saved); saved to agent-tools as `57fa43aa-02c1-4fce-a426-3dc504fb404d.txt` (4034 lines, 171.5 KB). Same content as earlier fetch; used for theme extraction.
- **#2652**: Fetched via mcp_web_fetch (re-tried and saved); saved to agent-tools as `pr-2652-openshift-api.txt` (~170 lines). PR was closed very quickly; only bot/automation comments and no detailed human review in the captured content.
- **Inline review comments** attached to specific lines in the "Files changed" view may not appear in the main timeline used here; some everettraven "reviewed: commented" events are present but the full inline body is not always in the same format. Where specific file/line references appeared in CodeRabbit or quoted replies, they were used in the theme sections above.
- **Resolutions** mentioned (e.g. volumeClaimTemplate reverted to *v1.PersistentVolumeClaim, listMapKey to name, headers to slice of structs) come from the conversation summary; the PR diffs themselves were not re-checked for current state.

---

## [Possible unresolved comments](#possible-unresolved-comments)

Feedback from the PRs that may not yet be fully resolved or that was left open / optional. Worth checking against the current branch and open PR #2653.

- **OpenAPI / generated schema vs Go types** — Several CodeRabbit comments asked for enum, min/max, and pattern constraints to appear in the generated OpenAPI and CRDs. Resolution depends on `make update` and generator behavior; worth confirming that all kubebuilder markers are reflected in the payload CRDs and `openapi/openapi.json`.

- **SecretKeySelector `optional` field** — CodeRabbit (PR #2463) asked to document a Required/Optional field and use enum + fail-closed handling. The current `SecretKeySelector` in this repo has only `name` and `key`; if another API or a different version has `optional`, that comment may apply there or may have been dropped by design.

- **additionalAlertmanagerConfigs listMapKey** — Discussion about using `name` (or a composite key) vs `apiVersion` for list uniqueness and server-side apply. Unclear if the final choice was reflected in the CRD and whether multi-owner (e.g. ACM + cluster admin) is supported.

- **listType=atomic vs map for additionalAlertmanagerConfigs** — Comment that atomic does not work well with SSA when multiple controllers manage entries; may still be a limitation or may have been accepted as-is.

- **retentionSize vs volumeClaimTemplate** — Suggestion to add a validation or operator check when retention size exceeds PVC capacity. May be operator-side only and not in this API repo.

- **Max values (e.g. list sizes, retention)** — simonpasquier noted some max values may be too low and suggested using Insights data; no concrete change was specified. May still be open for product decision.

- **stricter staticConfigs validation** — CodeRabbit noted that `isURL('http://' + self)` accepts host:port/path or host:port?query while the description says only host:port. Optional stricter CEL or doc clarification may still be pending.

- **Prometheus label name pattern on `Label.key` and `sourceLabels`** — Some comments asked for `^[a-zA-Z_][a-zA-Z0-9_]*$` (or similar) validation. The current types may allow UTF-8 or a different rule; worth confirming whether this was intentionally not added or is still desired.

- **Docstring coverage (0% vs 80% threshold)** — CodeRabbit and pre-merge checks reported this; resolution is to add docstrings. May still be outstanding on the open PR.

- **everettraven inline review threads** — Some "reviewed: commented" events may contain unresolved inline comments on specific lines. The timeline used here does not always include full inline bodies; the open PR should be scanned for any remaining requested changes.
