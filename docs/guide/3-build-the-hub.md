# Build the hub

`make hub` turns a plain OpenShift cluster into the AI Grid hub. It installs OpenShift GitOps and
points Argo CD at your fork, and Argo CD builds everything else from `apps/`. The scripts then add
what can't live in git: your registry login, Keycloak's users, and the hub's certificate digest.

**Field-tested:** the hub build (2026-10-19 and later), on demo.redhat.com clusters.

Before you start, complete [Configure your clusters and fork](2-configure.md), including pushing the
branch that `git.ref` names in `clusters.local.yaml` with `git push -u origin BRANCH`.

## Run `make hub`

1. Build the hub:

   ```bash
   make hub
   ```

   `make hub` runs these scripts in order. Each one is safe to run again.

   | Script | What it does | What you see |
   |---|---|---|
   | `bootstrap/00-gitops.sh` | Installs the OpenShift GitOps operator, makes you an Argo CD admin, and creates the root ApplicationSet `apps`, which reads `apps/*` from your fork | `Argo CD now builds the hub from ...` |
   | `bootstrap/21-acm-feature-gates.sh` | Waits for ACM to finish installing, then turns on the `ManifestWorkReplicaSet` feature gate | Waits up to 30 minutes for the `MultiClusterHub` to be `Running` |
   | `bootstrap/25-hub-secrets.sh` | Creates Secret `rhoai-quay-pull` from `.env`, which an ACM policy copies to every spoke's pull secret | `waiting for Argo CD app ai-grid-policies` until that app exists |
   | `bootstrap/35-keycloak.sh` | Waits for the Keycloak realms, pins Keycloak's host name, and creates the test users and their passwords | Waits up to 15 minutes for each realm import |
   | `bootstrap/50-grid-hub-info.sh` | Waits up to 15 minutes for the hub's grid identity and prints its certificate digest if git doesn't have it yet | `commit this digest: set hubSiteDigest: "..."`, or nothing when git already matches |

   Most of the time goes to ACM installing; the script waits up to 30 minutes for it. A rerun on a
   built hub takes under a minute.

2. Optional: watch Argo CD build the hub:

   ```bash
   oc --kubeconfig .kube-cache/host get applications.argoproj.io -n openshift-gitops
   ```

   Use the full name `applications.argoproj.io`. On an ACM hub, `oc get application` means ACM's own
   kind and shows nothing useful.

   Every app ends `Synced` and `Healthy`. `ai-grid-policies` can show `Degraded` while a policy that
   only reports, such as `ai-grid-ready`, has nothing to report on.

## Commit the hub's digest

Every grid gateway pins the hub site's certificate digest, `hubSiteDigest`, from git. A new hub has a
new grid certificate authority, so its digest changes on every build.

1. Copy the digest that `make hub` printed into `apps/ai-grid-spokes/manifests/values.yaml`:

   ```yaml
   hubSiteDigest: "HUB_SITE_DIGEST"
   ```

   Replace `HUB_SITE_DIGEST` with the 64-character value. If you missed it, run
   `bash bootstrap/50-grid-hub-info.sh` again; it prints the line only when git differs from the hub.

2. Commit and push it to the branch Argo CD reads:

   ```bash
   git add apps/ai-grid-spokes/manifests/values.yaml
   git commit -m "hub: hubSiteDigest"
   git push
   ```

## Clear values from an earlier build

`apps/ai-grid-spokes/manifests/values.yaml` and `apps/grid-enrollment/manifests/values.yaml` hold
values from the build this repository was last tested on, such as `siteDigests.site-a` and
`invites.site-a`. They belong to clusters that you don't have.

- If you name your first spoke `site-a`, keep the entries; [Add a spoke](4-add-a-spoke.md) has you
  replace `siteDigests.site-a` with your own.
- Otherwise, delete those `site-a` entries after you join your first spoke, not before. Keep the
  `invites:` map non-empty, because `make spoke-join` adds new invites after the existing ones. Make
  the deletions before you answer `y` to `make spoke-join`'s commit prompt, or commit and push them
  yourself, because Argo CD reads only what you push.
- Keep the directory `spokes/clusters/site-a/`. `make spoke-join` copies its models list for every new
  spoke.

## Prepare the spokes and the console

1. Prepare spoke-wide settings:

   ```bash
   make spokes
   ```

   `make spokes` checks the pull-secret mode (an ACM policy handles it by default, so the script has
   nothing to do) and rewrites Keycloak's portal callbacks: one for every spoke invited in git, plus
   `local-cluster` while it's in AI Grid or in `SPOKES`, less `inference` spokes. It reads each spoke's
   apps domain from the hub, so it needs no spoke kubeconfig. It doesn't add any cluster to the grid.

2. Give the provider console its sign-in and Keycloak credentials:

   ```bash
   make console
   ```

   The output ends with the following line:

   ```
   ==> console: https://<route ai-grid-provider-console in ai-grid>; sign in with an OpenShift login that may patch managedclusters
   ```

3. Find the console's address:

   ```bash
   oc --kubeconfig .kube-cache/host get route ai-grid-provider-console -n ai-grid -o jsonpath='https://{.spec.host}{"\n"}'
   ```

4. Open that address and sign in with the hub cluster's OpenShift login. On a demo.redhat.com
   cluster, that's the cluster's own single sign-on page, not Keycloak realm `ai-grid`, which is for
   tenants. The console accepts users who may update `ManagedCluster` objects, such as a
   `cluster-admin`.

The **Clusters** page lists `local-cluster`, which is the hub itself. This guide keeps the hub out of
the grid, so it runs no OpenShift AI. For a one-cluster environment, you can instead add
`local-cluster` with role `combined`, as [Add a spoke](4-add-a-spoke.md#add-the-spoke-to-ai-grid)
describes for a remote spoke; ACM then installs OpenShift AI on the hub too.

## Check the hub

At this point, the check is that every Argo CD app is `Synced` and `Healthy` (step 2 of
[Run `make hub`](#run-make-hub)). `make verify` checks the hub and its spokes together, so run it
after you add your first spoke; [Add a spoke](4-add-a-spoke.md#check-the-spoke) shows how.

## Give someone else access

`make hub` makes you an Argo CD admin. To add another person after they can sign in to the hub:

```bash
oc --kubeconfig .kube-cache/host adm groups add-users cluster-admins USER_NAME
```

Replace `USER_NAME` with their OpenShift user name. The group `cluster-admins` maps to Argo CD's admin
role.

## What's next

[Add a spoke](4-add-a-spoke.md).
