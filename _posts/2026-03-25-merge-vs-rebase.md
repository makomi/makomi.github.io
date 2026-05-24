---
title: "Merge vs. Rebase: A Technical Examination"
permalink: /posts/merge-vs-rebase
excerpt: The merge vs. rebase debate generates more heat than light. This article examines each argument on its technical merits, separates consensus from derivation, and draws conclusions that hold under stated preconditions rather than assumed ones.
categories: [version control, git, workflows]
tags: [git, merge, rebase, branching strategy, team workflow]
toc: true
toc_sticky: true
#last_modified_at: "2026-03-25"
---

The merge vs. rebase debate generates more heat than light. Most guidance on the topic is written for the general case — heterogeneous teams with variable discipline — and presents merge as the safe default. That framing is worth examining critically, because the "general case" may not be your case, and the arguments advanced for merge are not all of equal weight.

# The Question and Why It Matters

When integrating a feature branch into a shared branch, Git offers two fundamentally different strategies. **Merge** creates a new commit that joins two branch tips, preserving both histories. **Rebase** replays the commits of one branch on top of another, producing a linear sequence of commits with new SHAs.

The choice matters because it affects three things that are hard to reverse: the shape of the commit graph, the stability of commit SHAs, and how conflicts are surfaced. It also affects the operational cost of the workflow imposed on developers.

# Mechanics

## Merge

```bash
git checkout master
git merge feature/x
```

```
Before:
      A - B - C          (master)
               \
                D - E    (feature/x)

After:
      A - B - C - M      (master, M = merge commit)
               \   /
                D - E
```

The SHAs of `D` and `E` are unchanged. `M` has two parents and records the integration point.

## Rebase

```bash
git checkout feature/x
git rebase master
# then fast-forward:
git checkout master
git merge feature/x
```

```
Before:
      A - B - C            (master)
               \
                D - E      (feature/x)

After rebase:
      A - B - C - D' - E'  (feature/x replayed; new SHAs)

After fast-forward merge into master:
      A - B - C - D' - E'  (master, linear)
```

`D'` and `E'` are new commit objects. They carry the same content and author date as `D` and `E`, but have new SHAs and new committer dates.

# Arguments Examined

## History Shape and "True" History

**The claim:** Merge preserves the true history of development. Rebase rewrites history to make commits appear as though they were written sequentially on top of master, which falsifies the record.

**The counter:** The claim rests on the premise that the original commit sequence carries inherent value. It does not. Code quality is determined by correctness, test coverage, and the level of verification and validation achieved — none of which are properties of graph topology.

Git's interactive rebase (`git rebase -i`) is explicitly designed to let developers restructure, squash, split, and reorder commits before integration. This is not a misuse of the tool; it is a first-class feature for producing a coherent, reviewable history. The pre-rebase sequence represents intermediate work state, not a record worth preserving for its own sake.

Understanding design rationale is a legitimate requirement. The appropriate artifact for that is a design document or Architecture Decision Record — not a commit graph.

**Verdict: Argument invalid.** Graph topology has no intrinsic value. The "true history" framing is rhetorical, not technical.

## Auditability and Cross-System References

**The claim:** Rebase changes SHAs, which breaks references to the original commits in CI logs, issue trackers, deployment records, and PR comments.

**The counter and qualification:** This argument has real substance, but only from the point at which a commit has left the local repository. A rebase performed entirely locally before first push breaks nothing — no external system has yet referenced the original SHAs.

The relevant rule is therefore: do not rewrite commits that have been published to a shared remote. This is a workflow rule, not an inherent property of rebase. Enforcing it eliminates the cross-system reference problem entirely for personal and pre-review branches.

For regulated environments — where CI results, deployment records, and review artifacts must be traceable to specific commit SHAs — SHA stability on shared branches is a hard requirement. In that context, a force-push to a shared branch after rebase is not merely a best-practice violation; it is a process failure with evidence consequences.

**Verdict: Argument partially valid.** SHA stability is the strongest objective argument for merge as the integration strategy into a shared branch. The argument is correctly scoped only when applied to published commits; it has no force for local-only branches.

## Timestamps

**The claim:** Rebase loses original commit timestamps.

**The counter:** Git maintains two timestamps per commit object:

| Field | Meaning | After Rebase |
|---|---|---|
| Author date | When the commit content was written | Preserved |
| Committer date | When the commit object was created or applied | Updated to rebase time |

`git log` displays author date by default. The timestamp a developer sees after rebasing is the original authorship timestamp. The committer date changes silently, which is relevant if tooling depends on it, but the claim as commonly stated is imprecise.

**Verdict: Argument invalid as stated.** The author date is preserved. The imprecision undermines it as a general argument.

## Conflict Resolution Burden

**The claim for merge:** Conflicts are resolved once, at the merge commit.

**The claim for rebase:** Conflicts are resolved once per replayed commit, which can mean many resolution events.

The full picture:

| Factor | Merge | Rebase |
|---|---|---|
| Number of resolution events | One | One per conflicting commit |
| Complexity per event | Higher — all divergence accumulated | Lower — each conflict is isolated |
| Risk of inconsistency | One large, hard-to-review resolution | Same logical conflict possibly resolved differently across commits |

A feature branch with twenty commits each touching the same file produces twenty rebase conflict events. A branch that diverged for two weeks may produce a single merge conflict that is too large to reason about clearly. The better choice depends on branch topology, not on a universal rule.

**Verdict: Context-dependent.** Neither strategy minimizes conflict resolution risk in general. The argument for merge is not uniformly valid; it depends on the number of commits on the feature branch and the degree of divergence from the base.

## Shared Branch Safety

**The claim:** Rebase on a shared branch, followed by a force-push, causes duplicate commits when a collaborator later pulls. This makes rebase inherently risky in team settings.

The scenario typically described proceeds as follows:

```
Remote feature/x after Dev 1 rebases and force-pushes:
  A - B - C - F - D' - E'

Dev 2's local branch, not yet updated:
  A - B - C - D - E

Dev 2 runs `git pull` (merge pull, the default):
  A - B - C - F - D' - E'
                 \
                  D - E - M   <- D and E re-introduced; duplicates
```

**The counter:** This scenario requires two simultaneous workflow violations:

1. Dev 1 rebased a *shared* branch and force-pushed — violating the rule that published commits must not be rewritten.
2. Dev 2 responded with a merge pull (`git pull` without `--rebase`) rather than a rebase pull — applying the wrong integration strategy for a rebase-based workflow.

The duplicate-commit problem is not an inherent property of rebase. It is the outcome of an undefined or unenforced workflow. A team that defines and enforces a consistent strategy — rebase-only integration, `git pull --rebase`, no force-push after handoff — does not encounter this scenario.

The appropriate response to workflow failure risk is workflow definition and enforcement, not avoidance of the tool. Merge is more forgiving of undisciplined practice because it does not rewrite history. That tolerance is only a meaningful advantage for teams where workflow discipline cannot be reliably maintained.

**Verdict: Argument invalid as an inherent risk.** The failure mode requires two simultaneous violations of defined workflow. It is a workflow problem, not a rebase problem.

## CI Integration

**The claim:** Rebase may trigger CI on each replayed commit, increasing build load.

**The counter:** CI trigger behavior is a function of CI configuration, not of the Git strategy. Pipelines can be configured to run only on branch tips, on PR targets, or on push to specific branches. This is an infrastructure configuration choice, independent of the merge strategy.

**Verdict: Argument invalid.** CI behavior is configurable independently of the integration strategy.

# Argument Recap

| Argument for Merge | Status | Condition |
|---|---|---|
| Preserves "true" history | ❌ Invalid | Graph topology has no objective value |
| SHA stability / auditability | ⚠️ Scoped | Valid only for published commits |
| Timestamps preserved | ❌ Invalid | Author date is preserved; claim applies only to committer date |
| Single conflict resolution event | ⚠️ Context-dependent | Advantage reverses when the feature branch has many conflicting commits |
| Safe for shared branches | ⚠️ Operational only | Merge tolerates workflow violations; not inherently safer under discipline |
| CI-friendly | ❌ Invalid | CI trigger behavior is configurable; not a property of the Git strategy |

# Conclusion

When the arguments are stripped of rhetorical framing and evaluated on objective criteria, a single valid criterion separates merge from rebase as the integration strategy for shared branches:

> Once a commit has been published to a shared remote, its SHA must not change. Rewriting published history breaks referential integrity across CI systems, issue trackers, deployment records, and review artifacts.

This is the only robust, objective argument for using merge to integrate into shared branches such as `master`. All other commonly cited arguments either do not hold under scrutiny, are context-dependent, or are reducible to workflow discipline problems rather than inherent properties of the tool.

The practical workflow that follows from this analysis:

| Use rebase for | Use merge for |
|---|---|
| Cleaning up commits before first push | Integrating an approved PR into a shared branch |
| Updating a local feature branch with latest master | Any integration where source commits are already published and externally referenced |
| Preparing a coherent commit sequence before opening a PR | Environments where workflow discipline cannot be reliably enforced |

The boundary condition is **publication**, not branch type. The question is not "is this a feature branch?" but "have these commits been referenced by any external system?" Before that point, rebase is the more capable tool. After it, the SHAs must be treated as stable.

Teams should encode this boundary explicitly in their workflow definition, enforce the merge strategy at the repository level via branch protection settings, and configure `git pull --rebase` as the default pull strategy for developer workstations to prevent accidental merge commits during daily sync operations.

**Rebase locally. Merge into shared branches. The distinction is SHA stability after publication — not aesthetic preference.**
