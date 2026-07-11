# Git vs GitHub Investigation Guide

A beginner-friendly guide that explains why a GitHub repository can show a recent "Updated"
timestamp even when `git log` does not show any recent commit.

![Markdown Lint](https://img.shields.io/badge/markdownlint-passing-brightgreen)
[![License: MIT](https://img.shields.io/badge/license-MIT-blue)](LICENSE)

Version francaise disponible ici : [README.fr.md](README.fr.md)

## Table of Contents

- [Why this project exists](#why-this-project-exists)
- [What you will learn](#what-you-will-learn)
- [Basic vocabulary](#basic-vocabulary)
- [When Git and GitHub tell different stories](#when-git-and-github-tell-different-stories)
- [A real example](#a-real-example)
- [Investigation method: how to reach this conclusion](#investigation-method-how-to-reach-this-conclusion)
- [Useful commands](#useful-commands)
- [Reading the results](#reading-the-results)
- [Common mistakes](#common-mistakes)
- [Special cases](#special-cases)
- [Recommended learning path](#recommended-learning-path)
- [Repository structure](#repository-structure)
- [Additional resources](#additional-resources)
- [License](#license)
- [Contributing](#contributing)
- [Help and contact](#help-and-contact)

## Why this project exists

Sometimes a GitHub repository page shows "Updated 3 hours ago", but running `git log` on your
local clone shows the last commit was made days ago. This is confusing for beginners, because
it looks like Git and GitHub disagree with each other.

They do not actually disagree. They are just reporting two different things:

- **Git** reports commits that exist in the branch you are looking at.
- **GitHub** reports repository activity, which includes things that are not commits on your
  current branch, such as merges, pull requests, or branch deletions.

This guide walks through the investigation method, step by step, so a complete beginner can
reproduce it on their own repositories.

## What you will learn

By the end of this guide, you will be able to:

- Read a `git log` output with confidence.
- Recognize a merge commit.
- Understand what a `DeleteEvent` is on GitHub.
- Use the `gh` command-line tool to list pull requests and repository events.
- Correctly interpret why GitHub activity and local Git history can differ.

## Basic vocabulary

These words are used throughout the guide. If you already know them, feel free to skip ahead.

- **Commit**: a saved snapshot of your project at a specific point in time.
- **Branch**: an independent line of work, created from the project's history, that lets you
  make changes without affecting the main line of work.
- **Merge**: the act of combining the changes from one branch into another.
- **Pull request (PR)**: a proposal to merge changes from one branch into another, reviewed
  before it is accepted.
- **DeleteEvent**: a GitHub event recorded when a branch or tag is deleted.

> **To remember:** a branch is a workspace, a commit is a saved step, and a pull request is a
> proposal to merge that work back into the main line.

## When Git and GitHub tell different stories

`git log` only shows commits that are part of the branch history you currently have checked
out. It does not show:

- Commits made on branches you have not fetched.
- Branches that were deleted after being merged.
- Repository-level events such as issue comments, releases, or settings changes.

GitHub's repository list page, on the other hand, shows a general "last activity" timestamp.
That timestamp can be updated by a merge, a branch deletion, or other repository events, even
if no new commit was added to the branch you are viewing locally.

> **Warning:** an "Updated" timestamp on GitHub is not proof that a new commit was pushed. It
> only means something happened in the repository.

## A real example

This is the situation that motivated this guide, on the `ShellFromBrowser` repository:

- The most recent commit visible in `git log` was 43 hours old.
- The GitHub repository list showed "Updated 3 hours ago", which did not match.
- Querying the GitHub Events API confirmed the exact cause of the mismatch.

The table below shows what the GitHub Events API returned for the event that updated the
repository's timestamp:

| Field | Value |
| :--- | :--- |
| Event type | `DeleteEvent` |
| Exact date | `2026-07-11T06:18:38Z` |
| Actor | `valorisa` |
| Resource deleted | Branch `valorisa-patch-2` |
| File changed | None, this is a Git metadata operation |

No commit and no file change caused the "Updated 3 hours ago" label. A branch deletion is
enough on its own to refresh a repository's activity timestamp on GitHub.

### Why the branch was deleted

The branch `valorisa-patch-2` was the source branch of pull request #6, which bumped the Go
version to 1.26.3. That pull request was merged on July 9. Deleting a branch right after it
has been merged is a common cleanup practice, and it matches this project's usual habit of
removing `feat/*` and `fix/*` branches once their work has been integrated.

### Cross-checking with other signals

Three additional signals confirmed the diagnosis:

- **Last commit**: `2092cbe`, from July 9, is the merge commit for pull request #6. No commit
  activity happened in the following 43 hours.
- **Last release**: `v0.4.0`, from May 18. No release activity happened recently either.
- **Pull request #6**: closed and merged on July 9. Its source branch, `valorisa-patch-2`, was
  deleted on July 11 at 06:18 UTC, which lines up exactly with the "3 hours ago" label.

Taken together, this confirms that the local clone was fully up to date, and that no action
was needed on the developer's side.

## Investigation method: how to reach this conclusion

The diagnosis above did not come from a single clue. It came from combining three types of
reasoning, which you can reuse on your own repositories.

1. **Spot a timing inconsistency.** A gap of a few minutes between `git log` and GitHub's
   "Updated" label is normal, caused by caching or network latency. A gap of several hours,
   like the 43 hours seen here, is not normal. A gap that large means the timestamp cannot be
   explained by a commit, which rules out the commit history as the source and points you
   toward repository-level events instead.
2. **Use your own working habits as a hypothesis.** If you know you usually delete feature
   branches after merging them, and a pull request was recently merged, a branch deletion
   becomes the most likely explanation for activity with no visible commit.
3. **Know what actually updates the "Updated" timestamp.** GitHub refreshes a repository's
   activity timestamp for any event on its refs, not only commits. `DeleteEvent`,
   `ReleaseEvent`, and even some issue or discussion activity can all trigger it. This is why
   `git fetch` and `git log` alone cannot answer the question: they only see commit history,
   never these metadata events. The GitHub Events API is the tool that can see them.

## Useful commands

The commands below can be run from Termux, or any terminal with `git` and the GitHub CLI
(`gh`) installed.

```text
# Show the full commit graph across all local branches
git log --oneline --decorate --graph --all

# Show only commits made after a given date and time
git log --since="2026-07-11 00:00" --oneline --all

# List merged pull requests targeting the main branch
gh pr list --state merged --base main

# List raw repository events, including branch deletions
gh api repos/OWNER/REPO/events
```

Replace `OWNER/REPO` with the actual owner and repository name, for example
`valorisa/ShellFromBrowser`.

## Reading the results

- If `git log --all` shows the commit you expected, the branch history is complete locally.
- If `gh pr list --state merged` shows a recent merge you do not see locally, run `git fetch`
  to update your local view of the remote branches.
- If `gh api repos/OWNER/REPO/events` shows a `DeleteEvent` around the time GitHub says the
  repository was updated, that explains the mismatch.
- Cross-check with your release history too: an unexpected "Updated" label with no matching
  commit, pull request, or release is a strong sign that the event was a branch or tag
  deletion, as shown in [A real example](#a-real-example).

## Common mistakes

- Assuming `git log` (without `--all`) shows every branch. By default it only shows the
  current branch.
- Assuming the "Updated" label on GitHub always means a new commit was pushed.
- Forgetting to run `git fetch` before comparing local and remote history.

## Special cases

- **Squash merges**: the pull request is closed as merged, but the individual commits from
  the feature branch may not appear in `main`, only a single squashed commit.
- **Force pushes**: history can be rewritten, which may make old commits disappear from
  `git log` even though they still exist in GitHub's event history for a while.
- **Protected branches**: some repository settings can restrict who can push directly,
  which changes how updates typically arrive on `main`.

## Recommended learning path

1. Read [Basic vocabulary](#basic-vocabulary) until the terms feel familiar.
2. Reproduce the commands in [Useful commands](#useful-commands) on one of your own
   repositories.
3. Compare your own results using the checklist in
   [Reading the results](#reading-the-results).
4. Review [Common mistakes](#common-mistakes) to avoid the most frequent
   misunderstandings.
5. Keep this guide as a reference the next time a repository's activity looks confusing.

## Repository structure

```text
repo/
|-- README.md
|-- README.fr.md
|-- LICENSE
|-- .gitignore
`-- .github/
    `-- workflows/
        `-- markdownlint.yml
```

## Additional resources

- GitHub's own documentation on writing a good README.
- GitHub's documentation on repository activity and events.
- The official `gh` command-line tool documentation.

## License

This project is licensed under the MIT License. See [LICENSE](LICENSE) for details.

## Contributing

Suggestions and corrections are welcome. Please open an issue describing the change you
would like to propose before submitting a pull request.

## Help and contact

If something in this guide is unclear, please open an issue on this repository describing
which section is confusing. This helps make the guide better for the next beginner.
