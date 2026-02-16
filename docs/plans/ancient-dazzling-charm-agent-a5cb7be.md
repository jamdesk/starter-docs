# Critical Review: `jamdesk init` Starter-Docs Download Plan

Review of `/Users/everdome/engineering/projects/jamdesk/docs/plans/ancient-dazzling-charm.md`

---

## Observations

1. **The plan file is named `ancient-dazzling-charm.md`**, violating the CLAUDE.md requirement of `YYYY-MM-DD-<descriptive-name>.md`. Should be something like `2026-02-12-init-starter-download.md`.

2. **The current `init.ts` has two code paths** (lines 30-87): one copies from `templates/` directory, the other creates inline files if templates are missing. The plan says "rewrite" but does not specify what happens to the inline fallback path.

3. **The starter-docs `docs.json` has no `hostAtDocs` field.** The `dev.ts` logic at line 690: `const basePath = hostAtDocs ? '/docs' : '';`. When `hostAtDocs` is absent/falsy, `basePath = ''`, meaning pages serve at `http://localhost:3000/introduction`, NOT `http://localhost:3000/docs/introduction`. The current init success message at line 104 says `http://localhost:3000/docs` -- this is **wrong for the current code too**, but the plan perpetuates the error.

4. **`images/` directory contains only `.gitkeep`** -- no actual binary files. Template substitution running on this is harmless but wasteful.

5. **The starter-docs contain 28+ references to "Acme"** across MDX files. The plan says "MDX content keeps Acme references as realistic sample content." This is a deliberate choice, not a gap.

6. **The `docs/` and `docs/plans/` directories exist in the starter** and would be extracted into user projects. These are repo-scaffolding artifacts, not documentation content.

7. **The `.gitignore` in starter-docs** contains `.DS_Store`, `node_modules/`, `.env`, `.env.local`. This is useful for user projects.

8. **The OpenAPI spec `openapi/acme.yaml`** is referenced by `docs.json` as `"/openapi/acme.yaml"`. This file will be included in the download but is branded "Acme".

9. **The repo is ~400KB including `.git`**. The tarball will be much smaller (~30KB as the plan states).

10. **The CLI already uses `execFileSync` for npm commands** (update.ts) and `execSync` for shell commands (deploy, dev). There is precedent for shelling out to system binaries.

---

## Risks & Failure Modes

### 1. Windows `tar` Portability -- REAL RISK

**Fact:** The `package.json` lists `"os": ["darwin", "linux", "win32"]`. Windows 10 build 17063+ (April 2018 update) ships `tar.exe` (BSD tar via libarchive). However:

- **Windows `tar.exe` does support `--strip-components`** (it is libarchive-based, not GNU tar).
- **BUT `execFileSync('tar', ...)` on Windows will find `tar.exe` in `C:\Windows\System32`** only if that is on PATH. In restricted enterprise environments or WSL edge cases, it may not be available.
- **The `xzf` flag ordering matters.** BSD tar on Windows expects `tar -xzf <file> -C <dir>`. The plan writes `['xzf', path, '-C', dir, '--strip-components=1']` -- this is missing the leading dash. Should be `['-xzf', path, '-C', dir, '--strip-components=1']` or `['xf', path, '-C', dir, '--strip-components=1']` (gzip decompression is auto-detected by libarchive).

**Recommendation:** The `tar` approach is acceptable. The CLI already shells out to `npm` on all platforms. Add the dash prefix. If Windows support is truly critical, consider a pure-JS fallback, but this is probably over-engineering for the user base. The plan should at minimum document this assumption.

### 2. GitHub API Rate Limiting -- CONFIRMED RISK

**Fact:** `api.github.com/repos/.../tarball/main` counts against the 60 req/hour unauthenticated limit. This is shared across ALL GitHub API calls from the user's IP.

**Impact:** A user running `jamdesk init` in a CI pipeline or classroom setting (shared IP/NAT) could hit this trivially. The error message from GitHub will be a 403 with a JSON body, not a tarball -- the code needs to handle this gracefully and not try to extract a JSON error as a tarball.

**Recommendation:** Use `https://codeload.github.com/jamdesk/starter-docs/tar.gz/main` instead. This is the direct download endpoint, has no API rate limits, and returns the same tarball. The redirect from `api.github.com` goes here anyway -- the plan is doing one extra HTTP hop for no benefit.

### 3. `hostAtDocs` and the Success Message -- CONFIRMED BUG

**Evidence:** `dev.ts` line 690 sets `basePath = ''` when `hostAtDocs` is falsy. The starter-docs `docs.json` does NOT set `hostAtDocs`. So `jamdesk dev` will serve at `http://localhost:3000/introduction`, not `http://localhost:3000/docs`.

The plan's success message says `http://localhost:3000/docs`. This is incorrect and will confuse every new user on their first run.

**Options:**
- A) Add `"hostAtDocs": true` to the starter-docs `docs.json` (makes the success message correct, matches what users deploying to `slug.jamdesk.app/docs` will need)
- B) Change the success message to `http://localhost:3000/introduction` (matches current behavior)
- C) Read the actual `docs.json` after extraction and compute the correct URL

Option C is most robust. The current `init.ts` hardcodes the URL, which is already wrong for projects without `hostAtDocs`.

### 4. Extracted `docs/` and `docs/plans/` Directories -- UNWANTED FILES

The starter-docs repo has a `docs/` directory with a `plans/` subdirectory. These are repo-management artifacts (for Claude Code plans), not documentation content. They will be extracted into the user's project, creating confusing empty directories.

**Recommendation:** Add `docs/` to the exclusion list alongside `README.md` and `LICENSE`. Or add a `.jamdeskignore` or similar mechanism. Or just remove `docs/` from the starter-docs repo itself -- it should not be there.

### 5. Template Substitution Scope is Too Narrow

The plan only substitutes `name` and `description` in `docs.json`. But:

- `openapi/acme.yaml` has "Acme" throughout
- `api.openapi` references `/openapi/acme.yaml` -- a file named "acme"
- The navbar button label is "Get Started" -- fine as-is
- 28+ "Acme" references in MDX content

Leaving Acme everywhere is the stated intent, but it creates a disconnect: docs.json says "My Docs" while all content says "Acme Payments API." Users will need to grep-and-replace extensively. This is not necessarily wrong -- the plan should just be explicit that this is the expected UX and document it in the success message.

### 6. No Happy-Path Test for Tarball Extraction -- REAL GAP

The plan tests network failures, non-200 responses, and timeouts. It does NOT test:
- Successful download + extraction produces the expected file tree
- `--strip-components=1` works correctly
- README.md and LICENSE are excluded
- Template substitution produces valid JSON

Without a happy-path test, you could ship code where the extraction logic silently fails or produces wrong output. You can create a small `.tar.gz` fixture in the test fixtures directory and test against it.

### 7. Temp Directory Cleanup on Windows

`execFileSync` can throw. The plan says "clean up temp dir in `finally` block" which is correct. However, on Windows, files extracted by `tar.exe` may have open handles or permissions that prevent `fs.rmSync`/`fs.removeSync` from succeeding immediately. This is a minor edge case but should use `{ recursive: true, force: true }`.

### 8. Response Body Handling for Non-200

If GitHub returns a 403 (rate limit) or 302 (redirect), the `fetch` response body is JSON or HTML, not a tarball. The plan says "non-200 response" returns `{ success: false }`, but it needs to ensure it does NOT write the error response body to disk and try to extract it. The check must happen BEFORE writing to the temp file.

### 9. Missing `--offline` Flag

The plan asks about this. My assessment: it is not needed yet. The fallback to bundled templates serves this purpose. Adding `--offline` is premature -- you can always add it later if users ask. The plan's fallback design already handles the use case.

### 10. `.gitignore` Should Be Included

The starter-docs `.gitignore` contains useful entries (`.DS_Store`, `node_modules/`, `.env`, `.env.local`). This is exactly what a new project needs. It should NOT be excluded. The plan currently does not exclude it, but it also does not explicitly include it -- tar will extract it by default, which is correct.

**Caveat:** Some users running `jamdesk init` inside an existing git repo may already have a `.gitignore`. The extraction will overwrite it. This should be documented or handled (e.g., merge or skip if exists).

### 11. The `$schema` Field Must Be Preserved

The plan says `personalizeDocsJson` only touches `name` and `description`. But the implementation approach (read as JSON, modify, write back) will reformat the JSON. Specifically:
- Trailing newlines may change
- Key ordering depends on `JSON.stringify` behavior (inserts in order)
- The `$schema` field being first is conventional but `JSON.stringify` preserves insertion order

This is fine as long as the implementation reads the object, modifies in-place, and writes with `{ spaces: 2 }`. Just be aware it will normalize formatting.

### 12. Binary File Safety for Template Substitution

The plan asks about this. **Non-issue.** The `images/` directory only contains `.gitkeep` (a zero-byte file). `personalizeDocsJson` only operates on `docs.json`, not other files. No binary file will be string-replaced.

### 13. The `description` Default

Setting description to `"<Name> documentation"` (e.g., "My Docs documentation") is slightly redundant when the name already contains "Docs". The original is "Acme Payments API documentation" which is more specific.

**Recommendation:** Just set `description` to the same value as `name`, or leave the Acme description and let users change it. `"My Docs documentation"` is awkward.

---

## Blind Spots

### A. The Starter-Docs Repo May Change

The plan hardcodes `main` branch. If the starter-docs repo structure changes (new directories, renamed files, different docs.json schema), the init command could produce broken projects. There is no version pinning.

**Recommendation:** Use a tagged release (`/tarball/v1.0.0`) instead of `main`. Bump the tag in the CLI when the starter-docs changes. This decouples CLI releases from starter-docs changes.

### B. No Cache / Re-download on Every Init

Every `jamdesk init` hits GitHub. If a user creates 3 projects in a row (common when experimenting), that is 3 downloads of the same tarball. The plan should consider caching the tarball in `~/.jamdesk/` with a TTL (like the version check cache in `version.ts`).

### C. Proxy and Corporate Firewalls

`fetch()` in Node.js does not respect `HTTP_PROXY`/`HTTPS_PROXY` environment variables by default. Users behind corporate proxies will get timeout errors and fall back to the bare-bones template with no indication of why.

**Recommendation:** At minimum, log a warning when the download fails that mentions proxy settings. Ideally, use `undici`'s proxy support or document this limitation.

### D. The `.git` Directory in the Tarball

GitHub tarballs do NOT include `.git/`. This is correct -- users should `git init` their own repo. But the plan does not mention suggesting `git init` in the success message, which would be helpful.

### E. Concurrent Writes Race Condition

If `projectName` is a relative path like `../../somewhere`, `fs.ensureDir` will create it. The plan does not validate the target path. This is also a pre-existing issue in the current `init.ts` but worth noting.

---

## Alternative Approaches

### 1. Bundle the Starter in the npm Package

Instead of downloading at runtime, include the starter-docs files in the `templates/` directory that ships with the npm package. The `package.json` `files` array already includes `"templates/"`.

**Pros:** Zero network dependency, instant init, works offline, no rate limits, no proxy issues, no version skew.
**Cons:** Increases npm package size by ~30KB (trivial). Requires CLI release to update starter content.

**This is significantly simpler and more reliable.** The download approach is solving a problem (keeping starter-docs decoupled) that may not be worth the complexity. The starter-docs will likely change infrequently -- it is a template, not a living document.

### 2. Use `degit` Pattern (No Dependencies)

Instead of `fetch` + `tar`, use the same approach as `degit`: download the zip from GitHub (`/archive/main.zip`) and extract with Node.js's built-in `zlib` module. No system dependency on `tar`.

**Pros:** Pure JS, no `tar` binary dependency, works everywhere Node.js works.
**Cons:** Slightly more code. Zip extraction requires either a dependency or manual implementation.

### 3. Use `npm init` / `create-` Package Pattern

Publish `create-jamdesk` to npm so users can run `npm create jamdesk` or `npx create-jamdesk`. The create package contains the full template.

**Pros:** Standard Node.js pattern, users already know it, npm handles distribution.
**Cons:** Another package to maintain, users already have `jamdesk` installed.

---

## Assessment

The plan is functional and addresses a real gap (bare-bones 3-file init vs. full starter template). However, it introduces network dependency and platform-specific behavior (`tar`) where a simpler approach exists: just bundle the starter-docs in `templates/` and ship them with the npm package. The 30KB size increase is negligible. The download approach adds 5+ failure modes (network, rate limits, proxy, tar binary, temp file cleanup) that the bundled approach has zero of. If decoupling from CLI releases is important, the download approach is fine but needs the fixes above: use `codeload.github.com` not `api.github.com`, pin to a tag not `main`, fix the `hostAtDocs` success message, exclude the `docs/` directory, add happy-path tests, and add the dash prefix to tar flags.
