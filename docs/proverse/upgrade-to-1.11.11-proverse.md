# Upgrading labs ArtKeep to 1.11.11-proverse (TASK-006)

Source of the running deployment: image digest `sha256:9620bf42…`, built
2026-08-20 from `967411f2` (`hotfix/bug-003-npm-sha512-on-d6111765`, tag
`released/2026-08-20-967411f2`). That build sits on upstream `95b161c4`
(2026-07-22, `v1.5.3-169`), between the 1.6.0 and 1.7.0 releases.
`v1.8.2-proverse` was built but never deployed. The upgrade therefore spans
every upstream release from 1.7.0 to 1.11.11.

Target: branch `release/1_11_x_proverse`, which is upstream `v1.11.11`
(`faec02d0`) plus the commits listed below.

## What the branch carries

| Fork commit | Status on 1.11.11 | On the branch |
|---|---|---|
| S3 multipart sizing env knobs (BUG-002) | still hardcoded upstream | carried |
| `S3_REQUEST_TIMEOUT_SECS` alias for `S3_BULK_TIMEOUT_SECS` | no alias upstream | carried |
| `S3_RETRY_TIMEOUT_SECS` (lifts object_store's 180 s retry cap) | still never set upstream | carried |
| npm legacy `dist.shasum` for Unity UPM (BUG-003) | upstream parses it on publish but never serves it | carried |
| ghcr-only publish, lowercase names, ProVerseAI signing identity | upstream mirrors to Docker Hub | re-applied by hand |
| grype grpc-go CVE-2026-84304 override | fixed upstream (grype 0.120.1, override retired #3698) | dropped |
| `/v2` read confinement (TASK-006) | fixed upstream (#3261, #3316, #3408, #3704, #3716, #3717) | acceptance tests added |
| BUG-020 raw API token loses its allow-list on `/v2` | fixed upstream (`generate_tokens_with_repo_scope`) | acceptance tests added |
| BUG-021 cargo renamed deps served in publish form | still broken upstream | fixed on serve |
| Maven stored `maven-metadata.xml` keeps a stale `<latest>` | still broken upstream | fixed on serve |
| Remote docker tags never revalidate | fixed upstream (#3712 `revalidate_expired_remote_tag`, #3725 virtual members) | upstream tests cover it |

## Deploy procedure

1. Back up the database.
2. **Scale the backend to 0 replicas.** Do not do a rolling or mixed-version
   deploy. Migration 233 (`lifecycle_policy_assignments`) changes lifecycle
   policy scope. A pre-1.11 binary reads an unassigned or multi-repository
   policy as global and could delete across every repository.
3. Roll out the new image with 1 replica. Migrations run at boot, before the
   listener binds, so probes see connection-refused until they finish.
4. Check the backend log for `Proxy cache scope:` and for migration
   completion. Smoke-test login, a docker pull, an npm install through the UPM
   registry, a cargo build against the sparse index and a maven resolve.
5. Scale back to the normal replica count.

**Downtime estimate.** The live database is 185 MB:

| Table | Rows |
|---|---|
| `artifacts` | 2,373 |
| `oci_blobs` | 7,851 |
| `oci_tags` | 1,859 |
| `audit_log` | 25,729 |
| `download_statistics` | 39,488 |
| `proxy_download_statistics` | 0 |

The heaviest migrations are full-table updates or index builds on
`artifacts` and `oci_blobs`: 176, 193, 206, 227–229, 247, 261, 270, 271 and
284. At this size each finishes in well under a second, and all 103 together
should take under a minute. The startup backfills (`oci_manifests` from
`oci_tags`, `storage_metrics` history) are also seconds at this size.
**Budget 5 minutes of API downtime**, mostly image pull and pod start. Plan a
15-minute window.

Rollback: the database cannot be handed back to the old image after the
upgrade. Older binaries refuse a newer `_sqlx_migrations` ledger, and
`SKIP_MIGRATIONS=true` does not make the 1.11 schema safe for a July build.
To roll back, restore the backup taken in step 1.

## Database migrations

- The live build has applied 160 migration files (`001`–`175`, with gaps).
  1.11.11 adds 103 (`176`–`289`). Upstream did not modify or remove any
  migration the live database has applied, so sqlx raises no checksum or
  missing-migration error.
- **Visibility (migration 245).** It adds `repositories.visibility`
  (`public` / `internal` / `private`) and backfills it as
  `is_public ? 'public' : 'private'`. It never assigns `internal`. Every labs
  repository has `is_public = false`, so every one lands **private**. A
  trigger keeps `is_public` as a mirror. A legacy client writing
  `is_public = false` does not change `visibility`, and only a genuine
  `true → false` change narrows a repository. To verify after the deploy:
  `SELECT visibility, count(*) FROM repositories GROUP BY 1;` should return
  only `private`.
  `internal` means *any authenticated principal can read*, so never set it on
  a labs repository.
- **Migration 232** re-keys CI OIDC service accounts in place, keeping their
  ids, grants and history.
- **Migration 268** adds `oci_manifests`. A boot-time backfill fills it from
  the existing tags.

## Configuration

- **No env var the live deployment reads was renamed or removed.** Our fork
  aliases keep working: `S3_REQUEST_TIMEOUT_SECS` is now an alias of
  `S3_BULK_TIMEOUT_SECS` (the canonical name wins if both are set), and
  `S3_MAX_SINGLE_COPY_SIZE` is an alias of `S3_MAX_SINGLE_COPY_BYTES`.
- **Hard requirements:**
  - `JWT_SECRET` must be at least 32 characters and about 128 bits of
    entropy, or the backend exits. The live secrets are 128 characters.
  - A malformed `AK_WEBHOOK_SECRET_KEY`, `GRPC_BIND_IP`, `METRICS_BIND_IP` or
    `S3_PUBLIC_ENDPOINT` makes startup fail.
- **Review these:**
  - `S3_BULK_TIMEOUT_SECS` (default 1800) and `S3_CONTROL_TIMEOUT_SECS`
    (default 30). Our `S3_RETRY_TIMEOUT_SECS` defaults to 3600.
  - `S3_MULTIPART_CHUNK_SIZE`, `S3_MULTIPART_MAX_IN_FLIGHT_PARTS` and
    `S3_MULTIPART_TARGET_PARTS` are honoured again (fork carry).
  - `HTTP_HEADER_READ_TIMEOUT_SECS` (default 10) also closes idle HTTP/1
    keep-alive connections after 10 s. Keep the ingress upstream idle timeout
    below that.
  - `OCI_UPLOAD_COMPLETION_RETRY_WAIT_SECS` (default 30) and the ingress
    timeouts in `docs/operations/oci-reverse-proxy-timeouts.md`.
  - `UPLOAD_MIN_PROGRESS_BYTES` / `UPLOAD_PROGRESS_WINDOW_SECS`. Uploads slower
    than 16 KiB per 60 s get 408, which matters for large kaniko pushes over
    slow links.
  - `UPLOAD_MEMORY_BUDGET_BYTES` defaults to half the cgroup memory limit.
  - `RATE_LIMIT_TOKEN_EXCHANGE_PER_WINDOW` (default 300) limits `/v2/token`.
    A CI burst that exchanges tokens per job can hit it.
  - `RATE_LIMIT_TRUSTED_PROXY_CIDRS` and `XFF_TRUSTED_FOR_AUTHZ` (default
    false) decide whether client IPs from the ingress are trusted.
  - `ENABLE_SWAGGER` is now opt-in. `/swagger-ui` returns 404 without it.
  - `AK_GUEST_ACCESS_ENABLED`, when set, also gates anonymous `/v2`.
  - `UPSTREAM_PRIVATE_IP_ALLOWLIST` / `UPSTREAM_ALLOW_PRIVATE_IPS` (1.11.11
    #5095). A Remote whose upstream is a private-IP literal answers
    `502 UPSTREAM_DESTINATION_REFUSED` until it is admitted.
- **Proxy cache goes cold (#3454).** Cache keys gain a per-deployment segment
  (`proxy-cache/<peer-instance-id>/…`), and pre-upgrade entries are not read
  again. After the deploy, every Remote refetches from upstream on first use,
  so expect a burst of upstream traffic and possible Docker Hub anonymous rate
  limits. The old objects stay in the bucket until a cache purge. A
  credentialed Docker Hub Remote also needs
  `oci_trusted_bearer_realms: ["https://auth.docker.io"]` (#3591), or private
  pulls get 502 and public pulls fall under anonymous limits.
- **Metrics.** `ak_http_request_duration_seconds` is now a histogram, so
  `quantile=` series are gone. Use `histogram_quantile()` over `_bucket`.

## Authorization changes that can lock callers out

These are the intended fix for the `/v2` gap, but they also change who can
pull:

- `/v2` reads (manifests, blobs, tags/list, referrers, `_catalog`) require a
  real `read` grant: a user grant, a group grant, or a role assignment.
  Private repositories answer an ungranted caller with
  `404 NAME_UNKNOWN`, the same response as a missing repository. Any pull
  credential that only worked through the old hole stops working.
- API tokens need the `read:artifacts` scope to pull. A `write:artifacts`-only
  token gets `403 DENIED` on pull.
- An API token's repository allow-list (`api_token_repositories` or
  `repo_selector`) is enforced on Basic `user:TOKEN`, raw `Bearer TOKEN` and
  `/v2/token` JWTs. An unparseable `repo_selector` now means *no
  repositories*.
- REST reads require the `read` action, and several REST mutations now check
  the repository action. Repository management needs `write:repositories`.

## Web UI

The live web UI is 1.5.8 (2026-07-15). The backend repo has no compatibility
matrix. The web UI follows its own release cadence, and its 1.11.x releases
are written against backend 1.11.x. **Deploy `artifact-keeper-web` 1.11.2**
(2026-10-29, the latest). Web 1.5.8 predates several 1.11 API changes:
token-mint endpoints reject unknown fields, malformed bodies return 400 instead
of 422, the peer subscription shape changed, `format` values are now
`conda_native` / `wasm_oci` / `helm_oci`, webhook headers are redacted, and
cookie-authenticated mutations need the CSRF header.
