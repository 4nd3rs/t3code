# Review desk: design

Status: approved by the developer on 2026-11-12 (second revision). Nothing in here is built yet.

This fork is public. The GitHub organization it will be used with, its repository names, and the
review rules are private and must never appear in this repository. This document says "the org"
on purpose.

## Intent

The developer reviews pull requests across one GitHub organization with a few dozen repositories.
Today that means checking GitHub repo by repo. They want, inside T3 Code:

- a list of every open PR in the org with one clear status each,
- agent reviews that start on their own for the PRs their rules select,
- a click on a PR to see its details next to the agent's live work, and
- one long-lived "master reviewer" agent that they talk to about all of it.

Everything T3 Code already does (remote environments, any provider, spawning other agents) must
keep working, because this is a fork that will keep merging upstream.

"Review desk" is a working name chosen by the designer, not by the developer.

## Goals

1. All open PRs in the configured org in one pane, grouped, with status, people, CI and comment
   counts per row.
2. Automatic agent reviews, selected by rules the developer edits inside T3 Code.
3. A draft review per PR that the developer reads, edits and posts, or that is posted on its own
   where the rules say so.
4. A master reviewer agent that explains findings, re-runs or redirects reviews, and edits the
   rules.
5. Cheap upstream merges: new files wherever possible, short edits to upstream files.

## Non-goals

- A general dashboard or plugin framework. The developer deferred that explicitly. Do not add a
  registry, an abstraction for "dashboards", or anything named for future dashboards.
- Mobile UI. Web and desktop only (designer's call).
- GitHub webhooks. Polling only (developer agreed).
- Forges other than GitHub.
- Sharing review state between several people. One developer, one GitHub account per environment.
- Merging PRs, requesting reviewers, or any PR action other than posting a review.

## Anti-goals

These would make the feature a failure even if it works:

- A review is posted to GitHub when neither the developer clicked Post for it nor the rules mark its
  repository for automatic posting.
- An approval or a change request is posted without a click, under any rule.
- The same review is posted twice, or two machines review the same PR, because both run the desk.
- The org name, a repository name, or rule text is committed to the fork.
- The fork adds an entry to the upstream migration chain (see Storage).
- Agent reviews start on every dependency-bot PR and burn tokens with no rule asking for it.
- An upstream merge becomes a rewrite because the feature is woven through core files.

## Decided by the developer

- The fork is public; org, repository exclusions and review rules live outside the fork's git
  history. They may live in a private repository in the org so they are versioned and shared
  between machines.
- The rules must be easy to reach and change from inside T3 Code.
- Scope is the PR part only.
- When to review is part of the rules: when a review is requested from me, and always for some
  named repositories.
- Whether a finished review is posted on its own is also part of the rules. Manual is the default;
  automatic is an option.
- The master reviewer is the main agent the developer talks to, including to resolve issues.
- Each row shows the author's avatar and name, who else is involved, CI status, whether GitHub
  Copilot reviewed, and how many comments the PR has.
- A person's review state is written out in a text pill. Color-coded dots on avatars were tried in
  a mockup and rejected as hard to read.
- Statuses: needs review, in review by me, in review by someone else, changes requested, approved,
  draft.
- It runs on the developer's MacBook, and must also be able to run on a remote environment such as
  their Mac mini.
- Polling GitHub is fine.
- Review state goes in its own small database.

## Decided by the designer (not confirmed; the developer should check these)

1. **Layout.** The desk is a right-hand pane beside the master reviewer's thread, not a separate
   full page (see Client). This is the largest unconfirmed call.
2. **Automatic posting is per repository and comment-only.** `auto_post` lists repositories. An
   automatic post is always a "comment" review. Approve and request changes need a click.
3. **Automatic work is switched on per machine.** A local setting, off by default, decides whether
   this environment starts reviews and posts on its own. Rules can be shared; the switch is not.
4. **"Requested from me" includes requests to a team I am on**, as GitHub's own filter does.
5. **Own PRs** are listed with their status but never reviewed automatically.
6. **Bot PRs** (dependency bots) sit in their own group and are reviewed only if a rule says so.
7. **"In review by someone else"** is inferred, because GitHub has no such state (see Status).
8. **Rules file format:** a small structured header for when to review and post, free text for what
   to look for (see Files).
9. **Polling** every 120 seconds.
10. **A review needs a checkout.** A PR is reviewed only if its repository is a T3 project in this
    environment. Otherwise the row offers "Add repo".
11. **Re-review** on new commits reuses the PR's existing review thread.

## How it works

### Files

A config folder holding two files, `repos.md` and `rules.md`. Both are editable from the pane, by
the master reviewer, or in any editor.

Where the folder is: a path in this environment's local server settings. The default is
`review-desk/` inside the server's state directory. The developer can point it at a folder in a
checkout of a private repository instead, which versions the rules and lets several machines share
them through git. The fork never contains these files. In a dev run the default folder sits under
the checkout's gitignored `.t3`; it must never be force-added.

`repos.md`:

```markdown
---
org: example-org
skip:
  - some-repo-to-hide
---

Free notes; ignored by the server.
```

`rules.md`:

```markdown
---
review:
  when_requested: true # a review is requested from me, directly or through a team
  always: [repo-a, repo-b] # every non-draft PR in these repositories
  never_authors: [dependabot]
  drafts: false
  own_prs: false
auto_post: [repo-a] # finished reviews here are posted as a comment without a click
max_concurrent: 2
---

# What to look for

Plain-language rules. A `## repo-a` heading scopes the text under it to that repository.
```

Rules for the files:

- The header is YAML front matter, parsed with the `yaml` package the server already depends on,
  and validated with an Effect Schema. Unknown keys are an error.
- `auto_post` defaults to empty: everything is manual unless a repository is listed.
- If either file is missing or invalid, the desk starts no reviews and posts nothing, and the pane
  shows the error with the line. It never falls back to defaults for an invalid file.
- The server re-reads the files at the start of every sweep. No file watcher.
- The text under the header is passed to review agents verbatim: the top-level text plus the
  section for the PR's repository.

### Per-machine switch

One local setting per environment, `automatic`, off by default. When off, the desk still lists PRs
and the developer can start a review by hand; it starts nothing and posts nothing on its own. This
is what keeps two machines that share one rules repository from both reviewing the same PR. The
pane shows the switch and which machine it applies to.

### Server

New directory `apps/server/src/reviewDesk/`. Follow the repository's rule that features are
services and that RPC handlers and MCP tools stay thin.

**Reading GitHub.** Two GraphQL searches per sweep, paginated:

- `org:<org> is:pr is:open archived:false` for the list;
- the same with `review-requested:@me`, only to learn which PRs are requested from me directly or
  through a team. Its result becomes a boolean on each snapshot, so the trigger stays a pure
  function.

Per PR the first search reads: number, title, URL, draft flag, author login and avatar, updated
time, additions, deletions, changed files, head SHA, review decision, requested reviewers (users
and teams), latest review per reviewer (login, avatar, state, commit), participants, issue comment
count, review thread count and unresolved count, and the check rollup (overall state plus counts by
conclusion). The viewer's login is read once; "me" always means that login.

The existing PR list cannot be reused: it is keyed by project and builds `repo:` qualifiers, so it
cannot see repositories that are not projects. The GitHub client and its credentials are private to
the source-control driver, so the search is added as one optional method on the shared driver
interface and implemented in the GitHub driver.

**Status.** A pure function from one PR snapshot plus the viewer login to one status. First match
wins:

| Order | Condition                                                                              | Status                   |
| ----- | -------------------------------------------------------------------------------------- | ------------------------ |
| 1     | PR is a draft                                                                          | Draft                    |
| 2     | Approved                                                                               | Approved                 |
| 3     | Changes requested                                                                      | Changes requested        |
| 4     | I submitted a review, or my review agent is queued, running or has a draft for this PR | In review by me          |
| 5     | Another human submitted a review or is individually a requested reviewer               | In review by that person |
| 6     | Otherwise                                                                              | Needs review             |

"Approved" and "Changes requested" come from GitHub's review decision when it is present. GitHub
reports none for repositories without a required-review rule; then the latest review per human
reviewer decides: any change request gives Changes requested, otherwise any approval gives
Approved.

Bots never count as a human (GraphQL `__typename` of `Bot`, which covers Copilot and dependency
bots). Copilot's review is shown on the row as a reviewer pill ("Copilot commented") but does not
change the status. A request to a team is shown as a pill but does not make the PR "in review by"
anyone.

**Triggers.** A pure function from one PR snapshot, the viewer login and the parsed rules to
"review" or "skip, with the reason". A PR is reviewed when all of these hold:

- it is not skipped by `repos.md`;
- it is not a draft, unless `drafts: true`;
- its author is not me, unless `own_prs: true`;
- its author is not in `never_authors`;
- it is requested from me and `when_requested` is true, or its repository is in `always`.

The reason is stored and shown ("rule: always review repo", "no rule matched").

**Sweep.** Registered with the existing `Scheduler` under its own name. Every 120 seconds, and on
a manual refresh from the pane: read the files, read GitHub, compute status and trigger per PR, and
publish the snapshot to subscribers. Then, only if this machine's `automatic` switch is on, queue a
review for each PR whose trigger says "review" and that has no review recorded for its current head
SHA, and start queued reviews up to `max_concurrent`.

**A review.** One agent thread per PR, launched from server code with `ThreadLaunchService`, the
way `ScheduledTaskService` does it:

- project: the T3 project whose repository matches; if none, the review is not queued and the row
  says "no checkout";
- workspace: a worktree at the PR's head, using the existing preparation for PR threads;
- the thread is linked to the PR with the existing thread-to-PR link, so existing PR views find it;
- model: the project's default model selection, the same fallback chain the `t3_thread_launch`
  handler uses;
- first message: a fixed review instruction, the rules text for this repository, and the
  instruction to finish by calling `review_draft_submit`.

When the head SHA later changes and the trigger still matches, the server sends a new message into
the same thread asking for a re-review of the new commits.

**Draft handback.** New MCP toolkit `reviewDesk`:

| Tool                  | Who                                    | What                                                                                                                    |
| --------------------- | -------------------------------------- | ----------------------------------------------------------------------------------------------------------------------- |
| `review_draft_submit` | a review thread or the master reviewer | Stores or replaces the draft for a PR: a verdict suggestion, a summary, and findings with severity, file, line and text |
| `review_desk_list`    | master reviewer                        | The current snapshot: PRs, statuses, review states                                                                      |
| `review_desk_start`   | master reviewer                        | Queue a review for a PR regardless of triggers                                                                          |
| `review_desk_discard` | master reviewer                        | Discard a draft                                                                                                         |

There is deliberately no tool that posts to GitHub. New tool text counts against the repository's
agent context budget test, and `evals/README.md` applies to it.

**Posting.** One server function maps the stored draft to the existing
`PullRequestSubmitReviewInput` and calls `PullRequestService.submitReview`, so the review is posted
as the developer's GitHub account. It has two callers:

- _Manual:_ the Post button. The confirm dialog shows exactly what will be posted and lets the
  developer choose comment, approve or request changes; the default is comment.
- _Automatic:_ when a draft becomes ready, the server posts it itself if the PR's repository is in
  `auto_post` and this machine's `automatic` switch is on. The verdict is always "comment". It is
  skipped, and the draft left for a click, if the viewer already has a review on that head SHA.

Limit to state plainly: an agent running with shell access can call the `gh` CLI. The review and
master prompts forbid posting, and no tool offers it, but that is an instruction, not a technical
barrier.

**Master reviewer.** One thread per environment, in a T3 project rooted at the config folder, so
the rules and repository files are in its working directory. When the folder is inside a private
repository checkout, the master reviewer can commit a rule change when the developer asks. It is
created the first time the developer opens the desk, with a standing first message describing its
job, the toolkit above, and the no-posting rule. Its thread id is stored. It is an ordinary thread
in every other way: it appears in the sidebar, works remotely, and can delegate to other agents.

### Storage

The feature keeps its own SQLite file, `review-desk/state.sqlite` in the server's state directory
(never in the config folder, which may be a git checkout), with its own schema-version table. It
must not add to `apps/server/src/persistence/Migrations.ts`.

Why: the upstream migrator runs only ids above the highest recorded id, and this repository wraps
it with a name check. A fork migration numbered 86 or 9001 would make the server refuse to start
with a migration error once upstream ships its next migration. Staying out of the chain also keeps
the main database readable by an unmodified upstream build of the same version.

Contents: one row per review (repository, number, head SHA, thread id, trigger reason, state, draft
JSON, how it was posted, timestamps) and one row for the master reviewer's thread id. States:
queued, running, ready, posted, discarded, failed. The PR snapshot itself is held in memory and
rebuilt on each sweep.

### Contracts and remote

- `packages/contracts/src/reviewDesk.ts`: snapshot, row, review state and draft schemas.
- RPC, kept to as few methods as the builder can manage: a snapshot subscription, refresh, read and
  write of the two files, set the config folder and the `automatic` switch, start a review, update
  and discard a draft, post a review, and ensure the master thread exists. Mutating methods are
  added to the guarded RPC scopes.
- Capability flag `reviewDesk`: every server built from this fork advertises it. It means "this
  server has the desk", not "the desk is configured". The snapshot carries `configured` and any
  file error, and the pane shows a setup state when it is not configured. Unmodified upstream
  servers do not advertise it, and the client hides the desk for them.
- The desk belongs to one environment. It runs on whichever machine the fork's server runs on: the
  MacBook, or a remote one such as a Mac mini that the client connects to in the usual way. With
  several capable environments connected, each has its own desk; nothing is merged across them.

### Client (web and desktop)

**Layout.** The desk is a new right-panel kind that opens beside the master reviewer's thread:

- Left, the normal chat view: the master reviewer thread, unchanged, with every standard thread
  feature.
- Right, the Review desk pane: tabs for Pull requests, Rules and Repos. In Pull requests, the list
  is on top and the selected PR's detail is below it.
- Opening a PR's review thread uses the existing side-chat pane, whose parent is the master thread.

Why this layout: the developer asked for a pane, and said they mostly talk to the master reviewer.
It also needs no change to `ChatView.tsx`, a large and frequently changed upstream file. A separate
full page with two embedded chats would need a new embedding mode there.

A sidebar button and a command palette entry open the master thread with the pane showing, creating
the thread on first use. They are hidden when no connected environment has the capability.

Rule for the builder: cheap upstream merges outrank the embedded live view. `SideChatPane` assumes
a side chat forked from its parent and offers "Bring back", which does not apply to a review thread
in another project. If hiding that for review threads takes more than a few lines there, show
instead the review's state and latest agent message in the detail, with a button that opens the
full thread.

**Pull requests tab.**

- Three groups: From teammates, Yours, Bots.
- Each row: author avatar and name, title (up to two lines), repository and number, size, one text
  pill per other person involved (small avatar, name, and their review state in words: approved,
  commented, changes requested, review requested), CI result with counts, comment count with
  unresolved count, status pill, and the review agent's state.
- Detail for the selected row: people, CI checks, comment counts, and the draft review with Post,
  Edit, Re-review and Discard. If the repository is a project, a link opens the existing PR detail
  panel for diff and timeline.
- A count per status across the top.
- Status is never shown by color alone: each pill has a distinct glyph and label.
- Avatars come from GitHub's avatar host; rows fall back to initials if an image fails.

**Rules and Repos tabs.** Plain text editors for the two files, showing validation errors from the
server, plus the config folder path and the `automatic` switch for this machine.

The mockup shown to the developer in the planning thread (third pass, 2026-11-12) is the reference
for which fields a row carries and how pills read. Its two legend rows explained the mockup and are
not part of the product. Its layout (one column with a chat line at the bottom) is superseded by
the layout above. It is not in the repository because it contains private names.

### Upstream files this will edit

Not a hard limit: the principle is new files wherever possible and edits of a few lines elsewhere.
Known edits:

- `apps/server/src/server.ts`: compose the new service layer.
- `apps/server/src/ws.ts`: register the RPC handlers.
- `apps/server/src/observability/RpcInstrumentation.ts`: an entry per new RPC method.
- `apps/server/src/environment/ServerEnvironment.ts`: the capability.
- `apps/server/src/mcp/McpHttpServer.ts`, and `McpToolExposure.ts` if needed: the toolkit.
- `apps/server/src/provider/agentContextBudget.json` and the mock layers in its test.
- `apps/server/src/config.ts` and the server settings contract: paths and the two local settings.
- `packages/contracts/src/rpc.ts`, `environment.ts`, `clientRpcPermissions.ts`, and the index.
- `packages/source-control-core` driver interface and `packages/source-control-github` driver: one
  optional method and its GraphQL operation.
- `apps/web/src/rightPanelStore.ts`, `RightPanelSurfaceContent.tsx`, `RightPanelTabs.tsx`: the new
  panel kind.
- `apps/web/src/components/sidebar/SidebarChrome.tsx` and `CommandPalette.tsx`: entry points.

## Order of work

Each step is usable on its own.

1. Files, GitHub read, status function, snapshot stream, and the pane with the list. No agents.
2. Triggers, review threads, draft handback, the detail view, and the `automatic` switch.
3. The master reviewer.
4. Posting to GitHub, manual first, then automatic.

## Testing

Per the repository's rules: test the behavior that can break, at the lowest layer.

- Status and trigger functions: table tests, including bots, own PRs, drafts, both trigger paths,
  team requests, and a missing review decision.
- File parsing: valid, missing, invalid header, unknown key.
- Store: state transitions and one review per PR and head SHA.
- Sweep: with a fake GitHub reader, that a matching PR queues exactly one review, that a new head
  SHA queues a re-review, that `max_concurrent` holds, that an invalid rules file queues none, and
  that nothing is queued while `automatic` is off.
- Posting: the draft maps to the expected `PullRequestSubmitReviewInput`; automatic posting happens
  only for a listed repository with the switch on, always as a comment, and never when the viewer
  already reviewed that head SHA.
- No browser tests unless a layout problem needs one.

## Left to the builder

- Exact names of services, tables, RPC methods, settings and files.
- The wording of the review and master reviewer prompts.
- How the project for the config folder is created, using the existing project creation service.
- Which existing actor and creation-source values a review thread uses, or whether to add one.
- Draft JSON shape beyond the fields listed.
- Visual details not fixed by the mockup.

## Risks

- **Search rate limit.** GitHub's search API allows about 30 requests a minute. Two searches per
  120 seconds, plus pages, is far below it; the manual refresh needs a short cooldown.
- **Cost.** Every review is an agent run. `max_concurrent`, the author exclusions, the per-machine
  switch and the one-review-per-head-SHA rule are the controls.
- **Automatic posting.** It speaks under the developer's name to colleagues. It is off unless a
  repository is listed, comment-only, and guarded against double posts, but a wrong review will
  still be public.
- **Upstream drift.** The exact-match context budget file conflicts whenever upstream changes tool
  text, and the panel files change often. The edits listed above are where conflicts will occur.
- **Pane width.** The list has many columns and now lives in a side panel. Pills wrap under the
  title to cope; check it at the panel's narrowest width.
- **Avatar loading.** The web app's content security policy may block GitHub's avatar host; check
  before relying on it.
