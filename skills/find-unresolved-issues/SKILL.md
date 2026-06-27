---
name: find-unresolved-issues
description: Find genuinely unresolved GitHub issues — open, unassigned, no linked PR, no fix-in-progress. Filters out issues that appear open but are effectively resolved. Excludes bot-only engagement.
triggers:
  - "找 issue"
  - "找未解决的 issue"
  - "find unresolved issues"
  - "找能做的 issue"
  - "good first issue"
  - "help wanted"
  - "找能贡献的 issue"
---

# Find Unresolved GitHub Issues

Find GitHub issues that are **genuinely unresolved** — excluding those that appear open but are already fixed, have an active PR, or are effectively closed.

## What We're Looking For

An issue is "truly unresolved" when ALL of these hold:

1. **Open** — not closed
2. **No linked PR** — no PR that closes/fixes it
3. **No fix-in-progress** — no human comment saying "working on this" or "PR coming" (bot comments don't count)
4. **Not superseded** — another issue or PR didn't make this one redundant
5. **No merged fix** — no merged PR that references this issue number in its body ("fixes #NNN" / "closes #NNN")

Common traps:
- Issue #2674 (HugeGraph) looked open but PR #2982 had already implemented it
- Issues whose only comment is from a bot (Dosu, dependabot) — looks engaged but is untouched
- "I'll follow up on this after #XXXX" — soft claim, someone is waiting to work on it
- Maintainer asks "is this by design?" — the issue may not be a real bug

## The Pipeline (4-Layer Filter)

Always follow this order to minimize API calls:

```
  Layer 0: Search broadly → get candidates
  Layer 1: Filter in Python (labels, comment count, bot-only detection)  
  Layer 2: Cross-ref check (timeline API) + merged PR search
  Layer 3: Read issue body + human comments for claim/fix signals
```

Layer 3 is the most expensive (one `gh issue view` per candidate). Run Layer 2 first to eliminate issues before hitting Layer 3.

## Layer 0: Search for Candidates

```
gh search issues --repo <owner/repo> \
  --state open is:issue \
  --limit 40 \
  --json number,title,labels,commentsCount,createdAt,updatedAt
```

Key points:
- **Always include `is:issue`** — without it, PRs leak into results
- **The JSON field is `commentsCount`** (NOT `comments`) — using `comments` causes "Unknown JSON field" error
- **DO NOT use `--sort`** — `--sort comments --order asc` returns 0 results with `is:issue`. Filter programmatically instead
- **Label negation is unreliable** — `-label:question` may return empty. Filter labels in Python
- **`--no-assignee` is optional** — use it for repos with many stale assignees, but skip it if you want to catch issues where the assignee has abandoned them (check last activity date)
- **`--limit 40`** is a good default — enough breadth, won't blow through API rate limits in Layer 2/3

## Layer 1: Programmatic Filter

```python
import json, subprocess

# Run the search
result = subprocess.run([...], capture_output=True, text=True)
issues = json.loads(result.stdout)

# Filter candidates
candidates = []
for issue in issues:
    labels = [l["name"] for l in issue.get("labels", [])]
    cc = issue.get("commentsCount", 0)
    
    # Skip: questions, inactive, meta/summary/track/epic
    if "question" in labels or "inactive" in labels:
        continue
    title_lower = issue["title"].lower()
    if any(kw in title_lower for kw in ["summary", "track", "epic", "roadmap"]):
        continue
    
    # Skip: graduation/trademark (not code)
    if "graduation" in labels:
        continue
    
    candidates.append(issue)
```

## Layer 2: Cross-Referenced PRs + Merged PR Search

For each candidate from Layer 1, run these BEFORE reading issue body/comments:

```python
# Check 1: Cross-referenced PRs via timeline
tl = subprocess.run([
    "gh", "api", f"repos/{repo}/issues/{num}/timeline",
    "--jq", '[.[] | select(.event=="cross-referenced") | select(.source.issue.pull_request != null)] | length'
], capture_output=True, text=True, timeout=12)

# Check 2: Merged PRs that mention this issue
prs = subprocess.run([
    "gh", "search", "prs", "--repo", repo,
    "--state", "merged", f"#{num} in:body",
    "--json", "number", "--limit", "1"
], capture_output=True, text=True, timeout=12)
```

If either check returns results → SKIP. This is the fastest elimination step.

## Layer 3: Issue Body + Human Comments

```python
view = subprocess.run([
    "gh", "issue", "view", str(num), "--repo", repo,
    "--comments", "--json", "body,comments"
], capture_output=True, text=True, timeout=12)

data = json.loads(view.stdout)
```

### CRITICAL: Filter Out Bot Comments First

Many repos have automated bots. Their comments look like engagement but are NOT human. **Always filter out bot comments before scanning for claims.**

```python
KNOWN_BOTS = {
    "dosubot", "dependabot", "github-actions", "codecov",
    "renovate", "stale", "imgbot", "allcontributors",
    "sonarcloud", "coveralls", "sizebot", "changeset-bot"
}

human_comments = [
    c for c in data.get("comments", [])
    if c["author"]["login"].lower() not in KNOWN_BOTS
]
```

An issue whose **only** comments are from bots is effectively untouched — treat it as available.

### Skip Phrases (check against issue body + human comments)

These indicate the issue is resolved, claimed, or not a real bug:

```
In English:
  "duplicate of", "dup of", "fixed in", "resolved by",
  "working on this", "i'll take this", "i'll follow up",
  "follow up on this", "i will work", "i am working",
  "i'm on it", "assign me", "let me handle", "taking this",
  "pr: #", "see #", "won't fix", "wontfix",
  "addressed in", "solved by", "close?",
  "no longer relevant", "already in", "not a bug",
  "is this by design", "is this intended"

In Chinese / mixed:
  "已经在版本", "已修复", "已解决", "不再维护",
  "这个在版本 X 修了", "已有 pr", "已经被修复",
  "设计如此", "不是 bug", "预期行为"
```

### Red Flag: Maintainer "Design Philosophy" Questions

If a maintainer comments asking whether behavior is "by design" or "is this intended" — the issue is likely NOT a real bug. Flag these for the user but don't auto-skip (sometimes the maintainer is wrong).

## Output Format

Present results as a ranked table with Why It's Safe column:

| # | Title | Labels | Age | Why It's Safe |
|---|-------|--------|-----|---------------|
| 2756 | Port 8080 false positive | bug | 14mo | No PR refs, no cross-refs, only bot comment, no human claims |

Sort by recommendation: smallest scope + clearest reproduction steps first. Mark top picks with ★.

## After Finding Issues

1. Present the filtered list to the user with Why It's Safe evidence
2. Let them pick, or recommend based on:
   - **Scope clarity** — is the fix well-defined or vague?
   - **Reproduction steps** — can you reproduce without a complex cluster setup?
   - **Labels** — good first issue > help wanted > bug > feature > empty
   - **Age** — older = less likely someone is secretly working on it
   - **Human comments** — 0-2 human comments = less competition
3. Before starting work, comment on the issue: "I'd like to work on this" — give maintainers 24h to respond

## Rate Limiting

The timeline API check (Layer 2) is the bottleneck. For GitHub.com:
- Authenticated: 5,000 requests/hour
- Each candidate costs ~3 API calls (timeline + search prs + issue view)
- With 40 candidates from Layer 0 and ~50% filtered by Layer 2, ~15 reach Layer 3
- Total: ~40 (search) + ~40 (timeline) + ~40 (search prs) + ~15 (issue view) ≈ 135 calls
- Well within limits; but for repos with 100+ open issues, batch in groups of 40

## Real-World Example (HugeGraph)

A session found 40 open non-question issues. After the 4-layer pipeline:
- Layer 1 filtered to ~15 candidates (excluded questions, meta, graduation)
- Layer 2 eliminated 5 (cross-referenced PRs) + 0 (merged PR mentions)
- Layer 3 eliminated 2 more (soft claims like "I'll follow up after #2994")

Final 3 genuinely unresolved:
- #2756 — Port 8080 false positive (★ best scope)
- #2745 — DEFAULT_CAPACITY configurable (★ clearest requirements)
- #2801 — PostgreSQL connection leak (★ highest practical impact)
