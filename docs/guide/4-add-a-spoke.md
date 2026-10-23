# Add a spoke

Adding a spoke takes a bare OpenShift cluster to one that serves models through AI Grid. You prepare
the cluster from your machine, join it to the hub, commit its invite, and then add it in the provider
console. ACM and Argo CD install the rest.

This page uses the spoke name `site-a`. Replace it with the name of your spoke's entry in
`clusters.local.yaml`.

**Field-tested:** the GPU node (about 5 minutes) and the import (about 1 minute) on a fresh
demo.redhat.com cluster (2026-10-21); **Add to AI Grid** as `combined` took a spoke to
**Provisioned** in about 11 minutes (2026-10-21); the GPU operators built the driver on a `g6.xlarge`
node (2026-10-21). On 2026-10-22, an `inference` spoke whose GPU node, nightly
catalog and network were already in place reached **Provisioned** about 4 minutes after the Add, with
the model Ready at about 5.5 minutes and no manual step. Newer than those runs: the commit prompt in
`make spoke-join`, and its `ai-grid/network` label (pending PR 18).

Before you start:

- Complete [Build the hub](3-build-the-hub.md).
- Run `make login` within the last day, so `.kube-cache/site-a` works.
- Check out the branch that Argo CD reads (`git.ref` in `clusters.local.yaml`), with your earlier
  commits pushed. `make spoke-join` commits to it.

## Optional: add a GPU node

The sample model runs on CPU, so a GPU is optional. To serve GPU models, `make spoke-infra` adds an AWS
GPU node to the spoke by cloning one of its worker MachineSets. You can add the node before or after
the other steps on this page.

1. Preview the MachineSet without creating it:

   ```bash
   DRY_RUN=1 make spoke-infra SPOKE=site-a
   ```

   The output is the MachineSet's YAML: by default one `g6.xlarge` (NVIDIA L4) instance, a 120 GB
   volume, the cloned MachineSet's zone, and the taint `nvidia.com/gpu:NoSchedule`.

2. Create it:

   ```bash
   make spoke-infra SPOKE=site-a
   ```

   The script waits for the node. The output ends with the following line:

   ```
   ==> site-a: 1 gpu node(s) Ready
   ```

   This takes 5 to 15 minutes. If AWS refuses the instance, for example for lack of capacity, the
   script stops with AWS's own message; delete the MachineSet with `make spoke-gpu-delete SPOKE=site-a`
   and retry with another zone or type, such as `GPU_AZ=us-east-2b` or `GPU_INSTANCE_TYPE=g6e.xlarge`.

> **Caution:** A GPU instance costs money for as long as it runs. Delete it with
> `make spoke-gpu-delete SPOKE=site-a` when you no longer need it.

To add a CPU worker as well, add `CPU=1`. For every setting, see
[Variables](reference.md#variables).

The node can't run GPU workloads until the NVIDIA GPU operator builds its driver. The operator comes
with the Add, later on this page.

## Warm the nightly catalog

`make spoke-infra` ends by running `make spoke-catalog`. If you skipped the GPU node, run it on its own:

```bash
make spoke-catalog SPOKE=site-a
```

It merges your `quay.io/rhoai` login from `.env` into the spoke's pull secret, creates the nightly
OpenShift AI catalog `rhoai-catalog-dev` exactly as the ACM policy does after the Add, and waits until
the catalog is `READY`. A fresh spoke then doesn't wait about 20 minutes for its operators after the
Add. The command skips itself without `.env` (run `hack/quay-env.sh` first) or on a hosted control
plane, and a rerun reports that the catalog is already `READY`. This step is *not yet run on a fresh
cluster*.

## Join the spoke to the hub

`make spoke-join` does everything on the hub that a spoke needs before you add it in the console.

1. Join the spoke:

   ```bash
   make spoke-join SPOKE=site-a
   ```

   The command does the following:

   1. Imports the cluster into ACM as `ManagedCluster` `site-a`, using its kubeconfig once to install
      ACM's agent. The cluster gets no `ai-grid/enabled` label, so nothing is installed yet.
   2. Labels it `ai-grid/network=true` (pending PR 18), which puts it on the Submariner network with
      the hub now, before the Add. The network is a cluster prerequisite, like the GPU node, so a later
      **Remove from AI Grid** leaves it in place. Without PR 18, the spoke joins the network at the Add
      instead, and leaves it at Remove.
   3. Adds `site-a` under `grid-enrollment.invites` in `apps/grid-enrollment/manifests/values.yaml`.
      Grid enrollment mints an invite token for each entry.
   4. Creates `spokes/clusters/site-a/models/`, a copy of `site-a`'s list of models (the sample model
      `opt-125m`).
   5. Rewrites the portal callbacks of Keycloak's `maas-cli` client: one for every spoke invited in
      git, plus `local-cluster` while it's in AI Grid, less `inference` spokes, which have no portal. It
      reads each spoke's apps domain from the hub's record of its console URL, so it needs no other
      spoke's kubeconfig, and drops any other callback.
   6. Shows the change and asks the following question:

      ```
      Commit and push to BRANCH? [y/N]
      ```

2. Enter `y`. The command commits only the invite and the models directory, and pushes them to the
   branch Argo CD reads. The output ends with the following line:

   ```
   ==> pushed COMMIT to BRANCH: add site-a in the console
   ```

   If you answer `n`, or the command runs without a terminal, commit later with
   `make spoke-commit SPOKE=site-a`. Either way, the commit refuses to run unless your current branch is
   the one Argo CD reads.

3. Wait about a minute for Argo CD to sync. Grid enrollment then mints the spoke's invite token, which
   ACM delivers to the spoke.

> **Note:** If you named your spoke `site-a`, its invite and models directory are already in git from
> the repository's earlier build. Steps 3 to 5 of `make spoke-join` change nothing, there's no
> prompt, and the command prints `site-a: invite and models directory already committed`. Its invite
> token is old, so renew it as described in [Renew an invite](#renew-an-invite) before you add the
> spoke.

### Renew an invite

The invite token lasts 24 hours from the Argo CD sync that minted it. Running `make spoke-join` again
doesn't mint a new one. If more than a day passes before you add the spoke, renew the token on the
hub (*not yet run on a cluster*):

1. Delete the old invite Secret:

   ```bash
   oc --kubeconfig .kube-cache/host delete secret grid-invite-site-a -n grid-enrollment
   ```

2. Ask Argo CD to sync grid-enrollment, which runs the job that mints invites:

   ```bash
   oc --kubeconfig .kube-cache/host patch applications.argoproj.io grid-enrollment -n openshift-gitops --type merge -p '{"operation":{"sync":{}}}'
   ```

3. Check that the Secret exists again:

   ```bash
   oc --kubeconfig .kube-cache/host get secret grid-invite-site-a -n grid-enrollment
   ```

## Add the spoke to AI Grid

1. In the provider console, go to **Clusters**.
2. In the row for `site-a`, open the actions menu and click **Add to AI Grid**.
3. In the **Location** field, enter a location, such as `us-east`. Use the same value later in
   `SPOKE_LOCATIONS`.
4. In the **Role** list, select `combined`.
5. Click **Add to AI Grid**.

The console labels the cluster. ACM then does the following, in about 11 minutes on a fresh spoke:

- Puts the spoke on the Submariner network with the hub, about 4 minutes, unless `make spoke-join`
  already did (pending PR 18).
- Installs the OpenShift AI nightly, MaaS, and the grid, and enrolls the spoke as a grid site, about
  a minute after the network is up.
- Deploys the sample model and the tenant settings.
- On a `combined` or `inference` spoke, installs Node Feature Discovery and the NVIDIA GPU operator
  (Argo CD app `gpu-site-a`). An `ingress` spoke gets neither.
- On a `combined` or `ingress` spoke, sets the MaaS gateway's rate limits to fail closed: if
  Limitador can't be reached, a tenant call gets `500` instead of reaching the model uncounted. The
  gateway also waits up to 2 seconds for Authorino instead of 200 milliseconds.

The cluster shows **Provisioned** when the ACM policy `ai-grid-ready` reports that OpenShift AI is
ready and the grid operator runs.

### Check the GPU

If the spoke has a GPU node, the driver builds on it in about 10 minutes after the GPU operator
installs. Then check that the node offers its GPU:

```bash
oc --kubeconfig .kube-cache/site-a get nodes -l node-role.kubernetes.io/gpu -o jsonpath='{.items[*].status.allocatable.nvidia\.com/gpu}{"\n"}'
```

The output is `1`.

## Commit the spoke's digest

The spoke's provider gateway pins its own site's certificate digest from git. Until you commit it, the
gateway pins 64 zeros and refuses every caller, so tenant calls through this spoke fail.

1. Print the digest. `bootstrap/50-grid-hub-info.sh` reads only the spokes named in `SPOKES`:

   ```bash
   SPOKES=site-a bash bootstrap/50-grid-hub-info.sh
   ```

   The script first waits up to 15 minutes for the hub's own grid identity. The output includes the
   following line:

   ```
   ==> commit this digest: set siteDigests.site-a: "SITE_DIGEST" in apps/ai-grid-spokes/manifests/values.yaml
   ```

   If it says `site-a hasn't enrolled yet`, wait a few minutes and run it again. If it prints nothing
   about `site-a`, git already has the right digest.

2. Set the value in `apps/ai-grid-spokes/manifests/values.yaml`:

   ```yaml
   siteDigests:
     site-a: "SITE_DIGEST"
   ```

3. Commit and push it:

   ```bash
   git add apps/ai-grid-spokes/manifests/values.yaml
   git commit -m "spoke site-a: siteDigests"
   git push
   ```

## Check the spoke

`make verify` runs every check against the hub and the spokes you name. Tell it which spokes exist,
their roles, and their locations:

```bash
export SPOKES="site-a" SPOKE_ROLES="site-a=combined" SPOKE_LOCATIONS="site-a=us-east"
make verify
```

The run takes about 3 minutes. Each check prints `PASS` or `FAIL`; a failure also prints its likely
cause. The checks include a tenant call to the sample model through the spoke's MaaS.

With several spokes, list them all, separated by spaces:

```bash
export SPOKES="site-a site-b" SPOKE_ROLES="site-a=combined site-b=combined" SPOKE_LOCATIONS="site-a=us-east site-b=us-east"
```

Exported variables stay set in this terminal, and other targets read `SPOKES` too, such as
`make spokes` and `make teardown`. Keep the list complete, or unset it with `unset SPOKES SPOKE_ROLES
SPOKE_LOCATIONS`.

## Add a second remote spoke

Repeat this page for each spoke. A second remote spoke has two extra needs:

- **Gossip between remote sites.** Remote sites exchange gossip through an AWS Network Load Balancer,
  which `remoteSwimServiceType: LoadBalancer` in `apps/ai-grid-spokes/manifests/values.yaml` sets up.
  That's the default, so there's nothing to do. With only one remote spoke, you can set `ClusterIP`
  to skip the load balancer.
- **Submariner's ports.** Open IPsec and Submariner's ports between the clusters, because ACM has no
  AWS credentials to do it. For more information, see
  [Submariner doesn't connect](troubleshooting.md#submariner-doesnt-connect).

## What's next

- To sign in as a tenant on this spoke, see [Sign in as a tenant](6-tenant-sign-in.md).
- To serve models from a site that tenants don't call directly, see
  [Serve a model from another site](5-inference-split.md).
