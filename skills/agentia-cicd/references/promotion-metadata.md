<!-- Agentia managed skill file: agentia setup skills -->

# Promotion Metadata Dataset Reference

Use this reference for the CICD Gateway **promoted-metadata** dataset through Agentia.

| Surface | Name |
| --- | --- |
| CLI query | `agentia cicd promotion metadata query` |
| CLI schema | `agentia cicd promotion metadata schema` |
| MCP query | `agentia_promotion_metadata` |
| MCP schema | `agentia_promotion_metadata_schema` |

## Purpose

Answer: *Which metadata components rode which user stories into which promotions, and which commit(s) delivered them?*

Prefer this tool over raw SOQL, Apex, or ad-hoc Copado object queries whenever the user asks about:

- what was promoted / will be promoted
- which stories and commits introduced a component
- stale / stalled stories riding a newer promotion
- override / last-writer-wins risk across promotions
- provenance of a component in an environment path
- release / pipeline / project scoped promotion content

Do **not** invent Copado object relationships. Reason only from dataset rows + schema.

This is **not** Org Intelligence (`metadata explore`), the Copado metadata index (`metadata list`), or release data-template dataset files (`data dataset-file`).

## Agent Workflow (mandatory)

1. Clarify the target: promotion, release, destination env/org, story, and/or component.
2. Call `promotion metadata schema` / `agentia_promotion_metadata_schema` if filters or fields are unclear.
3. Run the **narrowest** query that can answer; widen only if empty or incomplete.
4. For override / stalled-story questions, **always** use `commitMode=all` for the candidate promotion/stories, then a second pass for colliding components on the destination path.
5. Paginate with `cursor` / `--cursor` until `page.nextCursor` is null before concluding “no conflicts.” Keep the same filters/sort/fields/commitMode across pages.
6. Answer with evidence: story ids/names, promotion ids/names/dates, component type/name, commit SHAs/dates, and the risk reasoning.
7. If DSM-005 (too broad), add filters and retry — do not invent data.
8. If the Gateway is unreachable, say so; do not fabricate rows.
9. Treat empty `rows` as a valid answer (“nothing matched”), not a tool failure.

## commitMode (critical)

| Mode | Rows | Use |
| --- | --- | --- |
| `latest` (default) | One row per component per promoted story — newest Complete commit with SHA whose `commitDate <= promotionDate` (fallback: newest Complete commit) | “What landed in this promotion.” |
| `all` | One row per Complete commit SHA for that story’s components | “Which older commits / stalled work may be piggybacking.” |

Never treat `latest` as full history when assessing stalled-story override risk.

## Important field semantics

- `promotionDate` = Promotion **CreatedDate** (UTC), not deploy completion.
- `sourceOrganizationId` / `destinationOrganizationId` = Salesforce org IDs from Copado environment `Org_ID__c`.
- `commitId` = git SHA from snapshot commit (Complete commits only).
- `commitRefId` = stable external id from the user-story commit record.
- `commitMessage` may be truncated; check `commitMessageTruncated`.
- `releaseId` / `releaseName` / `releaseVersion` are null when the promotion has no release.
- `isBackPromotion` marks back-promoted stories.
- `metadataName` is the component name **without** type prefix. Type may be `"Unknown"` when it cannot be derived.
- Row identity: `(promotionId, userStoryId, metadataType, metadataName[, commitId])`.

## Useful filters

Comma-separated multi-value unless noted. Environment filters accept **name, Copado env Id, or Salesforce org Id**.

`promotionId`, `promotionName`, `userStoryId`, `userStoryName`,  
`metadataType`, `metadataName`, `metadataNameContains` (min 3 chars),  
`commitId` (SHA or ≥7-char prefix),  
`promotionDateFrom`, `promotionDateTo`, `commitDateFrom`, `commitDateTo`,  
`promotionStatus`, `sourceEnvironment`, `destinationEnvironment`,  
`pipeline`, `project`, `release`,  
`includeBackPromotions` (default true on the gateway),  
`commitMode`, `fields`, `sort` (`-field` = desc), `limit` (1–2000, default 200), `cursor`

CLI uses kebab-case flags (`--promotion-id`, `--commit-mode`, `--no-include-back-promotions`). MCP uses camelCase.

Project only needed columns with `fields=...`. Default sort is already `-promotionDate,promotionName,userStoryName,metadataType,metadataName`.

CSV/ndjson export is **CLI-only** (`--format csv|ndjson`, optional `--output`). MCP returns JSON only.

## Primary use cases

### UC-1 — Stalled / old user stories piggybacking (override risk)

1. Identify the promotion under review (`promotionId` / `promotionName` / recent destination + date window).
2. Query with `commitMode=all` and fields including at least:  
   `promotionId,promotionName,promotionDate,userStoryId,userStoryName,userStoryTitle,metadataType,metadataName,commitId,commitRefId,commitDate,commitMessage,sourceEnvironmentName,destinationEnvironmentName,releaseName,releaseVersion`
3. Per `userStoryId`, compute `commitCount`, `oldestCommitDate`, `newestCommitDate`, `commitSpanDays`, `stallGapDays`.
4. Flag stalled / high piggyback risk when many Complete commits (e.g. 3+), long commit span (e.g. ≥14 days), or old commits far before `promotionDate`.
5. List distinct `(metadataType, metadataName)` for each flagged story.
6. **Override risk check:** for those components, query other promotions to the **same destination** with overlapping/recent `promotionDate`. Same component in a different story with a **newer `commitDate`**, while the stalled story would deploy later/together → report potential override / last-writer conflict.
7. Present risk level, stalled stories + stats, conflicted components, competing stories/promotions/commits, and recommended action. Show evidence rows, not just labels. Heuristic thresholds are starting points — adjust from user context.

### UC-2 — Promotion content inventory

`commitMode=latest`, filter by `promotionId` / `release` / `destinationEnvironment`; summarize by `metadataType` and story.

### UC-3 — Component provenance

Filter `metadataType` + `metadataName` (or `metadataNameContains`), optionally destination + date range; use `commitMode=all` for full commit trail.

### UC-4 — Cross-promotion collision scan

Pull promotion A’s component set (`latest`), then query recent promotions to that destination and intersect on `(metadataType, metadataName)`.

### UC-5 — Story impact in a promotion

`userStoryId` or `userStoryName` + optional `promotionId`; compare `latest` vs `all` when history vs selected commit matters.

### UC-6 — Environment / org path audit

Filter `sourceEnvironment` / `destinationEnvironment`; report org IDs when the user needs Salesforce-side correlation.

### UC-7 — Release notes / change set drafting

For a `release` or date window, aggregate stories + components + `commitMessage` with `commitMode=latest` unless they ask for full history.

### UC-8 — Back-promotion awareness

Set `includeBackPromotions=false` / `--no-include-back-promotions` unless the question is explicitly about back promotions; call out `isBackPromotion=true` rows separately when included.

## Examples

```sh
# Inventory (selected commits)
agentia cicd promotion metadata query --promotion-id a0X000000000001 --commit-mode latest --json

# Stalled-story / override analysis (full commits)
agentia cicd promotion metadata query --promotion-id a0X000000000001 --commit-mode all \
  --fields promotionId,promotionName,promotionDate,userStoryId,userStoryName,metadataType,metadataName,commitId,commitRefId,commitDate,commitMessage,destinationEnvironmentName,destinationOrganizationId \
  --json

# Component collision on a destination
agentia cicd promotion metadata query \
  --destination-environment UAT --metadata-type ApexClass --metadata-name MyClass \
  --promotion-date-from 2026-01-01 --commit-mode latest --json

# Discover filters/fields
agentia cicd promotion metadata schema --json

# Export
agentia cicd promotion metadata query --promotion-id a0X000000000001 --commit-mode all --format csv --output ./promo.csv
```

MCP equivalents use the same camelCase args (`promotionId`, `commitMode`, `fields`, …).

## Anti-patterns

- Do not query unfiltered org-wide history without a date or promotion/project bound.
- Do not treat `commitMode=latest` as full history for override risk.
- Do not confuse `promotionDate` with deploy-finished time.
- Do not assume every story commit is included — only **Complete** commits with a SHA appear.
- Do not use this dataset for write operations (promote, deploy, commit) — it is **read-only**.
