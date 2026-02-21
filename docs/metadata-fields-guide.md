# Metadata Fields Guide

This document identifies every field within `_meta["io.modelcontextprotocol.registry/publisher-provided"]["aregistry.ai/metadata"]`, traces where each field originates, and documents the code paths used to populate and modify them.

---

## Architecture Overview

Metadata flows through this pipeline:

```
Server seed data (JSON / Registry API)
        │
        ▼
importServer()                          # importer.go:184
        │
        ├── validators.ValidateServerJSON()
        │
        ▼
enrichServer()                          # importer.go:398
        │
        ├── fetchGitHubRepoSummary()    → repo, activity, stars
        ├── fetchGitHubReleasesSummary()→ downloads, releases
        ├── fetchGitHubTopics()         → repo.topics (fallback)
        ├── fetchGitHubTags()           → repo.tags
        ├── fetchGitHubOrgIsVerified()  → identity.org_is_verified
        ├── detectDependabotEnabled()   → security_scanning.dependabot_enabled
        ├── detectCodeQLEnabled()       → security_scanning.codeql_enabled
        ├── fetchDependabotAlertsCount()→ security_scanning.dependabot_alerts
        ├── fetchCodeScanningAlertsCount() → security_scanning.code_scanning_alerts
        ├── fetchOpenSSFScore()         → scorecard.openssf
        ├── runScorecardLibrary()       → scorecard.openssf + scans.details (highlights)
        ├── runScorecardLocal()         → scorecard.openssf (fallback)
        ├── fetchDependencyHealthSummary() → scans.dependency_health
        ├── fetchDockerHubSummary()     → scans.container_images
        ├── runOSVScan()               → scans.summary + scans.details
        ├── probeEndpointHealth()       → endpoint_health
        ├── isSemverVersion()           → semver.uses_semver
        └── score computation           → score (computed inline)
        │
        ▼
server.Meta.PublisherProvided["aregistry.ai/metadata"] = enterprise
        │
        ▼
registry.CreateServer() / UpdateServer()  → persisted to database
```

The entire enrichment happens in `enrichServer()` at `internal/registry/importer/importer.go:398-608`. All sub-fields are assembled into the `enterprise` map (line 495) and assigned to `server.Meta.PublisherProvided["aregistry.ai/metadata"]` at line 607.

---

## Field-by-Field Reference

### `stars` (int)

| Attribute | Value |
|-----------|-------|
| **Source** | GitHub REST API `/repos/{owner}/{repo}` → `stargazers_count` |
| **Fetched by** | `fetchGitHubRepoSummary()` — `importer.go:668` |
| **Stored at** | `enterprise["stars"]` — `importer.go:496` |
| **Modify** | Change the `Stars` field in `githubRepoSummary` struct (line 789) or override the value in the `enterprise` map before line 607 |

---

### `score` (float64)

| Attribute | Value |
|-----------|-------|
| **Source** | Computed inline: `0.6*log10(stars+1) + 0.4*log10(downloads.total+1)` |
| **Computed at** | `importer.go:420` |
| **Stored at** | `enterprise["score"]` — `importer.go:500` |
| **Modify** | Change the formula at line 420. The two coefficients (0.6, 0.4) control the relative weight of stars vs downloads. You can add more signals by extending the formula. |

---

### `downloads.total` (int)

| Attribute | Value |
|-----------|-------|
| **Source** | GitHub REST API `/repos/{owner}/{repo}/releases` → sum of all `assets[].download_count` across all release pages |
| **Fetched by** | `fetchGitHubReleasesSummary()` — `importer.go:724` |
| **Struct** | `githubReleasesSummary.TotalDownloads` — `importer.go:800` |
| **Stored at** | `enterprise["downloads"]["total"]` — `importer.go:497-499` |
| **Modify** | The value comes from paginated release asset counts. To add npm/PyPI download counts, you'd extend the function or add a parallel fetch and merge the totals. |

---

### `repo.forks_count` (int)

| Attribute | Value |
|-----------|-------|
| **Source** | GitHub REST API `/repos/{owner}/{repo}` → `forks_count` |
| **Fetched by** | `fetchGitHubRepoSummary()` — `importer.go:668` |
| **Stored at** | `enterprise["repo"]["forks_count"]` — `importer.go:502` |
| **Modify** | Mapped directly from GitHub API response field `forks_count` |

---

### `repo.watchers_count` (int)

| Attribute | Value |
|-----------|-------|
| **Source** | GitHub REST API `/repos/{owner}/{repo}` → `watchers_count` |
| **Fetched by** | `fetchGitHubRepoSummary()` — `importer.go:668` |
| **Stored at** | `enterprise["repo"]["watchers_count"]` — `importer.go:503` |

---

### `repo.primary_language` (string | null)

| Attribute | Value |
|-----------|-------|
| **Source** | GitHub REST API `/repos/{owner}/{repo}` → `language` |
| **Fetched by** | `fetchGitHubRepoSummary()` — `importer.go:668` |
| **Stored at** | `enterprise["repo"]["primary_language"]` — `importer.go:504` |

---

### `repo.topics` ([]string)

| Attribute | Value |
|-----------|-------|
| **Source** | GitHub REST API `/repos/{owner}/{repo}` → `topics` field, with fallback to `/repos/{owner}/{repo}/topics` if empty |
| **Fetched by** | `fetchGitHubRepoSummary()` (primary), `fetchGitHubTopics()` (fallback) — `importer.go:426-429` |
| **Stored at** | `enterprise["repo"]["topics"]` — `importer.go:505` |

---

### `repo.tags` ([]string)

| Attribute | Value |
|-----------|-------|
| **Source** | GitHub REST API `/repos/{owner}/{repo}/tags` → tag `name` fields (up to 100) |
| **Fetched by** | `fetchGitHubTags()` — `importer.go:858` |
| **Stored at** | `enterprise["repo"]["tags"]` — `importer.go:506` |

---

### `activity.created_at` (string, RFC3339)

| Attribute | Value |
|-----------|-------|
| **Source** | GitHub REST API `/repos/{owner}/{repo}` → `created_at` |
| **Fetched by** | `fetchGitHubRepoSummary()` — `importer.go:668` |
| **Stored at** | `enterprise["activity"]["created_at"]` — `importer.go:509` |
| **Format** | Converted via `timePtrToRFC3339()` — `importer.go:813` |

---

### `activity.updated_at` (string, RFC3339)

| Attribute | Value |
|-----------|-------|
| **Source** | GitHub REST API `/repos/{owner}/{repo}` → `updated_at` |
| **Fetched by** | `fetchGitHubRepoSummary()` — `importer.go:668` |
| **Stored at** | `enterprise["activity"]["updated_at"]` — `importer.go:510` |

---

### `activity.pushed_at` (string, RFC3339)

| Attribute | Value |
|-----------|-------|
| **Source** | GitHub REST API `/repos/{owner}/{repo}` → `pushed_at` |
| **Fetched by** | `fetchGitHubRepoSummary()` — `importer.go:668` |
| **Stored at** | `enterprise["activity"]["pushed_at"]` — `importer.go:511` |

---

### `releases.latest_published_at` (string | null, RFC3339)

| Attribute | Value |
|-----------|-------|
| **Source** | GitHub REST API `/repos/{owner}/{repo}/releases` → most recent `published_at` |
| **Fetched by** | `fetchGitHubReleasesSummary()` — `importer.go:724` |
| **Stored at** | `enterprise["releases"]["latest_published_at"]` — `importer.go:514` |

---

### `identity.org_is_verified` (bool)

| Attribute | Value |
|-----------|-------|
| **Source** | GitHub REST API `/orgs/{owner}` → `is_verified` |
| **Fetched by** | `fetchGitHubOrgIsVerified()` — `importer.go:908` |
| **Stored at** | `enterprise["identity"]["org_is_verified"]` — `importer.go:518` |
| **Note** | Returns `false` if owner is a user (not an org), or if the API returns 404 |

---

### `identity.publisher_identity_verified_by_jwt` (bool)

| Attribute | Value |
|-----------|-------|
| **Source** | Hardcoded to `false` in the importer (importer lacks JWT context) |
| **Stored at** | `enterprise["identity"]["publisher_identity_verified_by_jwt"]` — `importer.go:517` |
| **Note** | Set to `true` elsewhere in the publish flow when JWT verification succeeds |

---

### `semver.uses_semver` (bool)

| Attribute | Value |
|-----------|-------|
| **Source** | Computed from `server.Version` using regex validation |
| **Computed by** | `isSemverVersion()` — `importer.go:806` |
| **Stored at** | `enterprise["semver"]["uses_semver"]` — `importer.go:521` |
| **Regex** | `^v?(0|[1-9]\d*)\.(0|[1-9]\d*)\.(0|[1-9]\d*)(-pre)?(\+build)?$` |

---

### `scorecard.openssf` (float64)

| Attribute | Value |
|-----------|-------|
| **Source** | OpenSSF Scorecard, with 3 fallback strategies: (1) `runScorecardLibrary()` using the Go library, (2) `runScorecardLocal()` using a local CLI binary, (3) `fetchOpenSSFScore()` via public API |
| **Fetched by** | `importer.go:455-464` |
| **Files** | `scorecard_lib.go:22` (library), `importer.go:1513` (local CLI) |
| **Stored at** | `enterprise["scorecard"]["openssf"]` — `importer.go:524` |
| **Highlights** | When using the library method, individual check results are returned as `scorecardHighlights` and added to `scans.details` |

---

### `endpoint_health.reachable` (bool | null)

| Attribute | Value |
|-----------|-------|
| **Source** | HTTP HEAD/GET probe against `server.Remotes[0].URL` |
| **Fetched by** | `probeEndpointHealth()` — `importer.go:1060` |
| **Stored at** | `enterprise["endpoint_health"]["reachable"]` — `importer.go:527` |
| **Note** | Only probes the first remote endpoint. `null` if no remotes configured. |

---

### `endpoint_health.response_ms` (int | null)

| Attribute | Value |
|-----------|-------|
| **Source** | Round-trip time measurement from `probeEndpointHealth()` |
| **Stored at** | `enterprise["endpoint_health"]["response_ms"]` — `importer.go:528` |

---

### `endpoint_health.last_checked_at` (string | null, RFC3339)

| Attribute | Value |
|-----------|-------|
| **Source** | Timestamp captured at probe time by `probeEndpointHealth()` |
| **Stored at** | `enterprise["endpoint_health"]["last_checked_at"]` — `importer.go:529` |

---

### `security_scanning.codeql_enabled` (bool)

| Attribute | Value |
|-----------|-------|
| **Source** | Heuristic scan of `.github/workflows/*.yml` files for `codeql` references |
| **Fetched by** | `detectCodeQLEnabled()` — `importer.go:978` |
| **Stored at** | `enterprise["security_scanning"]["codeql_enabled"]` — `importer.go:532` |

---

### `security_scanning.dependabot_enabled` (bool)

| Attribute | Value |
|-----------|-------|
| **Source** | Checks for existence of `.github/dependabot.yml` via GitHub Contents API |
| **Fetched by** | `detectDependabotEnabled()` — `importer.go:947` |
| **Stored at** | `enterprise["security_scanning"]["dependabot_enabled"]` — `importer.go:533` |

---

### `security_scanning.code_scanning_alerts` (int | null)

| Attribute | Value |
|-----------|-------|
| **Source** | GitHub REST API code scanning alerts endpoint (requires `githubToken`) |
| **Fetched by** | `fetchCodeScanningAlertsCount()` — `importer.go:1119` |
| **Stored at** | `enterprise["security_scanning"]["code_scanning_alerts"]` — `importer.go:534` |
| **Note** | `null` when no GitHub token is configured |

---

### `security_scanning.dependabot_alerts` (int | null)

| Attribute | Value |
|-----------|-------|
| **Source** | GitHub REST API Dependabot alerts endpoint (requires `githubToken`) |
| **Fetched by** | `fetchDependabotAlertsCount()` — `importer.go:1110` |
| **Stored at** | `enterprise["security_scanning"]["dependabot_alerts"]` — `importer.go:535` |
| **Note** | `null` when no GitHub token is configured |

---

### `scans.summary` (string | null)

| Attribute | Value |
|-----------|-------|
| **Source** | Concatenation of up to 3 sub-summaries separated by ` \| `: OSV scan summary, dependency health summary, container image summary |
| **Components** | `osvScanResult.Summary`, `dependencyHealthSummary.summaryString()`, `containerImageSummary.summaryString()` |
| **Assembled at** | `importer.go:537-553` |
| **Example** | `"osv: npm=0, pip=0, go=0 | deps: total=274 top=npm:274 unknown=19"` |

---

### `scans.details` ([]string)

| Attribute | Value |
|-----------|-------|
| **Source** | Array of up to 50 detail strings from: OpenSSF scorecard highlights, dependency health detail, container image detail, OSV vulnerability details |
| **Components** | `scorecardHighlights[]`, `dependencyHealthSummary.detailString()`, `containerImageSummary.detailString()`, `osvScanResult.Details[]` |
| **Assembled at** | `importer.go:554-573` |

---

### `scans.dependency_health.packages_total` (int)

| Attribute | Value |
|-----------|-------|
| **Source** | GitHub REST API SBOM endpoint `/repos/{owner}/{repo}/dependency-graph/sbom` → count of packages |
| **Fetched by** | `fetchDependencyHealthSummary()` — `dependency_health.go:81` |
| **Stored at** | `enterprise["scans"]["dependency_health"]["packages_total"]` — `importer.go:582` |

---

### `scans.dependency_health.ecosystems` (map[string]int)

| Attribute | Value |
|-----------|-------|
| **Source** | SBOM packages grouped by purl type (e.g., `npm`, `pypi`, `golang`) |
| **Fetched by** | `fetchDependencyHealthSummary()` + `detectPurlType()` — `dependency_health.go:81, 159` |
| **Stored at** | `enterprise["scans"]["dependency_health"]["ecosystems"]` — `importer.go:583` |

---

### `scans.dependency_health.copyleft_licenses` (int)

| Attribute | Value |
|-----------|-------|
| **Source** | Count of SBOM packages with copyleft licenses (GPL, AGPL, LGPL, SSPL, CC-BY-SA) |
| **Fetched by** | `hasCopyleftLicense()` — `dependency_health.go:183` |
| **Stored at** | `enterprise["scans"]["dependency_health"]["copyleft_licenses"]` — `importer.go:584` |

---

### `scans.dependency_health.unknown_licenses` (int)

| Attribute | Value |
|-----------|-------|
| **Source** | Count of SBOM packages where license is empty or `NOASSERTION` |
| **Fetched by** | `isLicenseUnknown()` — `dependency_health.go:194` |
| **Stored at** | `enterprise["scans"]["dependency_health"]["unknown_licenses"]` — `importer.go:585` |

---

### `scans.container_images` ([]object)

| Attribute | Value |
|-----------|-------|
| **Source** | Docker Hub API for matching container image |
| **Fetched by** | `fetchDockerHubSummary()` — `container_scan.go:66` |
| **Stored at** | `enterprise["scans"]["container_images"]` — `importer.go:588-602` |
| **Sub-fields** | `registry`, `image`, `pull_count`, `star_count`, `last_updated_at`, `latest_tag`, `latest_tag_updated_at` |
| **Note** | Returns empty array `[]` if no Docker Hub image found |

---

## How to Modify Metadata

### 1. Add a new field

To add a new metadata field to the enrichment output:

1. **Fetch the data** — Add a new `fetch*()` or `detect*()` method to `importer.go` (or a new file in `internal/registry/importer/`)
2. **Call it in `enrichServer()`** — Add the call between lines 398-487 (alongside the existing enrichment calls)
3. **Add to `enterprise` map** — Insert your field into the `enterprise` map literal (lines 495-605)
4. **Update TypeScript types** — Add the field to the `ServerJSON._meta` interface in `ui/lib/admin-api.ts`

Example — adding a `license` field:

```go
// In enrichServer(), after existing fetches:
repoLicense, _ := s.fetchGitHubRepoLicense(ctx, owner, repo)

// In the enterprise map:
"license": map[string]any{
    "spdx_id": repoLicense,
},
```

### 2. Remove a field

To remove a field from the enrichment output:

1. **Remove from `enterprise` map** — Delete the key from lines 495-605 in `enrichServer()`
2. **Remove the fetch call** — Delete the corresponding `fetch*()` / `detect*()` call if no longer needed
3. **Update TypeScript types** — Remove the field from `ui/lib/admin-api.ts`
4. **Clean up unused functions** — Delete the helper function if nothing else uses it

### 3. Conditionally include/exclude fields based on server details

The `enrichServer()` function receives the full `*apiv0.ServerJSON` object. You can inspect any server field to decide what metadata to include:

```go
// Only include endpoint_health if the server has remotes
if len(server.Remotes) > 0 && server.Remotes[0].URL != "" {
    // ... (already done, see importer.go:477-486)
}

// Only include container_images if server has OCI packages
hasOCI := false
for _, pkg := range server.Packages {
    if pkg.RegistryType == "oci" {
        hasOCI = true
        break
    }
}
if hasOCI {
    enterprise["scans"]["container_images"] = ...
}

// Only include security scanning if it's a GitHub repo
if server.Repository != nil && strings.Contains(server.Repository.URL, "github.com") {
    enterprise["security_scanning"] = ...
}
```

### 4. Modify the score formula

The score formula at `importer.go:420`:

```go
score := 0.6*math.Log10(float64(repoSummary.Stars)+1) + 0.4*math.Log10(float64(releasesSummary.TotalDownloads)+1)
```

To incorporate additional signals (e.g., endpoint health, OpenSSF score):

```go
score := 0.4*math.Log10(float64(repoSummary.Stars)+1) +
         0.3*math.Log10(float64(releasesSummary.TotalDownloads)+1) +
         0.2*(ossfScore/10.0) +
         0.1*boolToFloat(endpointReachable)
```

### 5. Modify enrichment at the API layer (post-database)

The database layer also injects a semantic score at query time:

- **File:** `internal/registry/database/postgres.go:279-287`
- **Key:** `aregistry.ai/semantic` (separate from `aregistry.ai/metadata`)
- This is the cosine-similarity score from embedding search, not the enrichment score

### 6. Modify via the UpdateServer API

Metadata can also be updated after initial import through the `UpdateServer` API:

- **Handler:** `internal/registry/api/handlers/v0/edit.go`
- **Service:** `internal/registry/service/` → `UpdateServer()`
- The `_meta.publisher-provided` data is part of the `ServerJSON` payload and gets persisted as-is

---

## Key Files Reference

| File | Purpose |
|------|---------|
| `internal/registry/importer/importer.go` | Main enrichment pipeline (`enrichServer` at line 398) |
| `internal/registry/importer/dependency_health.go` | SBOM-based dependency analysis |
| `internal/registry/importer/container_scan.go` | Docker Hub container image metadata |
| `internal/registry/importer/osv_scan.go` | OSV vulnerability scanning (npm, pip, go) |
| `internal/registry/importer/scorecard_lib.go` | OpenSSF Scorecard via Go library |
| `internal/registry/database/postgres.go` | Database storage and semantic score injection |
| `internal/registry/api/handlers/v0/servers.go` | API response assembly |
| `internal/registry/api/handlers/v0/edit.go` | Server update (can modify metadata) |
| `pkg/models/server_response.go` | Go response types |
| `ui/lib/admin-api.ts` | TypeScript type definitions |

---

## External API Dependencies

| API | Fields it feeds | Auth required |
|-----|----------------|---------------|
| GitHub REST API `/repos/{owner}/{repo}` | stars, repo.*, activity.* | Optional (token for higher rate limits) |
| GitHub REST API `/repos/{owner}/{repo}/releases` | downloads.total, releases.latest_published_at | Optional |
| GitHub REST API `/repos/{owner}/{repo}/topics` | repo.topics (fallback) | Optional |
| GitHub REST API `/repos/{owner}/{repo}/tags` | repo.tags | Optional |
| GitHub REST API `/orgs/{owner}` | identity.org_is_verified | Optional |
| GitHub REST API `/repos/.../contents/.github/dependabot.yml` | security_scanning.dependabot_enabled | Optional |
| GitHub REST API `/repos/.../contents/.github/workflows` | security_scanning.codeql_enabled | Optional |
| GitHub REST API `/repos/.../dependabot/alerts` | security_scanning.dependabot_alerts | **Required** |
| GitHub REST API `/repos/.../code-scanning/alerts` | security_scanning.code_scanning_alerts | **Required** |
| GitHub REST API `/repos/.../dependency-graph/sbom` | scans.dependency_health.* | Optional |
| OpenSSF Scorecard API | scorecard.openssf | None |
| Docker Hub API `/v2/repositories/{owner}/{repo}` | scans.container_images | None |
| OSV API `/v1/querybatch` | scans.summary, scans.details | None |
| Direct HTTP probe | endpoint_health.* | None |
