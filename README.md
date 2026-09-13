# Claude GitHub Workflows

A template repository holding three Claude-powered GitHub Actions workflows. Nothing else — no
language, no build, no project scaffolding. Start a repo from this template, or copy the
`.github/workflows/` directory into a repo you already have.

The workflows read your project's own context files (`CLAUDE.md`, `AGENTS.md`, `README.md`,
`CONTRIBUTING.md`) rather than carrying assumptions about the language or layout, so they work
unchanged across projects.

## The workflows

| File | Fires on | What it does |
|---|---|---|
| `issue-triage.yml` | An issue is opened or edited | Applies labels from those the repo already defines, then leaves one triage comment: type, effort, likely duplicate, likely files, missing information. |
| `pr-review.yml` | A pull request is opened, reopened, or pushed to | Reviews the diff against the project's stated conventions and leaves one review comment ranked by what would actually bite. Never approves — a human owns that. |
| `claude-mention.yml` | A comment containing `@claude` | Answers the question in the thread. Reads the code and the diff before answering; describes changes rather than making them. |

All three are read-only against the repository. Their only writes are labels and comments.

## Setup

1. **Create the repo.** Either use this repository as a template on GitHub, or:

   ```bash
   gh repo create my-project --template willowsenator/claude-github-workflows --private
   ```

   To add the workflows to an existing repo, copy `.github/workflows/` into it.

2. **Add the authentication secret.** Every workflow needs `CLAUDE_CODE_OAUTH_TOKEN`:

   ```bash
   claude setup-token                                    # prints a long-lived OAuth token
   gh secret set CLAUDE_CODE_OAUTH_TOKEN --repo owner/repo
   ```

   Set it once per organization instead if you are rolling this out widely
   (`gh secret set CLAUDE_CODE_OAUTH_TOKEN --org my-org --visibility all`).

   `GITHUB_TOKEN` is provided by Actions automatically — you do not create it.

3. **Delete what you do not want.** Each file stands alone; removing one does not affect the others.

## Behaviour worth knowing before you turn these on

- **Idempotent comments.** Triage and review each maintain exactly one comment, identified by a
  hidden marker on its first line (`<!-- claude-triage -->`, `<!-- claude-pr-review -->`) and edited
  in place on re-runs. Editing an issue or pushing to a pull request updates that comment rather
  than adding another. Mention replies are the exception: a conversation gets a new comment each
  time.
- **The review is verified, not just trusted.** `pr-review.yml` runs Claude in agent mode, which has
  no built-in tracking comment — posting or editing the marker comment is entirely up to the model
  following the prompt's last step, with no structural guarantee it does so on every run. A step
  after the review checks whether the marker comment's timestamp actually moved and fails the job
  loudly if it did not, rather than reporting a silent pass on a run that reviewed nothing. A pull
  request that edits `pr-review.yml` itself will always fail this check (GitHub's own
  workflow-validation guard skips the review step there) — see the comment above the verification
  step in that file.
- **Bots are skipped.** Issues, pull requests and comments authored by bots never trigger a run.
  Dependabot and Renovate pull requests are therefore not reviewed; drop the `user.type != 'Bot'`
  condition in `pr-review.yml` if you want them to be.
- **Mentions are restricted to people with write access.** `claude-mention.yml` requires the
  commenter's `author_association` to be `OWNER`, `MEMBER` or `COLLABORATOR`, so a drive-by
  commenter on a public repo cannot spend your Actions minutes. Relax that condition deliberately,
  not by accident.
- **Draft pull requests are not reviewed** until marked ready.
- **Untrusted input is treated as data.** Issue text, pull request bodies and diffs can all contain
  text aimed at the model. Each prompt states that such text is data and never instructions, and
  asks Claude to report attempts in its comment. This is mitigation, not a guarantee — keep the
  permissions in these files as narrow as they are.
- **Concurrency.** Rapid edits to one issue, or several pushes to one pull request, supersede each
  other instead of piling up runs. Mention replies queue rather than cancel.
- **Actions are pinned to commit SHAs**, not tags, with the human-readable version in a trailing
  comment. A tag is mutable: whoever controls it can repoint `v1` at new code, which would then run
  with `issues: write` and your Claude token. A SHA cannot be repointed. The cost is that pins do
  not move on their own — see below.

## Updating the pinned actions

Resolve a tag to its commit and replace both the SHA and the trailing comment:

```bash
gh api repos/actions/checkout/commits/v7 --jq '.sha'
gh api repos/anthropics/claude-code-action/commits/v1 --jq '.sha'
```

Use `repos/OWNER/REPO/commits/TAG` rather than the refs API: most of these tags are *annotated*, so
`git/matching-refs` returns the tag object's SHA, and a `uses:` pin needs the commit it points at.

You rarely have to. `.github/dependabot.yml` ships with this template and does it for you: weekly,
grouped into a single pull request, rewriting the pinned SHA and the trailing version comment
together. The commands above are for the times you want to bump something now rather than wait.

Two consequences of that file being here:

- **Dependabot's pull requests are not reviewed by Claude.** `pr-review.yml` skips bot-authored pull
  requests. That is intended — a SHA bump is a diff Claude has nothing useful to say about — but it
  does mean nothing automated reads them. Merge them yourself.
- **Only the `github-actions` ecosystem is declared.** A repo created from this template that also
  has an `npm`, `pip` or `cargo` manifest needs its own `updates:` entry added. Declaring an
  ecosystem whose manifest is absent is a silent no-op rather than an error, so nothing breaks
  either way.

## Does this fit my project?

The workflows never assume a language or a build. What actually varies is how much signal Claude
gets from your repository.

**Works well as-is** — anything where the meaningful changes are text a diff can show: application
and service code in any language, infrastructure and configuration, documentation, schemas, scripts.

**Works, with caveats** — repositories whose real content is binary or generated:

- **Game engines (Unreal, Unity, Godot).** Issue triage and mentions work fine; they read text.
  Pull request review is the limited one: `.uasset`, `.umap`, `.blend` and Unity scene/prefab files
  produce no readable diff, so review only reaches C++, C#, Blueprint-adjacent source, build scripts
  (`*.Build.cs`, `*.Target.cs`), config `.ini` files and shaders. Expect the review comment's
  "Not reviewed" section to carry the asset files. Repositories using Git LFS are checked out with
  LFS pointers rather than file contents by default — that is fine here, since the workflows only
  ever read text, but it means an asset genuinely cannot be inspected.
- **Monorepos and very large diffs.** Review reads the diff through `gh pr diff`; a several-thousand
  line change gets a partial review, and the prompt requires Claude to say which files it skipped.
  Narrowing the trigger with a `paths:` filter is usually better than letting it skim everything.
- **Generated code.** Lockfiles, generated clients and compiled output waste review attention. A
  `paths-ignore:` filter on `pr-review.yml` is worth adding.

**Not much use** — repositories that are essentially an asset store, a binary release archive, or
data with no source. There is nothing for a reviewer to read.

The single change that improves output most, in any project, is a `CLAUDE.md` at the repository root
describing the layout, the conventions and how the project is built and tested. All three workflows
read it, and a project's stated conventions outrank the model's general preferences.

## Cost

Each run is a Claude Code invocation plus an Actions runner minute or two. On a busy public
repository, issue triage is the one that adds up — it fires on every opened and edited issue. Narrow
the triggers or drop a workflow if that matters to you.
