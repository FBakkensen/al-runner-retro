# Prompt: 30-day retrospective on the agent workflow (run from your side)

Copy everything below the line into a fresh Claude Code session opened in your clone of
`StefanMaron/BusinessCentral.AL.Runner`. It reproduces the retrospective the other operator ran on
2026-09-09, but over *your* local session transcripts, so the two reports can be laid side by side.
The GitHub half of the data is identical for both operators; only the transcript half differs.

---

You are running a retrospective on the multi-agent workflow around
`StefanMaron/BusinessCentral.AL.Runner` and `StefanMaron/BusinessCentral.AL.Language.Tests`.
Two operators run independent agent loops that communicate only through GitHub issues and PRs. My
login is `<YOUR_GITHUB_LOGIN>`; my agent pools are `<e.g. impl-N, stma-auto-N, orchestrator, coord>`.
The other operator is `FBakkensen` with pool `fbk-N`. The `claude` / `claude[bot]` logins are mine.

## Scope, already decided (do not re-ask)

- **Window:** 2026-08-10 to 2026-09-09 (matching the first report so the numbers are comparable).
- **Audience:** both operators, public-safe. No employer names, hostnames, local paths, email
  addresses, or quoted user messages anywhere in the output. Refer to people by GitHub login or role.
- **Dimensions, in priority order:** (e) quality of what merged, (a) throughput and outcome,
  (b) coordination over the GitHub bus, (c) rule effectiveness, (d) agent cost and behaviour.
- **Output:** one self-contained single-page HTML report, written outside the repo. Nothing filed,
  labelled, commented, edited, or pushed on GitHub or in the repo. This is a read-only audit.
- **Deliver, don't ask.** Make routine judgment calls yourself. Stop only if `gh` cannot reach
  github.com.

## Step 0: preflight

```bash
export GH_HOST=github.com
gh api user --jq .login            # must print your login
gh api rate_limit --jq '.resources.core.remaining'
python3 --version                  # 3.11+
mkdir -p ~/tmp/al-runner-retro/{raw,derived,private} && cd ~/tmp/al-runner-retro
```

Prefix every `gh` call with `GH_HOST=github.com` if your default host differs. Filter the `mise`
banner out of any `$(...)` capture. Never use bare `grep -E` in this repo (shell-function trap);
use `command grep` or python.

## Step 1: pull one read-only snapshot to disk, then analyse only the snapshot

```bash
R=StefanMaron/BusinessCentral.AL.Runner; C=StefanMaron/BusinessCentral.AL.Language.Tests
S=2026-08-10T00:00:00Z
gh api --paginate "repos/$R/issues?state=all&since=$S&per_page=100" --jq '.[]' > raw/runner_issues.jsonl
gh api --paginate "repos/$R/issues/comments?since=$S&per_page=100"  --jq '.[]' > raw/runner_comments.jsonl
gh api --paginate "repos/$C/issues?state=all&since=$S&per_page=100" --jq '.[]' > raw/corpus_issues.jsonl
gh api --paginate "repos/$C/issues/comments?since=$S&per_page=100"  --jq '.[]' > raw/corpus_comments.jsonl
# repo-wide issue events, newest first, page until created_at < 2026-08-10 (~150 pages)
# PRs via GraphQL, ordered UPDATED_AT desc, stop when updatedAt < window: fields
#   number title state isDraft createdAt updatedAt closedAt mergedAt author.login headRefName
#   baseRefName additions deletions changedFiles mergeCommit.oid mergedBy.login bodyText
#   labels(30) closingIssuesReferences(20){number state} files(100){path additions deletions}
#   reviews(30){author state submittedAt} comments.totalCount commits.totalCount
#   timelineItems(100, LABELED/UNLABELED/CLOSED/REOPENED/MERGED/HEAD_REF_FORCE_PUSHED)
# for both repos -> raw/runner_prs.jsonl, raw/corpus_prs.jsonl
git fetch origin main --quiet
git log origin/main --since=2026-08-10 --format='%H%x09%ad%x09%an%x09%s' --date=iso-strict > raw/gitlog_main.tsv
git log origin/main --since=2026-08-10 --name-only --format='%x00%H' > raw/gitlog_files.txt
git log origin/main --since=2026-08-10 --numstat --format='%x00%H%x09%ad%x09%s' --date=iso-strict -- .claude CLAUDE.md docs > raw/governance_log.txt
# per week: bytes of .claude/rules/*.md + CLAUDE.md (auto-loaded every turn) and of agents+skills+commands
for w in 2026-08-10 2026-08-17 2026-08-24 2026-08-31 2026-09-07 2026-09-09; do
  c=$(git rev-list -1 --before="${w}T00:00:00" origin/main)
  printf '%s\t%s\t%s\t%s\n' "$w" "${c:0:8}" \
    "$(git ls-tree -r -l $c -- .claude/rules CLAUDE.md | awk '{s+=$4} END{print s+0}')" \
    "$(git ls-tree -r -l $c -- .claude/agents .claude/skills .claude/commands | awk '{s+=$4} END{print s+0}')"
done > raw/governance_size.tsv
git ls-remote --heads origin 'refs/heads/agent/*' | awk '{print $2}' > raw/agent_branches.txt
```

Expected sizes for the same window on 2026-09-09: 2,423 runner issues+PRs, 1,458 comments,
14,800 events, 829 runner PRs, 271 corpus PRs, 967 commits. If yours differ by more than a few
percent, the window or the pagination is wrong; fix before analysing.

## Step 2: extract structured facts from YOUR session transcripts (one python script)

Transcripts are JSONL under `~/.claude/projects/<encoded-repo-path>/*.jsonl`; each session has a
`<session-id>/subagents/*.jsonl` directory with a `.meta.json` per subagent (`agentType`,
`description`). Record types that matter: `assistant` (message.usage tokens, message.model,
content[].tool_use with `name` and `input`), `user` (a tool result if `toolUseResult` is present,
otherwise a human-typed or slash-command message), `system` with `subtype: turn_duration`,
`cost-state` (totalCostUSD, modelUsage; present in only some sessions and a point-in-time snapshot),
`pr-link`, `ai-title`.

Emit `derived/sessions.json` and `derived/subagents.json` with, per transcript: title, first/last
timestamp, harness `version`, assistant message count, tool calls by name, Bash/PowerShell commands
classified (read/search = grep|rg|sed|cat|head|tail|find|ls|wc as primary verb; gh; git; dotnet;
al-runner; python; tools/; cd-only; other), `Agent` spawns (subagent_type, description, prompt
length), typed human messages as **counts and lengths only**, turn durations, usage token sums,
cost-state, pr-links, tool-result errors. Write the human message texts to
`private/human_messages.jsonl` for classification only; that file is never shared and never quoted.

Also run the repo's own `tools/agent-cost.py <tasks-dir>` on at least one pre-2026-09-02 and one
post-2026-09-07 task directory, so the code-navigation guidance gets a before/after with the same
classifier CLAUDE.md quotes.

## Step 3: five analyses, run as parallel subagents against the snapshot

Give each subagent the file inventory, the operator mapping, and this output contract, and have it
write `derived/findings_<letter>.json`:

```json
{"dimension": "<letter> <name>",
 "method": "2-5 sentences: what was measured, how, and what could NOT be measured",
 "metrics": [{"name": "...", "value": "...", "unit": "...", "note": "..."}],
 "findings": [{"id": "<letter>-N", "title": "<=12 words", "verdict": "works|mixed|broken",
   "claim": "one public-safe paragraph", "evidence": ["#1234", "PR #2345", "corpus PR #12", "sha abcdef12", "count: 12 of 40"],
   "confidence": "high|medium|low", "why_confidence": "one sentence",
   "recommendation": "concrete change to .claude/ governance, tools, or process; or 'none'"}],
 "tables": [{"title": "...", "columns": [...], "rows": [[...]]}]}
```

Rules for every subagent: every finding carries at least one issue/PR/SHA or a count with a
denominator; a zero from a pattern the agent chose is not a finding until a second, differently
shaped query confirms it; duplicate-`Closes` detection uses `closingIssuesReferences` **and** the
issue number in `headRefName` (34 of 48 duplicate pairs were only visible through the branch name);
python over the snapshot, batched, not twenty greps; foreground only; read-only.

**(e) Quality of what merged.** Sample ~30 merged PRs stratified by pool and week; `git show` the
squash merge commit; classify the proving test as proves / weak / none (would it pass if the fix
returned a default?). Population: code-touching merged PRs that touch a test path. Corpus linkage:
re-derive AL-observable from the gate's own path list (`AlRunner/Patches/`, `Rewriters/`,
`Infrastructure/NclCecilRewrite*`, `BcCompiler*`, `BcAssembler.cs`); resolve every `Corpus-PR:` to
`raw/corpus_prs.jsonl` and report runner PRs merged **before** the corpus PR merged and runner PRs
whose corpus PR **closed unmerged**; read every `Corpus-NA:` reason. Hot-file churn: consecutive
merges on one file within 48 h / 6 h; strict follow-up/regression references vs loose keyword hits;
real reverts (verify every candidate). Expectations manifest: known-gap entries added vs removed
over time; PRs adding an entry while changing patch code. `main` health: floor-run failures,
cancelled-superseded matrix runs, "main is red" issues. Review effectiveness: PRs with a review
comment before merge, by date; pushes after the first review comment; header styles used.

**(a) Throughput.** Opened/merged/closed-unmerged per week by pool (branch prefix cross-checked
with author login); cycle time median/p90; issues opened/closed per week and the open backlog at
each week end; issues closed by a merged PR vs otherwise; merged PRs with no closing reference and
whether they carry the `No linked issue:` hatch; multi-close PRs; force-push share; formal GitHub
review objects; corpus PR throughput and who merges; merge hour-of-day histogram; issue author →
PR author matrix (who fixes whose filings).

**(b) Coordination.** Issues targeted by 2+ PRs and the subset that were genuinely concurrent
(both open at once); gap between starts; same- vs cross-operator. Claim protocol: in-progress label
→ PR within 24 h; issues relabelled to a different agent; unlabel events by a different login than
applied; open in-progress issues and their age; closed issues still carrying `status:`/`agent:`
labels; contradictory labels; ready-queue age distribution and who filed it. Cross-operator comment
threads and reply latency. Merged PRs whose branch names an issue they never declared (and whether
that issue is still open). Distinct `agent:` labels and max concurrent open PRs under one label.
Agent branches with no PR. Closed-unmerged PRs and their stated reasons. Self-identification in
agent comments before/after `public-posting-approval.md`'s rule (sha abdac823). Corpus side: merge
latency, `run-nightly-windows` usage, cross-operator comments.

**(c) Rule effectiveness.** For each file in `.claude/rules`, `.claude/agents`, `.claude/skills`,
`CLAUDE.md`: birth date, incidents cited (resolve issue numbers to dates), recurrences after birth
(the files admit some themselves), what finally stopped the class (prose / tool / required CI gate /
nothing yet). Count each incident class before vs after the rule in issues, comments, PR bodies,
commit subjects and structural queries (reopen events after a merge-close; ci-skip strings; CHANGELOG
touches; simultaneously-open PRs closing one issue; issues titled against ci-wait.py / preflight.py /
pr-body.py by date). Governance growth per week; share of auto-loaded bytes in incident-narrative
paragraphs (date, `#NNNN`, "Measured", incident verb); number of files stating each of 20 sampled
instructions; direction flips inside the window with dates and whether the superseded text is still
on `main`; governance commits whose subject corrects earlier guidance.

**(d) Cost and behaviour (your transcripts).** Cost-state coverage and a usage-based estimate;
share by model tier, by subagent type, cache-read share; rough cost per merged PR from your pools.
Tool profile per subagent type: calls median/p90, over-200 count, grep-family share (primary verb
and contains-anywhere), navigation-tool commands (`context-pack`, `lsp-query`, `graphify`; also
`Skill find-code` and `LSP` tool calls), bare-`cd` share by harness version, CI-polling share
before/after the never-block-on-CI change (impl-agent.md 2026-09-05, ci-verdicts.md sha 96b15f0e
2026-09-07). Turns over 10/30/60 min and what they are. Impl agents that never ran `gh pr create`
and why. Typed human messages by week and category (new task / answer / correction / status /
approval / rule statement / handoff / other) and the correction **themes** that recur 3+ times, in
neutral wording, counts only. Spawns, brief length over time, subagent transcript size over time,
`SendMessage` to subagent vs to a peer session.

## Step 4: reconcile before rendering

- Cross-check the dimensions against each other; where two measured the same thing with different
  instruments, cite the finer one and footnote the other. Name the basis behind any remaining
  disagreement (created-in-window vs updated-in-window; branch prefix vs author login).
- Spot-check the five claims a reader will push back on hardest by re-querying `raw/` directly.
- Do not credit a rule for behaviour that predates it: check daily counts around the rule's birth.
- Run a public-safety scan on the **final HTML**, not on the JSON: a blocklist of your own private
  names/hosts/paths/emails, and every 30-character substring of `private/human_messages.jsonl`.
  Path fragments that also occur in the repo are fine; prose is not.

## Step 5: the page

Self-contained HTML in `~/.agent/diagrams/`, sections in the order e, a, b, c, d. Lead with a
summary: five "working well", five "going wrong", stat tiles, one chart with three panels on a
shared time axis (weekly merges stacked by pool; open issues at week end; auto-loaded governance KB)
with dashed markers at 08-27, 08-31, 09-05, 09-06, 09-07, 09-08. Then ranked changes across all
dimensions, each tagged with the finding IDs that back it and where it lands (`.claude/`, `tools/`,
or a maintainer-owned workflow). Then each dimension: method, metrics, finding cards filterable by
verdict, tables. Close with method and caveats (GraphQL 100-file cap, one commit node per PR,
same-login collisions invisible in events, days of post-change data for the newest rules, cost
figures are one operator's floor plus an estimate). Link `#N` to the runner repo, `corpus PR #N`
to the corpus repo, `sha` to the commit. Inline SVG for the chart, no chart library; a table view
under the chart. Validate the palette for both light and dark surfaces. Open the page in a browser
and look at it before calling it done.

## Numbers from the first run, for comparison

827 PRs opened / 796 merged / 0.8 h median; 1,107 issues opened / 872 closed; open issues 8 → 237
(08-24 → 09-09); 407 merges in the week of 08-31; 625 of 638 code PRs touch a test path; 25/2/3
proves/weak/none in a 30-PR sample; 209 of 335 pre-gate AL-observable PRs had no corpus reference,
94 of 95 post-gate declare one; 4 post-gate runner PRs merged with the corpus PR still open; 4
merged runner PRs cite corpus PRs that closed unmerged; 8 genuine PR collisions, 2 after
check-open-prs landed; 110 distinct `agent:` labels; 561 of 872 closed issues still in-progress;
31 merged PRs with an undeclared branch issue, 12 open; auto-loaded governance 17 KB → 133 KB, 35%
narrative, 4.35 files per instruction, 8 direction flips, 18 of 87 governance commits correcting
earlier guidance; other operator's subagents: 93% Bash, navigation tools in 45 of 11,418 commands,
impl-agent median 116 calls, pure CI waiting 7.2% → 1.4% after the never-block rule.

The other operator's report is a file, not a repo artifact; ask them for it and put the two side by
side. Where your numbers on the shared GitHub data differ from the ones above, that is a counting
basis difference to name, not an error to hide.
