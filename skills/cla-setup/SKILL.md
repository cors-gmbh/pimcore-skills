---
name: cla-setup
description: Set up Contributor License Agreement (CLA) for CORS open source repositories on GitHub
allowed-tools: Read, Grep, Glob, Bash, Edit, Write, Task
---

# CLA Setup

You are helping set up a Contributor License Agreement (CLA) for a CORS GmbH open source repository on GitHub.

## Overview

CORS uses the [CLA Assistant GitHub Action](https://github.com/contributor-assistant/github-action) to automatically manage CLA verification on pull requests. Contributors must sign the CLA before their PRs can be merged.

## Step 1: Determine the GitHub repository name

Check the git remote to find the GitHub repository name:

```bash
git remote -v
```

The repo name is needed for the `path-to-document` URL in the workflow. Format: `cors-gmbh/<repo-name>`.

## Step 2: Create the CLA document

Create `CLA.md` in the project root with the standard CORS CLA text:

```markdown
# Contributor License Agreement

The following terms are used throughout this agreement:

**You** - the person or legal entity including its affiliates asked to accept this agreement. An affiliate is any entity
that controls or is controlled by the legal entity, or is under common control with it.

**Project** - is an umbrella term that refers to any and all CORS GmbH in combination with instride AG open source projects.

**Contribution** - any type of work that is submitted to a Project, including any modifications or additions to existing
work.

Submitted - conveyed to a Project via a pull request, commit, issue, or any form of electronic, written, or verbal
communication with CORS GmbH and instride AG, contributors or maintainers.

# 1. Grant of Copyright License.

Subject to the terms and conditions of this agreement, You grant to the Projects' maintainers, contributors, users and
to CORS GmbH and instride AG a perpetual, worldwide, non-exclusive, no-charge, royalty-free, irrevocable copyright license to reproduce,
prepare derivative works of, publicly display, publicly perform, sublicense, and distribute Your contributions and such
derivative works. Except for this license, You reserve all rights, title, and interest in your contributions.

# 2. Grant of Patent License.

Subject to the terms and conditions of this agreement, You grant to the Projects' maintainers, contributors, users and
to CORS GmbH and instride AG a perpetual, worldwide, non-exclusive, no-charge, royalty-free, irrevocable (except as stated in this
section) patent license to make, have made, use, offer to sell, sell, import, and otherwise transfer your contributions,
where such license applies only to those patent claims licensable by you that are necessarily infringed by your
contribution or by combination of your contribution with the project to which this contribution was submitted.

If any entity institutes patent litigation - including cross-claim or counterclaim in a lawsuit - against You alleging
that your contribution or any project it was submitted to constitutes or is responsible for direct or contributory
patent infringement, then any patent licenses granted to that entity under this agreement shall terminate as of the date
such litigation is filed.

# 3. Source of Contribution.

Your contribution is either your original creation, based upon previous work that, to the best of your knowledge, is
covered under an appropriate open source license and you have the right under that license to submit that work with
modifications, whether created in whole or in part by you, or you have clearly identified the source of the contribution
and any license or other restriction (like related patents, trademarks, and license agreements) of which you are
personally aware.
```

**Important:** This CLA text is standardized across all CORS repositories. Do NOT modify it.

## Step 3: Create the GitHub Actions workflow

Create `.github/workflows/cla-check.yml`:

```yaml
name: CLA Check
on:
  issue_comment:
    types: [created]
  pull_request_target:
    types: [opened, closed, synchronize]

permissions:
  actions: write
  contents: write
  pull-requests: write
  statuses: write

jobs:
  CLAAssistant:
    runs-on: ubuntu-latest
    steps:
      - name: "CLA Assistant"
        if: (github.event.comment.body == 'recheck' || github.event.comment.body == 'I have read the CLA Document and I hereby sign the CLA') || github.event_name == 'pull_request_target'
        uses: contributor-assistant/github-action@v2.5.1
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        with:
          path-to-signatures: '.github/signatures/version1/cla.json'
          path-to-document: 'https://github.com/cors-gmbh/<REPO-NAME>/blob/main/CLA.md'
          branch: "main"
          allowlist: user1,bot*
```

**Important:** Replace `<REPO-NAME>` with the actual GitHub repository name (e.g., `pimcore-prometheus`, `pimcore-webcare`).

## Step 4: Create the signatures file

Create `.github/signatures/version1/cla.json` with an empty signatures array:

```json
{
  "signedContributors": []
}
```

This file is automatically updated by the CLA Assistant action when contributors sign the CLA. The action commits signature entries with the contributor's GitHub username, ID, timestamp, and PR number.

## Directory structure

After setup, these files should exist:

```
├── CLA.md
├── .github/
│   ├── workflows/
│   │   └── cla-check.yml
│   └── signatures/
│       └── version1/
│           └── cla.json
```

## How it works

1. A contributor opens a PR
2. The CLA Assistant checks if the contributor has signed the CLA
3. If not signed, a comment is posted asking them to sign by commenting: `I have read the CLA Document and I hereby sign the CLA`
4. Once signed, the contributor's signature is recorded in `cla.json` and the check passes
5. The `recheck` comment can be used to re-verify CLA status

## Notes

- The `allowlist` in the workflow can be customized to skip CLA checks for specific users or bots (e.g., `dependabot*`, `renovate*`)
- The `path-to-document` must point to the CLA on the `main` branch
- The `branch` setting determines where signature commits are pushed (should be `main`)
- No additional secrets are needed beyond the default `GITHUB_TOKEN`