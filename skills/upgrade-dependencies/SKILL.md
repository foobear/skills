---
name: upgrade-dependencies
description: >-
  Runs a dependency upgrade round for a project: lists every available upgrade (including majors
  the project has pinned against, and runtime versions such as Ruby or Node), lets the human pick
  with a traffic-light table, then does the chosen upgrades commit by commit. Reads changelogs so
  that config and code follow the version bump, re-bases copied-in config files on the library's
  current template while keeping the project's customizations, verifies each commit, and hands
  over a summary of what the human still has to do. Use when the user asks to upgrade or update
  dependencies, libraries, gems or packages, to run `bundle update` or `pnpm update`, or to check
  what a version bump needs besides the new number.
# Names the session on the human's next prompt, unless they named it already.
hooks:
  UserPromptSubmit:
    - hooks:
        - type: command
          once: true
          command: >-
            python3 -c 'import datetime, json, sys;
            named = json.load(sys.stdin).get("session_title");
            title = "upgrade dependencies " + datetime.date.today().isoformat();
            named or print(json.dumps({"hookSpecificOutput": {"hookEventName": "UserPromptSubmit", "sessionTitle": title}}))'
---

# Upgrade dependencies

The human wants the project's dependencies brought up to date without surprises: in commits that
can be reviewed and bisected, with the application's config and code following the new versions,
and with the human choosing which upgrades happen. You do the discovery and the work; the human
makes the choices at two pauses and gets a summary at the end.

## Dependency check

This skill uses other skills from triskweline/skills. Your harness may announce only some
of the installed skills to you, so check which of them are installed by looking in its
standard skill locations. A skill you do not find there is missing.

| Skill | Role in this skill |
| --- | --- |
| `/work-in-branch` | Puts the upgrade commits on a feature branch before the first change. |
| `/pass-all-checks` | Enumerates the project's checks during discovery and runs the full suite on the final tree. |
| `/self-review` | Reviews the hand-written changes (config alignment, code adaptations) before the hand-off. |

If all are available, say nothing about it and go on. For each missing skill, say in one
line that it is missing and what you use instead: another skill you have that fills the
role, or that you do the step yourself. Then continue. Where this skill's instructions name
a missing skill, use your substitute for the rest of this run. Do not ask whether to install
or what to substitute, and never install anything yourself.

## Rules

These four rules shape every round. The phases below only spell out how to follow them.

- **Minor and patch upgrades stay together in one sweep commit.** Nobody reviews thirty
  one-line bumps separately.
- **A larger upgrade gets its own commit.** Larger means: a new major version, or an upgrade
  whose follow-up work is more than a few small edits: a generator run, migrations, a config
  file re-based on a new template, code changes beyond a handful of lines, even when the
  version is a minor one. Small follow-ups stay in the sweep commit: a renamed setting, two
  files with style fixes after a linter update.
- **Majors are offered, never forced.** Offer every available major, including the ones the
  project's manifest pins against (`~> 1.2`, `^1.2.0`): a pin alone is not a reason to skip.
  A comment next to the pin that gives a reason *is* worth surfacing; the human decides.
  Each major is a separate choice and a separate commit.
- **The version number is not the upgrade.** Read the changelog or upgrade guide between the
  old and the new version, and make the project follow it: renamed settings, changed defaults,
  removed APIs, new generators, new migrations. A config file that an installer once copied
  into the project is re-based on the library's *current* template, with the project's own
  customizations re-applied, so that the next upgrade can start from a known shape again.

## Phase 1: Discover

Find out what the project has and what it could get. Change nothing yet.

**Package managers and runtimes.** Find every manifest with a lockfile (e.g. `Gemfile` and
`Gemfile.lock`, `package.json` and its lockfile) and every place a runtime or tool version is
pinned: version files (`.ruby-version`, `.nvmrc`, `.node-version`, `.tool-versions`,
`mise.toml`), a `packageManager` field, every stage of every Dockerfile, container images in the
CI config, and the setup instructions in `README.md`, `CLAUDE.md` or `AGENTS.md`. Keep that list
of pin locations; a runtime upgrade has to touch all of them.

**Cooldowns.** Check what minimum release age each package manager is configured with (e.g.
pnpm's `minimumReleaseAge`, Bundler's cooldown setting). If none is configured, hold back
releases younger than a week and say so in the table.

**Candidates.** List outdated dependencies per package manager, without restricting the output to
the declared ranges (e.g. `bundle outdated` without `--strict`, `pnpm outdated`). For every
candidate, find out which of these it is:

- a minor or patch upgrade within the declared range: part of the sweep,
- a major, or an upgrade that is likely to need substantial follow-up work: a row of its own.
  Likely means the library ships generators, migrations or config templates, or the project
  holds a file that its installer generated (an initializer, a config file, a CI template),
- blocked by another dependency's constraint (e.g. gem A requires B `< 6`): not choosable until
  that owner relaxes; show it with the blocker's name,
- held back by a cooldown: not choosable today; show the date it becomes available.

**Security relevance.** The outdated listings show version numbers only, so look for security
fixes separately:

- Run the audit each package manager offers against the lockfile (e.g. `pnpm audit`,
  `npm audit`, `bundle-audit check --update` when the project has bundler-audit). They report
  published advisories with the fixed version.
- When a hosting platform CLI is configured (`gh`, `glab`), read the repository's dependency
  alerts there.
- For runtime candidates (Ruby, Node, the package manager), read the release announcements:
  they say when a release is a security release.

Mark what you find in the row's notes, also for rows a cooldown holds back: the human may decide
that such a fix is worth bypassing the cooldown for, and needs to see it to decide. When no audit
tool is available, say so above the table, so the human knows security relevance was not checked.
Do not install an audit tool for this.

For a runtime or tool (Ruby, Node, the package manager itself), look up the newest patch of the
current major and the newest major, and offer each as a row.

**The project's checks.** Use the `/pass-all-checks` skill's enumeration step to learn the
project's linters, test runners, license check and asset build, and how the project runs them.
Run the fast ones (linters, JS tests, build) once before changing anything, so that a failure
later is known to be new.

## Phase 2: Select

Print the candidate table and let the human choose. Every selectable row starts yellow.

```
| # | | Upgrade | From → To | Notes |
|---|---|---|---|---|
| 1 | 🟡 | Ruby | 4.0.6 → 4.0.9 | pinned in .ruby-version, Dockerfile (2 stages), CI image |
| 2 | 🟡 | Node.js | 24.18.0 → 24.22.1 | security fixes; .nvmrc, Dockerfile asset stage |
| 3 | 🟡 | Sweep: minor and patch updates | 23 gems, 14 packages | notable: rails 8.1.3 → 8.1.4, view_component 4.16 → 4.17 |
| 4 | 🟡 | good_job | 4.19 → 4.22 | minor, but ships 20 update migrations and a renamed setting |
| 5 | 🟡 | vitest | 4.1 → 5.0 | major; migration guide lists changed defaults |
| 6 | 🟡 | ransack | 4.3 → 5.0 | major; Gemfile pins `~> 4.0` with comment "5 breaks our sort helper" |
| 7 | ⚪ | rubyzip | 2.4 → 3.0 | blocked: license_finder requires `< 3` |
| 8 | ⚪ | makandra-rubocop | 17 → 18 | cooldown until 2026-10-29 |
```

The lights:

- 🟡 undecided. Every selectable row starts here; the notes column may carry your
  recommendation, the light may not.
- 🟢 do it.
- 🔴 skip it this round.
- ⚪ not selectable (blocked, in cooldown). Shown for information, never asked about. When the
  human turns such a row green anyway, e.g. for a security fix inside the cooldown, do the
  upgrade through the package manager's exception mechanism or by installing the exact version,
  and say so in the commit message.

The human switches lights by typing row numbers:

- A bare number, `+1`, `do 1`, `yes 1` or similar turns a row green.
- `-1`, `skip 1`, `drop 1`, `no 1` or similar turns it red.
- Lists and ranges work: `1 3 5`, `2-4`, `-6 -7`.
- `rest green` or `rest red` resolves every yellow row at once; `all` and `none` are the same
  for every selectable row.
- `go` starts the work. It is only valid when no yellow row remains; otherwise re-print the
  table and ask about the yellow rows.

Anything else is prose and gets a prose answer, e.g. a question about what row 4 would change.
Re-print the whole table after every reply, table first and your text after it; assume anything
printed before the table scrolls out of view. Row numbers are stable for the whole round, also
in the summary at the end.

## Phase 3: Check

For every green row, find out what the upgrade needs besides the version number, and report it
before changing anything.

**Read before you bump.** Read the changelog, release notes or upgrade guide for the range
between the project's version and the target. Look for: renamed or removed settings, changed
defaults, removed or changed APIs the project uses, new or changed generators, new migrations,
new required peer dependencies, new runtime requirements (`engines`, `required_ruby_version`),
and license changes. Also check what a direct upgrade drags along: a transitive major that rides
in with it (e.g. a linter pulling in a new major of a parser) is checked as part of this row, not
as a row of its own.

**Sweep rows, too.** After computing the sweep, skim the lockfile diff for notable bumps: the
framework, libraries with generators or migrations, anything whose minor releases are known to
carry deprecations. Read their notes the same way. Small follow-ups (a renamed setting, a few
style fixes) are made in the sweep commit and listed in the report. When a sweep item turns out
to need substantial work, take it out of the sweep and give it a commit of its own; say so in
the report.

**Copied-in config files.** When the project holds a file that the library's installer generated
(an initializer, a config template, a CI snippet), re-base it instead of patching it:

- Get the template of the *installed* version (from the installed package, or by fetching that
  version) and diff it against the project's file. The diff is the project's customizations.
- Take the template of the *target* version and re-apply exactly those customizations.
- Drop blocks an upgrade generator appended under an "added by" banner once their content is in
  the template's own place. Stale comments about old defaults go, too.
- Run the library's own status or check task afterwards, when it has one, to confirm the config
  mentions every setting it expects.

**Generators and migrations.** Run a generator with its dry-run flag (`--pretend`) first and read
what it would do. Note whether the migrations it adds are guarded and idempotent, and whether
they need anything on deploy.

**Report, then pause.** Print the table again, then one block per green row saying what the
upgrade needs: nothing but the bump, or the config changes, code changes, generator runs and
migrations you found, each in one line. The human may turn rows red here, or ask questions. Go on
when they confirm.

## Phase 4: Apply and commit

Before the first change, use the `/work-in-branch` skill.

**Order.** Runtime and tool upgrades come first, each as its own commit, so that every later
commit is verified with the toolchain it declares. Then the sweep. Then the other rows, in
dependency order: a library before the plugin that needs its new version (e.g. a bundler before
a bundler plugin), a linter before the cops that come with it.

**Making an upgrade.** Change the declared range in the manifest and let the package manager
resolve conservatively (e.g. `bundle update <gem> --conservative`, `pnpm update <package>`),
rather than adding the package anew: an add command can move a dependency between sections
(`dependencies` to `devDependencies`), which breaks image builds that install without dev
dependencies. Check the manifest diff for exactly that before committing.

**A sweep that pulls in a major.** When the sweep's resolution brings a new major along (because
a range allowed it, or a transitive dependency did), pin it back, redo the sweep, and offer the
major as a row of its own. The human did not choose it yet.

**Runtime upgrades.** Touch every pin location from the discovery list in one commit, including
Dockerfile stages, the CI image reference and the docs. When the CI image is built from a
Dockerfile in the repo, bump its tag and tell the human in the summary that the image needs a
rebuild and push before the branch can go through CI; until then CI may silently run the old
runtime.

**Commit messages.** Follow the project's conventions. Without any: `Upgrade to <library> <major>`
for a single upgrade, `Update dependencies` or the sweep command itself (`bundle update && pnpm
update`) for the sweep. Name the library correctly; re-read the subject against the manifest.

**Shared history stays as it is.** A commit that is already on the main branch, or on a branch
colleagues work from, gets a follow-up commit with a clear subject, never a fixup. Rewriting an
unpushed branch is fine; after any scripted rewrite (`rebase`, `cherry-pick`), check that the
branch still holds every commit it should.

**Sanity checks before each commit:**

- New migrations sort at or near the end of the migrations directory. A generator that copies
  migrations under the library's own timestamps puts them before the project's install
  migration, which fails on a database that has not run the install yet.
- The manifest diff shows only range changes, no section moves.
- After a build-tool upgrade, the build output is compared with the output before (byte-identical
  apart from hashes, or differences you can explain).
- The license check still passes. A transitive upgrade can change a package's declared license
  identifier (e.g. to its SPDX name) without changing the license; a new permit needs a reason
  that says so.
- New required peer dependencies are satisfied; the package manager's peer check is clean.
- A changed default that lives outside the repo (an environment variable on the servers, a
  hostname the callback URLs now use) goes into the summary for the human.

## Phase 5: Verify

Every commit must stand on its own, so that `git bisect` and partial reverts work.

**Per commit:** check out each commit in its own worktree with the runtime versions that commit
declares, install dependencies from the lockfile, and run the fast checks from discovery:
linters, JS tests, license check, asset build. Worktrees run in parallel. Fix a failure in the
commit that caused it (amend or fixup into it), not in a trailing fix commit.

**On the final tree:** run the full suite through the `/pass-all-checks` skill. For a small suite
you may run it per commit instead.

**Then review** the hand-written parts (config alignment, code adaptations) with the
`/self-review` skill, and fold the valid findings back into their commits.

## Phase 6: Hand off

Print the final table with its lights, then a summary with these parts:

- **Commits**, oldest first, each with one line on what it changed beyond the version number.
- **What the human still has to do**: install the new runtime locally, run the install command
  for their own environment, rebuild and push a CI image, set or rename an environment variable
  on the servers, confirm a hostname, push the branch. Say plainly what you could not verify
  (image builds, CI, deploys) and what would confirm it.
- **Held back**: rows kept red by the human, rows in cooldown with the date they become
  available, rows blocked by another dependency with the blocker's name. Suggest when to run
  the next round.

Never push. Never rewrite history that others have.

## Begin your work

Now discover the project's package managers, runtimes, cooldowns and candidates, and print the
selection table.
