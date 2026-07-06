---
name: find-unresolved-issues
description: Find genuinely unresolved GitHub issues — open, unassigned, no linked PR, no fix-in-progress. Filters out issues that appear open but are effectively resolved.
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
3. **No fix-in-progress** — no comment saying "working on this" or "PR coming"
4. **Not superseded** — another issue or PR didn't make this one redundant
5. **No merged fix** — no merged PR that references this issue number in its body ("fixes #NNN" / "closes #NNN")

Common traps to avoid:
- Issue #2674 (HugeGraph) looked open but PR #2982 had already implemented it
- Issues with "duplicate of #XXX" in comments — actually resolved
- Issues where the last comment is "fixed in commit abc123" — check the commit

## Method 1: GitHub Search API (fast, programmatic)

Use the GitHub Search API to find candidates, then filter:

```
gh search issues --repo <owner/repo> \
  --state open \
  --label "good first issue,help wanted" \
  --limit 30 \
  --no-assignee \
  --json number,title,labels,assignees,comments,createdAt,updatedAt
```

Key flags:
- `--no-assignee` — skip issues someone already claimed
- `--label "good first issue"` — beginner-friendly
- Combine labels with commas for OR: `--label "good first issue,help wanted"`

Then for each candidate, check if it's really unresolved:

### Step 1: Check for linked PRs
```
gh api "repos/<owner>/<repo>/issues/<number>/timeline" \
  --jq '.[] | select(.event == "cross-referenced") | .source.issue.pull_request.url'
```
If ANY cross-referenced event points to a PR → issue may already be handled.

### Step 2: Check merged PRs that reference this issue  
```
gh search prs --repo <owner>/<repo> \
  --state merged \
  "<issue number> in:body" \
  --json number,title,state,mergedAt
```
This catches PRs that say "Fixes #NNN" or "Closes #NNN" in their body. Crucial — GitHub auto-closes doesn't always work, so the issue stays open.

### Step 3: Read recent comments
```
gh issue view <number> --repo <owner>/<repo> --comments --json comments
```
Look for:
- "duplicate of" / "dup of"
- "fixed in" / "resolved by"
- "working on this" / "I'll take this"
- "PR: #NNNN" / "see #NNNN"
- "不再维护" / "won't fix" / "这个已经在版本 X 修了"

### Step 4: Check for commits referencing the issue
```
gh search commits --repo <owner>/<repo> \
  "#<number>" \
  --json sha,commit
```
Some repos merge fixes without a PR or the PR body doesn't reference the issue.

## Method 2: Browser-based (for repos without gh CLI)

1. Navigate to `https://github.com/<owner>/<repo>/issues?q=is:open+is:issue+no:assignee+label:"good+first+issue"`
2. For each candidate issue, check:
   - Scroll to bottom — any "linked pull request" in the sidebar?
   - Read last 3 comments — anyone claimed it?
   - Search the repo for PRs mentioning this issue number
3. Use `browser_console` to run JS that scrapes issue metadata

## Method 3: Bulk Python Script

```python
import subprocess, json

def find_truly_open(repo, labels="good first issue", limit=20):
    """Find issues that are genuinely unresolved."""
    # 1. Get open, unassigned issues
    result = subprocess.run([
        "gh", "search", "issues", "--repo", repo,
        "--state", "open", "--no-assignee",
        "--label", labels, "--limit", str(limit),
        "--json", "number,title,labels,createdAt,updatedAt"
    ], capture_output=True, text=True)
    issues = json.loads(result.stdout)
    
    truly_open = []
    for issue in issues:
        num = issue["number"]
        print(f"Checking #{num}: {issue['title'][:60]}...")
        
        # 2. Check for cross-referenced PRs (timeline)
        tl = subprocess.run([
            "gh", "api",
            f"repos/{repo}/issues/{num}/timeline",
            "--jq", '[.[] | select(.event=="cross-referenced") | select(.source.issue.pull_request != null)] | length'
        ], capture_output=True, text=True)
        pr_refs = int(tl.stdout.strip() or 0)
        
        # 3. Check merged PRs mentioning this issue
        prs = subprocess.run([
            "gh", "search", "prs", "--repo", repo,
            "--state", "merged", f"#{num} in:body",
            "--json", "number", "--limit", "1"
        ], capture_output=True, text=True)
        merged_prs = len(json.loads(prs.stdout))
        
        if pr_refs > 0 or merged_prs > 0:
            print(f"  SKIP #{num}: has linked PR or merged fix")
            continue
        
        # 4. Check recent comments for claim/fix signals
        comments = subprocess.run([
            "gh", "issue", "view", str(num), "--repo", repo,
            "--comments", "--json", "comments"
        ], capture_output=True, text=True)
        comments_data = json.loads(comments.stdout)
        last_comments = [c["body"] for c in comments_data.get("comments", [])[-3:]]
        
        skip_phrases = [
            "duplicate of", "dup of", "fixed in", "resolved by",
            "working on this", "I'll take this", "PR: #", "see #",
            "不再维护", "won't fix", "已经在版本", "close?",
            "addressed in", "solved by", "已修复", "已解决"
        ]
        if any(any(phrase in c.lower() for phrase in skip_phrases) for c in last_comments):
            print(f"  SKIP #{num}: comments indicate it's resolved or claimed")
            continue
        
        truly_open.append(issue)
        print(f"  ✓ #{num} is GENUINELY OPEN")
    
    return truly_open
```

## Output Format

Present results as a table:

| # | Title | Labels | Age | Why It's Safe |
|---|-------|--------|-----|---------------|
| 2345 | Fix Dockerfile entrypoint | good first issue | 3mo | No PR refs, no claim, no merged fix |

Include a "Why It's Safe" column — this is the evidence you checked. Never present an issue as "good to work on" without this evidence.

## Red Flags (auto-skip)

When scanning an issue, skip immediately if you see ANY of:

In the issue body or comments:
- `Duplicate of #`
- `dup of #`
- `Fixed in`
- `Resolved by #`
- `PR #` (as a fix reference)
- `已经在版本 X 修了` / `这个在版本 X 已解决`
- `Won't fix` / `Working as intended`
- `I'm working on this` / `I'll submit a PR`

In the timeline/sidebar:
- Linked pull requests in the sidebar
- Cross-referenced PR events
- A commit that says "fixes #NNN"

In the repo:
- A merged PR whose body contains `#<issue_number>` — even if the issue is still open!
- A commit whose message contains `#<issue_number>` on main/master branch

## For Chinese/Self-Hosted Repos

Some repos use Gitee, GitLab, or self-hosted Gitea. The same logic applies but:
- Gitee API: `https://gitee.com/api/v5/repos/<owner>/<repo>/issues`
- GitLab API: `https://gitlab.com/api/v4/projects/<id>/issues`
- Always check if the repo has a "已解决"/"已修复" label that the maintainer forgot to apply

## After Finding Issues

1. Present the filtered list to the user
2. Let them pick, or recommend based on:
   - Age (older = less likely someone is secretly working on it)
   - Labels (good first issue > help wanted > no label)
   - Comments count (0-2 comments = less competition)
3. Before starting work, comment on the issue: "I'd like to work on this" — give maintainers 24h to respond before submitting a PR
