# ReviewRoulette

A weighted random picker for choosing a code reviewer when nobody volunteers.

## The problem

Peer review works best when someone actually reads the code. In practice, review requests often sit unclaimed until someone feels guilty enough to pick it up, and it's usually the same one or two people every time based on the area they are most comfortable. A plain random picker solves the "nobody volunteers" problem but creates a new one: a brand-new team member and a principal engineer shouldn't have equal odds of being handed a complex, high-risk change.

ReviewRoulette is a **weighted** picker where a reviewer's odds are based on their experience on the codebase, how many reviews they've already done recently and the complexity/impact of the specific task at hand, not just a flat dice roll.

## Who this is for

- Teams that want knowledge of the codebase spread more evenly, rather than concentrated in whoever always ends up reviewing everything
- Teams where several people have similar skill levels, so there's no obvious "most senior person" to default to, and volunteering just doesn't happen
- Teams that want a bit of fun in an otherwise mundane part of the process. The occasional spinning wheel is a small thing, but it makes review assignment something people notice and laugh about rather than pure dread.

## Status: planning

As always, I like to plan first, therefore, nothing is built yet. This repo currently holds the problem statement and the roadmap below. Code starts with v0.

## Design approach: platforms as pluggable targets

The plan is to build this around a **target** which is a contract that defines what any version-control platform integration has to implement (fetch PR/MR details, list eligible reviewers, post a comment, etc.), rather than writing GitHub-specific code straight through. GitHub is the first target, since it's the platform most teams use, even though GitLab is what I actually work with day to day. The contract is what makes GitLab, Bitbucket, or anything else a matter of writing a new target, not rewriting the engine. The weighting logic itself never needs to know which platform it's running against.

Thanks to the Refactoring Guru https://refactoring.guru/design-patterns/factory-method

## Roadmap

- **v0 — core engine:** the weighting logic itself, built and tested in isolation, with no platform integration yet. Inputs: a reviewer's experience, their recent review load, and the task's complexity/impact. Output: a weighted pick.
- **v0.5 — spinner UI:** a standalone visual wheel, in the spirit of [wheelofnames.com](https://www.wheel0fnames.com/), that takes a predefined list of names and weights and spins. No platform data, no backend yet. Just proves the weighting engine produces something satisfying to actually watch, before wiring it into anything live.
- **v1 — target contract + GitHub target:** define the target interface, then implement it for GitHub, webhook on PR open, pick a reviewer using the v0 engine, comment on the PR.
- **v2 — reliability layer:** GitHub webhooks expect a fast response (a few seconds) or they retry, sometimes repeatedly. Processing everything inline risks duplicate picks if a downstream step (posting the comment, logging, etc.) is slow or fails. The plan is to put a message queue between the webhook handler and the actual picking logic, so the handler acknowledges instantly and the real work happens separately, with retries and a dead-letter queue if something fails partway through.

## Why weighted, not just random

The fairness mechanism isn't "exclude busy people," it's "make them progressively less likely to be picked again" which is essentially a soft decay rather than a hard cutoff. And experience should matter more on a genuinely complex, high-impact change than on a one-line fix. Getting that balance right is most of the actual design work here, more than the mechanics of picking a random number.

## Open design question: how does complexity get scored?

This was the most challenging part of the consideration.

But I have three options, honestly weighed against each other:

- **Manual**: a label or field the PR author sets (e.g. `complexity:3`). Simplest to build, but adds friction: people forget, or default to the same number every time, which quietly breaks the whole weighting system.
- **Automatic**: inferred from PR metadata like lines changed or files touched. No friction, but naive on its own: a one-line change to a critical payment path scores "low complexity" by line count even though it's genuinely high-risk.
- **Hybrid**: a sensible automatic default, with a manual override available if someone disagrees with the guess.

**Current plan: start manual for v0/v1**, via a simple label convention, same "keep it simple first" approach as the rest of this roadmap.

Also being considered is automatic scoring using an LLM to read a diff and suggest a complexity/impact score and a better approach than either staying manual forever or trying to use diff changes to determine complexity which is equally naive.
