# Bounty #4061 — Resolution Findings

**Issue:** [Bounty] [TEST BOUNTY] Fix typo in README
**Source:** https://github.com/jomaaabbe78-spec/bookningsapp/issues/1
**Branch:** `fix-bounty-4061`
**Outcome:** No code change. The bounty is stale — the reported defect does not exist in
this repository and was already fixed upstream before this task was published.

---

## 1. The reported defect

> There is a typo in README.md: "Welcom" should be "Welcome".

## 2. Verification

### 2.1 The typo does not exist in this repository

| Check | Command | Result |
|---|---|---|
| Text search | `grep -rn "Welcom" . --exclude-dir=.git` | Only hit is the quotation inside `BOUNTY_TASK.md` |
| README content | `grep -c "Welcome" README.md README.en.md web/README.md` | `0`, `0`, `0` — no "Welcome" text exists at all |
| History | `git log --all -S"Welcom"` | 0 commits, across every ref |

The three READMEs in this repo (`README.md`, `README.en.md`, `web/README.md`) contain no
"Welcome" section, so there is nothing to correct. This repository (`bounty-plaza`) is the
*bounty board* that mirrors upstream issues; the file named in the issue belongs to a
different project.

### 2.2 The upstream defect was already fixed and merged

| Fact | Value |
|---|---|
| Fix PR | jomaaabbe78-spec/bookningsapp#3 — "Fix: Add Welcome section to README (bounty #1)" |
| Merged at | 2026-10-20T20:50:42Z |
| Upstream `README.md` today | `## Welcome! 👋` / `Welcome to Bokningsapp ...` |
| Upstream issue #1 state | **OPEN** (never closed after the fix merged) |
| This bounty published | 2026-10-20T22:25:17Z — **1h35m after** the fix landed |

## 3. Root cause

`scripts/hunt_bounties.py` treats the upstream issue's `state` field as the ground truth for
"work still needed":

```python
SEARCH_QUERY = "label:bounty state:open is:issue sort:created ..."
```

The only other guard is `is_real_money()` (minimum $25) and a source-URL de-dupe against
already-published listings. There is no check of whether the linked issue already has a
merged pull request, and no staleness bound beyond `created:>2026-05-20`.

So the chain is:

1. `bookningsapp#1` was fixed by PR #3, but the owner merged the PR without closing the issue.
2. The issue therefore still matched `state:open` on every 5-minute crawl.
3. The crawler mirrored it here as #4061, a task for work that was already complete.

## 4. Why no patch was produced

The only edit that would make a diff appear is inserting "Welcom" and then correcting it, or
adding an unrelated file — i.e. manufacturing the appearance of a fix. That is a fabricated
change, not a fix, and it is precisely what this project's own rules prohibit:

- `RULES.md` → "提交不相关的代码 → 拉黑" (submitting unrelated code)
- `RULES.md` → "绝不能提空 PR" (never open an empty PR)

A duplicate PR was also not opened against the upstream repository. It already carries two
open PRs for this resolved issue (#2 and #4); #4 adds a 3-line `docs_update.md` that fixes
nothing. A third would be pure noise for that maintainer.

## 5. Recommended follow-ups (not applied — out of scope for this issue)

1. **Close #4061** as stale, and close `bookningsapp#1` upstream.
2. Add an "already fixed?" guard to `scripts/hunt_bounties.py` before `create_issue()`:

```python
def has_merged_fix(item):
    """Skip issues whose repo already references a merged PR for them."""
    url = item["html_url"].rstrip("/")
    data = json.loads(urllib.request.urlopen(urllib.request.Request(
        f"https://api.github.com/repos/{item['repository_url'].split('repos/')[-1]}"
        f"/issues/{item['number']}/timeline",
        headers={"Authorization": f"token {GH_TOKEN}",
                 "Accept": "application/vnd.github.mockingbird-preview+json"},
    )).read())
    for event in data:
        if event.get("event") == "cross-referenced":
            pr = event.get("source", {}).get("issue", {})
            if pr.get("pull_request", {}).get("merged_at"):
                return True
    return False
```

3. Prefer `state_reason == "completed"` / `closed_at` checks, and add a maximum issue age,
   so an issue that stays open for months cannot be re-published indefinitely.
