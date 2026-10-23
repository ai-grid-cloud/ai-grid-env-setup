# Run your own clusters from the template

This page explains how to keep your own environment up to date with this repository without forking
it or keeping a long-lived branch. You keep two repositories: this one as the template, and a small
cluster repository of your own that holds only what's specific to your clusters.

**Status:** pending the split PR. The template doesn't yet read a second repository: the multi-source
ApplicationSets and the `CLUSTER_REPO` and `CLUSTER_REF` settings come with that PR. Until it merges,
cluster values live in this repository, on the branch Argo CD reads. The two-repository render was
checked on a local Argo CD v3.5.5, the version OpenShift GitOps 1.22 runs (2026-10-23); nothing on this
page has run on a real hub yet.

## Two repositories

| Repository | Holds | Who changes it |
|---|---|---|
| `ai-grid-env-setup` (this one) | The template: every chart, kustomization, script, and check, with a default for every value | Pull requests to the template |
| Your cluster repository, such as `ai-grid-cloud/demo-cluster` | Cluster values (invites, certificate digests, provider IPs, consumer sites, image pins), and `catalog/` (models and subscriptions) | You, the scripts, and the backend's git writer, which commits to `catalog/` |

The two repositories never hold the same file. A template change never conflicts with a cluster
change, so you never rebase or merge one into the other.

A cluster repository has the following layout:

```text
clusters/CLUSTER/
  env.yaml                  # templateRef: the template commit this hub runs
  values/
    ai-grid-spokes.yaml     # hubSiteDigest, siteDigests, providerGlobalIPs, consumerSites
    grid-enrollment.yaml    # grid-enrollment.invites
  console-images.yaml       # optional console image pins
catalog/
  locations/LOCATION/       # models, by the spoke's location label
  models/                   # ExternalModel and MaaSModelRef for entry spokes
  subscriptions/            # one MaaSSubscription and its auth policy per file
```

Replace the following:

- `CLUSTER`: your hub's name in the cluster repository, such as `demo1`.
- `LOCATION`: a spoke's `location` label, such as `us-east`.

Argo CD on the hub reads the template at `templateRef` and adds your values on top. For each Helm app
that the template marks with `clusterValues`, it reads
`clusters/CLUSTER/values/APP.yaml` from your cluster repository if the file exists. If the file
doesn't exist, the app uses the template's defaults.

## Pin the template by commit

`templateRef` is a full commit SHA, never a branch name. A branch moves on its own; a SHA changes only
when you change it, so you always know which template a hub runs.

To update to a newer template:

1. Change `templateRef` in `clusters/CLUSTER/env.yaml` to the new commit, in a pull request to your
   cluster repository. Renovate can open this pull request for you.
2. Check that the pull request's CI passes. For more information, see
   [Check a bump before it syncs](#check-a-bump-before-it-syncs).
3. Merge it. Argo CD rolls the new template out to the hub and the spokes.
4. Run `make verify` against the hub and your spokes.

If the update breaks something, revert the same one line. Argo CD rolls back to the earlier template.

## Test an unmerged template pull request

To try a template pull request on your clusters before it merges:

1. Set `templateRef` to the pull request's branch. A branch in a fork works.
2. Wait for Argo CD to sync, and then run `make verify`.
3. Set `templateRef` back to the pinned SHA.

To test several pull requests together, create a throwaway branch that merges all of them, point
`templateRef` at it, and delete the branch when you're done. Don't keep it: it's a test, not a place
to collect changes.

> **Caution:** While `templateRef` names a branch, every push to that branch reaches your clusters.
> Point it back to a SHA as soon as the test ends.

## The values contract

The template promises that every value has a default. A new value in the template never requires a
change to your cluster repository, so a template update never breaks a cluster whose repository
didn't change. Your values files override only what's specific to your clusters.

When you add a value to the template, give it a default in the app's `values.yaml` that renders on a
hub with no spokes, such as an empty map or list. When you add a Helm app that clusters configure,
add `clusterValues` to its `application.yaml`.

## Check a bump before it syncs

CI in the cluster repository renders every app with the template at the new `templateRef` and the
cluster's own values files. A bump that doesn't render, or a values file that names a key the
template no longer has, fails in the pull request instead of on the hub.

This CI isn't written yet. It's part of setting up a cluster repository, pending the split PR.

## Why not a fork or a cluster branch

A long-lived branch of the template, or a fork, holding your cluster's changes looks simpler, but it
costs more over time:

- **Conflicts.** Every template change has to be merged into the branch. The `integration/demo1`
  branch drifted far enough that bringing it up to `main` took a rebuild, with 11 conflicting files.
- **Force-moving what Argo CD syncs.** A rebuilt branch replaces the history that Argo CD follows, and
  every hub on that branch takes the change at once, with no pull request to review it.
- **Cluster detail in a public template.** Digests, IPs, and image pins for one environment end up in
  the template's history, where everyone who forks it gets them.

With two repositories, the template stays generic, your cluster repository stays small, and an
update is one reviewed line.

## What's next

To build a new environment from the template, start at [Check the prerequisites](1-prerequisites.md).
