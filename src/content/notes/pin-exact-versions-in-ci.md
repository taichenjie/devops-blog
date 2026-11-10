---
title: "Pin exact versions in CI"
description: "A version range in my Infracost setup broke overnight with no code change. An exact pin fixed it."
pubDate: 2026-11-10
draft: false
---

On 21 Oct my Terraform plan workflow passed. On 22 Oct the same workflow failed at the Infracost setup step, and nothing in my repo had changed.

~~~text
Warning: 0.10.x did not match any release version.
Error: Unexpected HTTP response: 404
~~~

## What I thought I had

~~~yaml
- name: Setup Infracost
  uses: infracost/actions/setup@v3
  with:
    api-key: ${{ secrets.INFRACOST_API_KEY }}
~~~

I thought this installed a fixed version of Infracost. I never set a `version`, so I assumed the action had one built in. It did, but the default is `0.10.x`. That is a range, not a version, so the action decides on every run which release to install.

## What actually happened

A range has to be resolved before anything can be downloaded. I read the action's source to see how. It calls the GitHub API to list Infracost's releases and picks the newest one that matches `0.10.x`. On 22 Oct that lookup stopped finding a match. I never found out exactly what changed on Infracost's or GitHub's side, and that is the point. My CI depended on a lookup I did not control.

I expected a failed lookup to stop the job with a clear error. Instead, the action logged a warning and fell back to the raw input. It built a download URL for a release literally named `v0.10.x`. No such release exists, so the download returned 404. The real cause was a warning two lines above the error.

## The fix

~~~yaml
    version: 0.10.47
~~~

An exact version removes the lookup completely. The action downloads straight from that release's URL. I checked the URL returned 200, and the pull request with the fix passed its own CI, which was the real test.

## Two things I expected CI to do, and it did not

The failing run was on a stacked pull request. When the pull request underneath it merged, GitHub moved its base to `main`. I expected that to start a new CI run. It did not, because a base change is an `edited` event, and the `pull_request` trigger only runs on `opened`, `synchronize` and `reopened` by default. A new push is what starts a run.

After merging the fix, I clicked "Re-run failed jobs" and expected it to pass. It failed the same way. A rerun replays the same commit with the same workflow file, so it never sees fixes made after that commit. I had to merge `main` into the branch and push to get a fresh run.

## What is still not pinned

Fixing the CLI made me look at the line above it. The action is still `infracost/actions/setup@v3`, and `v3` is a tag that whoever controls that repo can move to different code. GitHub's security guidance is to pin third-party actions to a full commit SHA. That matters more in my setup, because this job runs with AWS credentials. I plan to pin actions by SHA, with Renovate opening pull requests for new versions so pinned versions do not go stale.

## What I learned

I assumed a pipeline only changes when I change it. A version range breaks that assumption, because the version is chosen at run time by something outside my repo. An exact pin means the only way my pipeline changes is through a pull request I can review.
