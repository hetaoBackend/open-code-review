# `ocr review --upstream` Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a `--upstream` flag so a fork user can review their current branch against the original (upstream) repository, whether or not upstream is configured as a git remote.

**Architecture:** `--upstream` is a pure front-end resolution step. In `runReview`, before ref validation, `resolveUpstream` turns the upstream spec (remote name or URL) into a concrete `from` ref (`<remote>/<branch>` or a fetched SHA) and defaults `to` to `HEAD`. Everything downstream reuses the existing ModeRange (`merge-base(from,to)..to`) path unchanged.

**Tech Stack:** Go 1.26, standard `git` CLI via the existing `runGitCmd` helper (`cmd/opencodereview/git.go`) and `internal/gitcmd.Runner`. Tests use temp git repos (pattern from `internal/diff/git_test.go`).

---

## File Structure

- **Create** `cmd/opencodereview/upstream.go` — `resolveUpstream` + helpers (`isUpstreamURL`, `upstreamRemoteExists`, `discoverDefaultBranch`, `parseSymrefDefaultBranch`, `defaultTo`).
- **Create** `cmd/opencodereview/upstream_test.go` — unit + integration tests for the above.
- **Create** `cmd/opencodereview/flags_test.go` — `parseReviewFlags` validation tests for the new flags.
- **Modify** `cmd/opencodereview/flags.go` — new fields, flag registration, validation, help text.
- **Modify** `cmd/opencodereview/review_cmd.go` — call `resolveUpstream` in `runReview`.
- **Modify** `internal/diff/git.go` — augment merge-base error with an unshallow hint.
- **Modify** `README.md` — add a fork-workflow section.

All `git` invocations in `upstream.go` use the package-local `runGitCmd(repoDir, args...)` helper (matches `validateReviewRefs` / `getCommitMessage`). Every ref/target arg is passed after `--end-of-options` to stay consistent with the repo's option-injection hardening.

---

## Task 1: Flags — fields, registration, validation

**Files:**
- Modify: `cmd/opencodereview/flags.go` (struct ~`97-113`, registration ~`120-133`, validation ~`144-157`, help ~`181-225`)
- Test: `cmd/opencodereview/flags_test.go` (create)

- [ ] **Step 1: Write the failing tests**

Create `cmd/opencodereview/flags_test.go`:

```go
package main

import (
	"strings"
	"testing"
)

func TestParseReviewFlagsUpstreamConflictsWithFrom(t *testing.T) {
	_, err := parseReviewFlags([]string{"--upstream", "upstream", "--from", "main"})
	if err == nil || !strings.Contains(err.Error(), "--upstream cannot be combined") {
		t.Fatalf("expected conflict error, got: %v", err)
	}
}

func TestParseReviewFlagsUpstreamConflictsWithCommit(t *testing.T) {
	_, err := parseReviewFlags([]string{"--upstream", "upstream", "--commit", "abc123"})
	if err == nil || !strings.Contains(err.Error(), "--upstream cannot be combined") {
		t.Fatalf("expected conflict error, got: %v", err)
	}
}

func TestParseReviewFlagsUpstreamBranchRequiresUpstream(t *testing.T) {
	_, err := parseReviewFlags([]string{"--upstream-branch", "develop"})
	if err == nil || !strings.Contains(err.Error(), "--upstream-branch requires --upstream") {
		t.Fatalf("expected requires-upstream error, got: %v", err)
	}
}

func TestParseReviewFlagsNoFetchRequiresUpstream(t *testing.T) {
	_, err := parseReviewFlags([]string{"--no-fetch"})
	if err == nil || !strings.Contains(err.Error(), "--no-fetch requires --upstream") {
		t.Fatalf("expected requires-upstream error, got: %v", err)
	}
}

func TestParseReviewFlagsUpstreamOK(t *testing.T) {
	opts, err := parseReviewFlags([]string{"--upstream", "upstream", "--to", "feature"})
	if err != nil {
		t.Fatalf("unexpected error: %v", err)
	}
	if opts.upstream != "upstream" || opts.to != "feature" {
		t.Fatalf("unexpected opts: %+v", opts)
	}
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `go test ./cmd/opencodereview/ -run TestParseReviewFlags -v`
Expected: COMPILE FAIL — `opts.upstream` / `opts.noFetch` undefined (fields don't exist yet).

- [ ] **Step 3: Add struct fields**

In `cmd/opencodereview/flags.go`, inside `type reviewOptions struct`, add after `commit string` (line ~103):

```go
	upstream       string // --upstream: remote name or git URL to review against (fork workflow)
	upstreamBranch string // --upstream-branch: branch on --upstream (default: its default branch)
	noFetch        bool   // --no-fetch: with --upstream, skip network fetch and use local ref
```

- [ ] **Step 4: Register the flags**

In `parseReviewFlags`, after the `--preview` registration (line ~133), add:

```go
	a.StringVar(&opts.upstream, "upstream", "", "upstream remote name or git URL to review the current branch against (fork workflow)")
	a.StringVar(&opts.upstreamBranch, "upstream-branch", "", "branch on --upstream to compare against (default: upstream's default branch)")
	a.BoolVar(&opts.noFetch, "no-fetch", false, "with --upstream: skip network fetch and use the local remote-tracking ref")
```

- [ ] **Step 5: Add validation**

In `parseReviewFlags`, after the existing `if opts.from != "" && opts.to == "" { ... }` block (line ~157), add:

```go
	if opts.upstream != "" {
		if opts.from != "" || opts.commit != "" {
			return opts, fmt.Errorf("--upstream cannot be combined with --from or --commit")
		}
	} else {
		if opts.upstreamBranch != "" {
			return opts, fmt.Errorf("--upstream-branch requires --upstream")
		}
		if opts.noFetch {
			return opts, fmt.Errorf("--no-fetch requires --upstream")
		}
	}
```

- [ ] **Step 6: Run tests to verify they pass**

Run: `go test ./cmd/opencodereview/ -run TestParseReviewFlags -v`
Expected: PASS (all 5).

- [ ] **Step 7: Update help text**

In `printReviewUsage` Examples block (after the `--commit` example, ~line 197), add:

```go
  # Review current branch against an upstream repo (fork workflow)
  ocr review --upstream upstream
  ocr review --upstream https://github.com/orig/repo --upstream-branch main
```

In the Flags list (alphabetical area, ~line 217), add:

```go
  --upstream string       upstream remote name or git URL to review the current branch against
  --upstream-branch string  branch on --upstream to compare against (default: its default branch)
  --no-fetch              with --upstream: skip network fetch, use local remote-tracking ref
```

- [ ] **Step 8: Commit**

```bash
git add cmd/opencodereview/flags.go cmd/opencodereview/flags_test.go
git commit -m "feat(flags): add --upstream/--upstream-branch/--no-fetch review flags"
```

---

## Task 2: URL detection + remote existence helpers

**Files:**
- Create: `cmd/opencodereview/upstream.go`
- Create/Modify: `cmd/opencodereview/upstream_test.go`

- [ ] **Step 1: Write the failing test**

Create `cmd/opencodereview/upstream_test.go`:

```go
package main

import "testing"

func TestIsUpstreamURL(t *testing.T) {
	cases := []struct {
		in   string
		want bool
	}{
		{"https://github.com/o/r", true},
		{"git://example.com/r.git", true},
		{"ssh://git@example.com/r.git", true},
		{"file:///tmp/bare.git", true},
		{"git@github.com:o/r.git", true},
		{"/abs/path/bare.git", true},
		{"./rel/bare.git", true},
		{"../rel/bare.git", true},
		{"upstream", false},
		{"origin", false},
		{"my-remote", false},
	}
	for _, c := range cases {
		if got := isUpstreamURL(c.in); got != c.want {
			t.Errorf("isUpstreamURL(%q) = %v, want %v", c.in, got, c.want)
		}
	}
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `go test ./cmd/opencodereview/ -run TestIsUpstreamURL -v`
Expected: COMPILE FAIL — `isUpstreamURL` undefined.

- [ ] **Step 3: Create upstream.go with the helpers**

Create `cmd/opencodereview/upstream.go` (import only `strings` for now — Go errors on unused imports; `fmt`/`os` are added in Tasks 3/4 as the functions that use them land):

```go
package main

import (
	"strings"
)

// isUpstreamURL reports whether spec looks like a git URL or local path rather
// than the name of a configured remote.
func isUpstreamURL(spec string) bool {
	if strings.Contains(spec, "://") {
		return true
	}
	if strings.HasPrefix(spec, "/") || strings.HasPrefix(spec, "./") ||
		strings.HasPrefix(spec, "../") || strings.HasPrefix(spec, "~") {
		return true
	}
	// scp-like syntax: user@host:path (a ':' that precedes any '/').
	if i := strings.Index(spec, ":"); i > 0 {
		slash := strings.Index(spec, "/")
		if (slash == -1 || i < slash) && strings.Contains(spec[:i], "@") {
			return true
		}
	}
	return false
}

// upstreamRemoteExists reports whether name is a configured git remote in repoDir.
func upstreamRemoteExists(repoDir, name string) bool {
	out, err := runGitCmd(repoDir, "remote")
	if err != nil {
		return false
	}
	for _, line := range strings.Split(string(out), "\n") {
		if strings.TrimSpace(line) == name {
			return true
		}
	}
	return false
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `go test ./cmd/opencodereview/ -run TestIsUpstreamURL -v`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add cmd/opencodereview/upstream.go cmd/opencodereview/upstream_test.go
git commit -m "feat(upstream): add URL detection and remote-existence helpers"
```

---

## Task 3: Default-branch discovery (ls-remote symref)

**Files:**
- Modify: `cmd/opencodereview/upstream.go`
- Modify: `cmd/opencodereview/upstream_test.go`

- [ ] **Step 1: Write the failing test**

Append to `cmd/opencodereview/upstream_test.go`:

```go
func TestParseSymrefDefaultBranch(t *testing.T) {
	out := "ref: refs/heads/main\tHEAD\n" +
		"a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0\tHEAD\n"
	branch, err := parseSymrefDefaultBranch(out)
	if err != nil {
		t.Fatalf("unexpected error: %v", err)
	}
	if branch != "main" {
		t.Fatalf("got %q, want main", branch)
	}
}

func TestParseSymrefDefaultBranchMaster(t *testing.T) {
	out := "ref: refs/heads/master\tHEAD\n0000\tHEAD\n"
	branch, err := parseSymrefDefaultBranch(out)
	if err != nil || branch != "master" {
		t.Fatalf("got %q, err %v; want master", branch, err)
	}
}

func TestParseSymrefDefaultBranchMissing(t *testing.T) {
	if _, err := parseSymrefDefaultBranch("0000\tHEAD\n"); err == nil {
		t.Fatal("expected error when no symref line present")
	}
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `go test ./cmd/opencodereview/ -run TestParseSymrefDefaultBranch -v`
Expected: COMPILE FAIL — `parseSymrefDefaultBranch` undefined.

- [ ] **Step 3: Add the parser and discovery function**

First update the import block in `cmd/opencodereview/upstream.go` to add `fmt`:

```go
import (
	"fmt"
	"strings"
)
```

Then append to `cmd/opencodereview/upstream.go`:

```go
// parseSymrefDefaultBranch extracts the default branch name from the output of
// `git ls-remote --symref <target> HEAD`, e.g. a line "ref: refs/heads/main\tHEAD".
func parseSymrefDefaultBranch(lsRemoteOut string) (string, error) {
	for _, line := range strings.Split(lsRemoteOut, "\n") {
		line = strings.TrimSpace(line)
		if !strings.HasPrefix(line, "ref:") {
			continue
		}
		fields := strings.Fields(line) // ["ref:", "refs/heads/main", "HEAD"]
		if len(fields) >= 2 && strings.HasPrefix(fields[1], "refs/heads/") {
			return strings.TrimPrefix(fields[1], "refs/heads/"), nil
		}
	}
	return "", fmt.Errorf("no default branch (HEAD symref) found in ls-remote output")
}

// discoverDefaultBranch queries target (a remote name or URL) for its default
// branch via `git ls-remote --symref`.
func discoverDefaultBranch(repoDir, target string) (string, error) {
	out, err := runGitCmd(repoDir, "ls-remote", "--symref", "--end-of-options", target, "HEAD")
	if err != nil {
		return "", fmt.Errorf("git ls-remote %q failed: %s", target, strings.TrimSpace(string(out)))
	}
	branch, perr := parseSymrefDefaultBranch(string(out))
	if perr != nil {
		return "", fmt.Errorf("determine default branch for %q: %w", target, perr)
	}
	return branch, nil
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `go test ./cmd/opencodereview/ -run TestParseSymrefDefaultBranch -v`
Expected: PASS (all 3).

- [ ] **Step 5: Commit**

```bash
git add cmd/opencodereview/upstream.go cmd/opencodereview/upstream_test.go
git commit -m "feat(upstream): discover default branch via ls-remote --symref"
```

---

## Task 4: `resolveUpstream` orchestration

**Files:**
- Modify: `cmd/opencodereview/upstream.go`
- Modify: `cmd/opencodereview/upstream_test.go`

- [ ] **Step 1: Write the failing integration tests**

Append to `cmd/opencodereview/upstream_test.go` (note the new imports — update the import block at the top of the file to include these):

```go
import (
	"context"
	"os"
	"os/exec"
	"path/filepath"
	"regexp"
	"strings"
	"testing"

	"github.com/open-code-review/open-code-review/internal/diff"
	"github.com/open-code-review/open-code-review/internal/gitcmd"
)

// initUpstreamAndFork builds: a bare "upstream" repo (branch main, one commit),
// and a "fork" clone of it with origin -> bare and a local "feature" branch that
// modifies app.go. Returns the bare repo path and the fork working dir.
func initUpstreamAndFork(t *testing.T) (upstreamBare, fork string) {
	t.Helper()
	root := t.TempDir()
	upstreamWork := filepath.Join(root, "upstream-work")
	upstreamBare = filepath.Join(root, "upstream.git")
	fork = filepath.Join(root, "fork")

	mustGit := func(dir string, args ...string) {
		t.Helper()
		cmd := exec.Command("git", args...)
		cmd.Dir = dir
		if out, err := cmd.CombinedOutput(); err != nil {
			t.Fatalf("git %v in %s failed: %v\n%s", args, dir, err, out)
		}
	}

	if err := os.MkdirAll(upstreamWork, 0o755); err != nil {
		t.Fatal(err)
	}
	mustGit(upstreamWork, "init", "-q", "-b", "main")
	mustGit(upstreamWork, "config", "user.email", "up@example.com")
	mustGit(upstreamWork, "config", "user.name", "Up")
	mustGit(upstreamWork, "config", "commit.gpgsign", "false")
	if err := os.WriteFile(filepath.Join(upstreamWork, "app.go"),
		[]byte("package app\n\nfunc A() int { return 1 }\n"), 0o644); err != nil {
		t.Fatal(err)
	}
	mustGit(upstreamWork, "add", ".")
	mustGit(upstreamWork, "commit", "-q", "-m", "upstream initial")

	mustGit(root, "clone", "-q", "--bare", upstreamWork, upstreamBare)

	mustGit(root, "clone", "-q", upstreamBare, fork)
	mustGit(fork, "config", "user.email", "me@example.com")
	mustGit(fork, "config", "user.name", "Me")
	mustGit(fork, "config", "commit.gpgsign", "false")
	mustGit(fork, "checkout", "-q", "-b", "feature")
	if err := os.WriteFile(filepath.Join(fork, "app.go"),
		[]byte("package app\n\nfunc A() int { return 2 }\n"), 0o644); err != nil {
		t.Fatal(err)
	}
	mustGit(fork, "add", ".")
	mustGit(fork, "commit", "-q", "-m", "my change")

	return upstreamBare, fork
}

// assertReviewsAppChange builds a range provider for from..to and asserts the
// only changed file is app.go.
func assertReviewsAppChange(t *testing.T, repoDir, from, to string) {
	t.Helper()
	runner := gitcmd.New(4)
	p := diff.NewProvider(repoDir, from, to, runner)
	diffs, err := p.GetDiff(context.Background())
	if err != nil {
		t.Fatalf("GetDiff(%s..%s): %v", from, to, err)
	}
	if len(diffs) != 1 {
		t.Fatalf("expected 1 changed file, got %d: %+v", len(diffs), diffs)
	}
}

func TestResolveUpstreamRemoteNameFetches(t *testing.T) {
	bare, fork := initUpstreamAndFork(t)
	cmd := exec.Command("git", "remote", "add", "upstream", bare)
	cmd.Dir = fork
	if out, err := cmd.CombinedOutput(); err != nil {
		t.Fatalf("add upstream remote: %v\n%s", err, out)
	}

	from, to, err := resolveUpstream(fork, reviewOptions{upstream: "upstream"})
	if err != nil {
		t.Fatalf("resolveUpstream: %v", err)
	}
	if from != "upstream/main" {
		t.Fatalf("from = %q, want upstream/main", from)
	}
	if to != "HEAD" {
		t.Fatalf("to = %q, want HEAD", to)
	}
	assertReviewsAppChange(t, fork, from, to)
}

func TestResolveUpstreamURLFetches(t *testing.T) {
	bare, fork := initUpstreamAndFork(t)

	from, to, err := resolveUpstream(fork, reviewOptions{upstream: "file://" + bare})
	if err != nil {
		t.Fatalf("resolveUpstream: %v", err)
	}
	if !regexp.MustCompile(`^[0-9a-f]{40}$`).MatchString(from) {
		t.Fatalf("from = %q, want a 40-hex SHA", from)
	}
	if to != "HEAD" {
		t.Fatalf("to = %q, want HEAD", to)
	}
	assertReviewsAppChange(t, fork, from, to)
}

func TestResolveUpstreamNoFetchUsesLocal(t *testing.T) {
	_, fork := initUpstreamAndFork(t)
	// origin/main + origin/HEAD already exist from the clone; no network needed.
	from, to, err := resolveUpstream(fork, reviewOptions{upstream: "origin", noFetch: true})
	if err != nil {
		t.Fatalf("resolveUpstream: %v", err)
	}
	if from != "origin/main" || to != "HEAD" {
		t.Fatalf("got from=%q to=%q, want origin/main HEAD", from, to)
	}
}

func TestResolveUpstreamUnknownRemote(t *testing.T) {
	_, fork := initUpstreamAndFork(t)
	_, _, err := resolveUpstream(fork, reviewOptions{upstream: "nope"})
	if err == nil || !strings.Contains(err.Error(), "no git remote named") {
		t.Fatalf("expected unknown-remote error, got: %v", err)
	}
}

func TestResolveUpstreamNoFetchRejectsURL(t *testing.T) {
	bare, fork := initUpstreamAndFork(t)
	_, _, err := resolveUpstream(fork, reviewOptions{upstream: "file://" + bare, noFetch: true})
	if err == nil || !strings.Contains(err.Error(), "--no-fetch") {
		t.Fatalf("expected --no-fetch/URL error, got: %v", err)
	}
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `go test ./cmd/opencodereview/ -run TestResolveUpstream -v`
Expected: COMPILE FAIL — `resolveUpstream` undefined.

- [ ] **Step 3: Implement `resolveUpstream` and `defaultTo`**

First update the import block in `cmd/opencodereview/upstream.go` to add `os`:

```go
import (
	"fmt"
	"os"
	"strings"
)
```

Then append to `cmd/opencodereview/upstream.go`:

```go
// defaultTo returns to, or "HEAD" when to is empty.
func defaultTo(to string) string {
	if to == "" {
		return "HEAD"
	}
	return to
}

// resolveUpstream turns an --upstream spec (remote name or URL) into a concrete
// range to review: it fetches the upstream branch as needed and returns a usable
// `from` ref plus the `to` ref (defaulting to HEAD). The result feeds the normal
// ModeRange path: merge-base(from,to)..to.
func resolveUpstream(repoDir string, opts reviewOptions) (from, to string, err error) {
	target := opts.upstream
	isURL := isUpstreamURL(target)

	if !isURL && !upstreamRemoteExists(repoDir, target) {
		return "", "", fmt.Errorf("--upstream %q: no git remote named %q; pass a git URL or run \"git remote add %s <url>\"", target, target, target)
	}
	if isURL && opts.noFetch {
		return "", "", fmt.Errorf("--no-fetch cannot be used with a URL upstream (%q); a URL must be fetched", target)
	}

	// Determine the upstream branch.
	branch := opts.upstreamBranch
	if branch == "" {
		if opts.noFetch {
			out, e := runGitCmd(repoDir, "rev-parse", "--abbrev-ref", "--end-of-options", target+"/HEAD")
			ref := strings.TrimSpace(string(out))
			if e != nil || !strings.HasPrefix(ref, target+"/") {
				return "", "", fmt.Errorf("--no-fetch: cannot determine default branch for %q locally; pass --upstream-branch", target)
			}
			branch = strings.TrimPrefix(ref, target+"/")
		} else {
			branch, err = discoverDefaultBranch(repoDir, target)
			if err != nil {
				return "", "", err
			}
		}
	}

	// Fetch unless asked not to.
	if !opts.noFetch {
		fmt.Fprintf(os.Stderr, "[ocr] fetching %s from %s...\n", branch, target)
		if out, e := runGitCmd(repoDir, "fetch", "--end-of-options", target, branch); e != nil {
			// For a configured remote, fall back to a local tracking ref if present.
			if !isURL {
				if _, verr := runGitCmd(repoDir, "rev-parse", "--verify", "--end-of-options", target+"/"+branch+"^{commit}"); verr == nil {
					fmt.Fprintf(os.Stderr, "[ocr] fetch failed (%s); using local %s/%s\n", strings.TrimSpace(string(out)), target, branch)
					return target + "/" + branch, defaultTo(opts.to), nil
				}
			}
			return "", "", fmt.Errorf("git fetch %s %s failed: %s", target, branch, strings.TrimSpace(string(out)))
		}
	}

	if isURL {
		out, e := runGitCmd(repoDir, "rev-parse", "--verify", "--end-of-options", "FETCH_HEAD^{commit}")
		if e != nil {
			return "", "", fmt.Errorf("resolve FETCH_HEAD after fetching %q: %s", target, strings.TrimSpace(string(out)))
		}
		return strings.TrimSpace(string(out)), defaultTo(opts.to), nil
	}
	return target + "/" + branch, defaultTo(opts.to), nil
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `go test ./cmd/opencodereview/ -run TestResolveUpstream -v`
Expected: PASS (all 5: RemoteNameFetches, URLFetches, NoFetchUsesLocal, UnknownRemote, NoFetchRejectsURL).

- [ ] **Step 5: Run the full package test suite**

Run: `go test ./cmd/opencodereview/ -v`
Expected: PASS (no regressions).

- [ ] **Step 6: Commit**

```bash
git add cmd/opencodereview/upstream.go cmd/opencodereview/upstream_test.go
git commit -m "feat(upstream): resolve remote/URL upstream into a review range"
```

---

## Task 5: Wire `resolveUpstream` into `runReview`

**Files:**
- Modify: `cmd/opencodereview/review_cmd.go` (after `resolveRepoDir`, ~line 48-54)

- [ ] **Step 1: Add the resolution call**

In `runReview`, between the `repoDir, err := resolveRepoDir(...)` block (ending line ~51) and `if err := validateReviewRefs(repoDir, opts); err != nil {` (line ~52), insert:

```go
	if opts.upstream != "" {
		from, to, uerr := resolveUpstream(repoDir, opts)
		if uerr != nil {
			return uerr
		}
		opts.from = from
		opts.to = to
	}
```

- [ ] **Step 2: Verify the build**

Run: `go build ./...`
Expected: success, no errors.

- [ ] **Step 3: Verify full test suite still passes**

Run: `go test ./...`
Expected: PASS across all packages.

- [ ] **Step 4: Manual smoke test against this very repo**

This repo's `origin` is the open-code-review GitHub remote, so we can exercise the resolution path without an LLM using `--preview`:

Run:
```bash
go run ./cmd/opencodereview review --upstream origin --upstream-branch main --no-fetch --preview
```
Expected: prints `[ocr] ...`-free preview output listing files that differ between `merge-base(origin/main, HEAD)` and HEAD (i.e. the files changed on the current branch). It must NOT error with "not a valid git ref" or "--to is required". (If `origin/main` isn't present locally, drop `--no-fetch` to fetch it first.)

- [ ] **Step 5: Commit**

```bash
git add cmd/opencodereview/review_cmd.go
git commit -m "feat(review): wire --upstream resolution into runReview"
```

---

## Task 6: Augment merge-base error with unshallow hint

**Files:**
- Modify: `internal/diff/git.go:115-117`
- Test: `internal/diff/git_test.go` (add)

- [ ] **Step 1: Write the failing test**

Append to `internal/diff/git_test.go`:

```go
// TestRangeMergeBaseErrorHasUnshallowHint verifies the range-mode error names
// the offending refs and hints at shallow clones when no common ancestor exists.
func TestRangeMergeBaseErrorHasUnshallowHint(t *testing.T) {
	repo := t.TempDir()
	runGitTest(t, repo, "init", "-q")
	runGitTest(t, repo, "config", "user.email", "test@example.com")
	runGitTest(t, repo, "config", "user.name", "Test User")
	runGitTest(t, repo, "config", "commit.gpgsign", "false")

	// Two unrelated root commits => no merge-base.
	if err := os.WriteFile(filepath.Join(repo, "a.txt"), []byte("a\n"), 0o644); err != nil {
		t.Fatal(err)
	}
	runGitTest(t, repo, "add", "a.txt")
	runGitTest(t, repo, "commit", "-q", "-m", "first")
	runGitTest(t, repo, "checkout", "-q", "--orphan", "other")
	runGitTest(t, repo, "rm", "-q", "-rf", ".")
	if err := os.WriteFile(filepath.Join(repo, "b.txt"), []byte("b\n"), 0o644); err != nil {
		t.Fatal(err)
	}
	runGitTest(t, repo, "add", "b.txt")
	runGitTest(t, repo, "commit", "-q", "-m", "unrelated")

	runner := gitcmd.New(4)
	p := NewProvider(repo, "master", "other", runner)
	_, err := p.GetDiff(context.Background())
	if err == nil {
		t.Fatal("expected merge-base error for unrelated histories")
	}
	if !strings.Contains(err.Error(), "merge-base") || !strings.Contains(err.Error(), "unshallow") {
		t.Fatalf("error missing merge-base/unshallow hint: %v", err)
	}
}
```

> Note: the default initial branch may be `master` or `main` depending on the host git config. If this test reports the branch ref as invalid, change `"master"` to the name printed by `git -C <repo> branch --show-current` on the first commit; the repo's other tests assume `master`.

- [ ] **Step 2: Run test to verify it fails**

Run: `go test ./internal/diff/ -run TestRangeMergeBaseErrorHasUnshallowHint -v`
Expected: FAIL — current error lacks the word "unshallow".

- [ ] **Step 3: Update the error message**

In `internal/diff/git.go`, in `GetDiff`'s `case ModeRange` (line ~115-117), change:

```go
		base := p.MergeBase(ctx)
		if base == "" {
			return nil, fmt.Errorf("cannot find merge-base between %s and %s", p.from, p.to)
		}
```

to:

```go
		base := p.MergeBase(ctx)
		if base == "" {
			return nil, fmt.Errorf("cannot find merge-base between %s and %s (if this is a shallow clone, run: git fetch --unshallow)", p.from, p.to)
		}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `go test ./internal/diff/ -run TestRangeMergeBaseErrorHasUnshallowHint -v`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add internal/diff/git.go internal/diff/git_test.go
git commit -m "feat(diff): hint at git fetch --unshallow on empty merge-base"
```

---

## Task 7: Document the fork workflow in README

**Files:**
- Modify: `README.md`

- [ ] **Step 1: Add a fork-workflow section**

Find the section that documents `ocr review` modes/usage (search README.md for `--from` or `--commit`). Immediately after it, add:

```markdown
### Reviewing a fork against upstream

When you work on a fork and want to review what your current branch changed
relative to the original repository, use `--upstream`:

```bash
# Against a configured remote (auto-fetches its default branch):
ocr review --upstream upstream

# Against the original repo by URL, without configuring a remote:
ocr review --upstream https://github.com/orig/repo

# Pick a specific upstream branch; default is HEAD vs upstream's default branch:
ocr review --upstream upstream --upstream-branch main

# Offline: use the already-fetched remote-tracking branch, no network:
ocr review --upstream upstream --no-fetch
```

`--upstream` reviews `merge-base(upstream, HEAD)..HEAD` — i.e. exactly the
commits your branch adds on top of upstream. Override the target with `--to`.
```

- [ ] **Step 2: Commit**

```bash
git add README.md
git commit -m "docs: document --upstream fork-review workflow"
```

---

## Self-Review notes (already applied)

- **Spec coverage:** §3 CLI → Task 1 + Task 7; §4 algorithm → Tasks 2–4; §5 code changes → Tasks 1/4/5/7; §6 error handling (unknown remote, fetch fallback, merge-base hint, anonymous URL fetch, fetch visibility) → Task 4 + Task 6; §7 testing → tests embedded in Tasks 1–4, 6.
- **Type consistency:** `resolveUpstream(repoDir string, opts reviewOptions) (from, to string, err error)` and `defaultTo` are used identically in Task 4 and Task 5. Field names `upstream` / `upstreamBranch` / `noFetch` match across flags.go, tests, and resolveUpstream.
- **No placeholders:** every code/test step contains complete code and an exact run command with expected result.
- **Known non-goal:** full `runReview` is not unit-tested end-to-end (needs an LLM endpoint); coverage stops at `resolveUpstream` + a `--preview` manual smoke test (Task 5 Step 4), which exercises the wiring without the model.
```
