# Remove a spoke or tear down

This page covers three levels of removal, from the lightest:

- [Remove a spoke from AI Grid](#remove-a-spoke-from-ai-grid): stop delivering to it; its software
  stays installed.
- [Uninstall a spoke's software](#uninstall-a-spokes-software): remove OpenShift AI, MaaS, and the
  grid from it, so a later Add starts from zero, or [keep its prerequisites](#keep-a-spokes-prerequisites)
  so a later Add installs only AI Grid's layer.
- [Tear down the environment](#tear-down-the-environment): stop delivery to every spoke.

> **Caution:** To uninstall a spoke's software, run `make spoke-uninstall` *instead of* **Remove from
> AI Grid** in the console, not after it. The uninstall first records what Argo CD deployed to the
> spoke and then removes the spoke from AI Grid itself. After a console Remove, that record is gone
> and the uninstall stops.

## Remove a spoke from AI Grid

Use this when you want the spoke to stop receiving changes but keep running what it has.

1. In the provider console, go to **Clusters**.
2. In the row for the spoke, open the actions menu and click **Remove from AI Grid**.
3. Click **Remove**.

The console drops the spoke's `ai-grid/enabled` label. Within about a minute, Argo CD deletes the
spoke's delivery apps and ACM stops enforcing its policies. The software that was installed keeps
running: removal stops delivery, it doesn't uninstall.

Whether a remote spoke stays on the Submariner network depends on its `ai-grid/network` label:

- **With the label** (set by `make spoke-join`, pending PR 18), the spoke stays on the network, so it
  keeps its route to the hub's database and enrollment, and the software that keeps running keeps
  working.
- **Without it,** Remove also takes the spoke off the network: Argo CD deletes the app
  `submariner-site-a`, and ACM removes Submariner from the spoke. A `combined` or `ingress` spoke's
  MaaS then can't reach the hub's database, so tenant calls through it fail until you add it again.
  Until PR 18 merges, label a remote spoke yourself before you remove it:

  ```bash
  oc --kubeconfig .kube-cache/host label managedcluster site-a ai-grid/network=true
  ```

To add the spoke again, click **Add to AI Grid** with the same role and location. Its apps come back
and sync over the software that's still installed, so it needs no new invite or digest.

**Field-tested:** Remove on the hub's own `local-cluster` removed its apps in 68 seconds and left
OpenShift AI, the grid, and the model running (2026-10-20). On 2026-10-22, Remove of the remote
`combined` spoke `site-a` without the label removed its 10 apps in about a minute and took it off
Submariner, which cut its MaaS off the hub's database for about 80 seconds. The Add that followed
brought all 10 apps back and the tunnel up again. With the label set, it stays on the network.

## Uninstall a spoke's software

`make spoke-uninstall` removes AI Grid's software from one spoke: tenant settings, the model, the
grid's objects, MaaS, OpenShift AI and its operators, and the nightly catalog. It also takes a remote
spoke off the Submariner network and removes its `ai-grid/network` label. It keeps what other layers own or what would reboot nodes: the
grid's namespace and site identity, the node pull secret and image mirror, cert-manager, Node Feature
Discovery and the NVIDIA GPU operator, and CRDs.

1. Uninstall:

   ```bash
   make spoke-uninstall SPOKE=site-a
   ```

   At a terminal, the command asks you to type the spoke's name to confirm. Without a terminal, add
   `CONFIRM=site-a`.

   The command saves what each of the spoke's apps deployed to `out/uninstall-site-a.txt`, removes
   the spoke from AI Grid, and then deletes in order. If something doesn't go away, it stops and
   prints the object's finalizers and last condition. Run it again after you fix the cause; it reuses
   the saved list.

   The output ends with the following line:

   ```
   ==> spoke layer removed from site-a. Provision it again: Add to AI Grid in the console (dev shortcut: bash bootstrap/40-spoke.sh)
   ```

2. To use the spoke again, add it in the console, as in
   [Add the spoke to AI Grid](4-add-a-spoke.md#add-the-spoke-to-ai-grid). You can choose a different
   role.

If you already clicked **Remove from AI Grid**, the uninstall stops with a message that the spoke was
already removed. Add the spoke again in the console, wait until its apps are `Synced`, about 2
minutes, and then run `make spoke-uninstall` again.

**Field-tested:** on the hub's own `local-cluster`, 159 seconds (2026-10-21). On the remote
`inference` spoke `site-b`, with `KEEP_PREREQS=1` (pending PR 18), 81 and 116 seconds in two runs
(2026-10-22). A full uninstall of a remote spoke in the current order is *not yet run
on a cluster*.

### Keep a spoke's prerequisites

Pending PR 18. `KEEP_PREREQS=1` removes only AI Grid's layer from a remote spoke and keeps what the
cluster brings as prerequisites, so the next Add installs only that layer:

```bash
KEEP_PREREQS=1 make spoke-uninstall SPOKE=site-b
```

The spoke keeps the following:

- The Submariner network (label `ai-grid/network`).
- The GPU node, Node Feature Discovery, and the NVIDIA GPU operator.
- The nightly catalog `rhoai-catalog-dev`, still `READY`.
- Namespace `grid` with the site's identity, and its exported provider Service, so its Globalnet
  address stays the same.

When you add the spoke again, it comes back at the same address with the same certificate, so
`siteDigests` and `providerGlobalIPs` in git still match, and it needs no new invite or commit.

> **Caution:** The spoke keeps its grid certificate and key, which the other sites still trust while
> its digest is in git, and your `quay.io/rhoai` login in its pull secret. Use `KEEP_PREREQS=1` only
> to add the same cluster again. To retire a cluster, run the uninstall without it, and then
> [forget the spoke](#forget-a-spoke).

**Field-tested:** on `site-b` (`inference`, 2026-10-22). The Add that followed reached
**Provisioned** in about 4 minutes, with the model Ready at about 5.5 minutes, the same Globalnet
address and certificate digest, and no manual step.

### Tenant API keys and other entry spokes

Uninstalling a `combined` or `ingress` spoke deletes its MaaS tenant, and the tenant's cleanup revokes
every tenant API key in the hub's shared MaaS database, including keys that tenants use on other
spokes. So `make spoke-uninstall` refuses, before it changes anything, when another spoke in AI Grid
also serves tenants:

```
ERROR: site-a serves tenants, and so does site-b: uninstalling site-a revokes every tenant API key in the shared maas-db, theirs included. Rerun with REVOKE_TENANT_KEYS=1 to accept that
```

To uninstall anyway, accept the revocation; tenants then create new API keys:

```bash
REVOKE_TENANT_KEYS=1 make spoke-uninstall SPOKE=site-a
```

### How a remote spoke stays connected during the uninstall

MaaS's cleanup jobs on the spoke need the hub's shared database, which the spoke reaches over
Submariner. The uninstall labels the spoke `ai-grid/uninstalling=true`, which keeps it on the network,
deletes the MaaS objects in order (subscriptions and the like, then the tenant configuration, then the
tenant), waits for each, and only then takes the spoke off the network.

## Forget a spoke

To remove a spoke from the environment entirely, after you uninstall its software:

1. Optional: delete its GPU node, which stops the cost:

   ```bash
   make spoke-gpu-delete SPOKE=site-a
   ```

2. Detach it from ACM:

   ```bash
   oc --kubeconfig .kube-cache/host delete managedcluster site-a
   ```

3. Remove its entries from git: `invites.site-a` in `apps/grid-enrollment/manifests/values.yaml`,
   `siteDigests.site-a` and any `providerGlobalIPs.site-a` and `consumerSites` entry in
   `apps/ai-grid-spokes/manifests/values.yaml`, and the directory `spokes/clusters/site-a/` unless
   other spokes still copy from it. Commit and push.
4. Remove its entry from `clusters.local.yaml` and its files from `.kube-cache/`.

These steps are *not yet run on a cluster* as a sequence.

## Tear down the environment

`make teardown` stops delivery to every spoke in `SPOKES` and leaves what's installed running:

```bash
SPOKES="site-a site-b" make teardown
```

It removes the spokes' AI Grid labels, deletes the root ApplicationSet and everything the delivery
apps created, and detaches the remote spokes from ACM. OpenShift AI, the grid, and the hub's services
keep running.

To rebuild delivery:

1. Run `make all`.
2. Run `make spoke-join SPOKE=NAME` for each remote spoke, because the teardown detached them from
   ACM. Their invites and models directories are still in git.
3. Add each spoke in the console again.

**Field-tested:** on a one-cluster environment (2026-10-19). With remote spokes, *not yet run*.

To remove the clusters themselves, delete them where you ordered them, such as demo.redhat.com.
