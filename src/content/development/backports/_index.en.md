---
title: "Backports"
date: 2021-06-21T16:03:26+02:00
weight: 5
---

Fixes for serious issues or regressions affecting previous releases may be backported to the corresponding branches,
to be included in the next release from that branch.

## Requesting a backport

Backports can only be requested on fixes made against a later branch, or the `devel` branch.
(This doesn’t mean that bugs can’t be fixed in older branches directly; but where relevant, they should first be fixed on `devel`.)

To request such a backport, identify the relevant pull request, and add the “backport” label to it.
You should also add a comment to the pull request explaining why the backport is necessary, and which branch(es) are targeted.
Issues should _not_ be labeled, they are liable to be overlooked or lack a one-to-one mapping to a code fix.

## Handling backports

Pending backports can be identified using
[this query, listing all non-archived pull requests with a “backport” label and without a “backport-handled” label](https://github.com/pulls?q=is%3Apr+archived%3Afalse+user%3Asubmariner-io+label%3Abackport+-label%3Abackport-handled).

Backports should only be handled once the reference pull request is merged.
This ensures that commit identifiers will remain stable during the backport process and for later history.

### Standalone pull requests

Backporting a commit is automated by running `make backport release=<release-branch> pr=<PR to cherry-pick>`. The `make` target runs a script,
[backport.sh](https://github.com/submariner-io/shipyard/scripts/shared/backport.sh). The script is originally being used by Kubernetes community
and has been reused, and edited to suit the needs, from [their repo](https://github.com/kubernetes/kubernetes/blob/master/hack/cherry_pick_pull.sh).
The script does the following:

1. Creates a PR on `<release-branch>` with title `Automated cherry pick of <original PR number> <original PR title>`.
2. Adds `backport-handled` label to the original PR.
3. Gives an option to create patches to a release branch without making a PR by setting `DRY_RUN` variable.
   When `DRY_RUN` is set the script will leave you in a branch containing the commits you cherry-picked.
4. Multiple PRs can be cherry-picked together by passing the comma-separated list of PRs to be cherry-picked like `pr=<PR1,PR2>`.   

The script assumes following values. Please change them according to your setup.
1. Remote for the upstream repository, `UPSTREAM_REMOTE`, is set to `origin`.
2. Remote for your forked repository, `FORK_REMOTE`  is set to `GITHUB_USER`.
3. `GITHUB_USER` needs to be set to your `GitHub username`.

### Pull requests requiring dependent backports

<!-- TODO skitt document dependent backports -->

## Reviewing backports

Backports need to go through the same review process as usual.
The author of the original pull request should be added as a reviewer.

Change requests on a backport should only concern changes arising from the specifics of backporting to the target release branch.
Any other change which is deemed useful as a result of the review probably also applies to the original pull request and should result in
an entirely new pull request, which might not be a backport candidate.
