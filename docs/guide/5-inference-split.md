# Serve a model from another site (inference split)

In an inference split, tenants call one spoke's MaaS, and the grid serves the request on a different
spoke. The spoke that tenants call has the role `combined` or `ingress`; the spoke that serves has the
role `inference` and runs no MaaS. The grid consumer on the entry spoke reaches the inference site's
provider over mutual TLS through Submariner.

> **Note:** The inference split is *not yet run on a cluster*. Every step on this page is designed and
> reviewed, and `make verify` has checks for it, but the end-to-end path hasn't been proven. In
> particular, these parts are unconfirmed: that the grid gateway accepts a Globalnet address as a
> backend and health-checks it, that MaaS passes the serving site's name back to the caller, and that
> the inference site's grid operator reaches its own Globalnet address.

This page uses `site-a` as the entry spoke (`combined`) and `site-b` as the inference site.

Before you start:

- Complete [Add a spoke](4-add-a-spoke.md) for `site-a`, including its digest.
- Make `site-b` a joined spoke: run [Join the spoke to the hub](4-add-a-spoke.md#join-the-spoke-to-the-hub)
  for it. If `site-b` is already in AI Grid with another role, first change its role as described in
  [Change a spoke's role](#change-a-spokes-role).
- Because `site-b` is a second remote site, open Submariner's ports between the clusters. For more
  information, see [Submariner doesn't connect](troubleshooting.md#submariner-doesnt-connect).

Add inference sites before any `ingress` spoke. An `ingress` spoke has no provider of its own, so its
consumer needs at least one `providerGlobalIPs` entry; with none, its values fail the chart's schema
and Argo CD can't render it.

## Add the inference site

1. In the provider console, go to **Clusters**.
2. In the row for `site-b`, open the actions menu and click **Add to AI Grid**.
3. Enter a location, select the role `inference`, and click **Add to AI Grid**.

ACM installs OpenShift AI, the grid operator and provider, and the sample model on `site-b`, but no
MaaS and no consumer. It also creates the Argo CD app `grid-provider-export-site-b`, which exports the
provider over Submariner. Globalnet gives the export an address in `242.0.0.0/8`.

## Wire the sites together

The consumers dial the inference site's provider at its Globalnet address, which changes on every
build, and the provider admits only the consumer sites that git lists. You commit both.

1. Print the values. `bootstrap/50-grid-hub-info.sh` reads the spokes in `SPOKES` and their roles from
   the console's labels:

   ```bash
   SPOKES="site-a site-b" bash bootstrap/50-grid-hub-info.sh
   ```

   The output includes lines similar to the following:

   ```
   ==> commit this digest: set siteDigests.site-b: "SITE_B_DIGEST" in apps/ai-grid-spokes/manifests/values.yaml
   ==> commit this address: set providerGlobalIPs.site-b: "242.X.Y.Z" in apps/ai-grid-spokes/manifests/values.yaml
   ```

   If it says that `site-b` has no Globalnet address yet, wait until the Argo CD app
   `grid-provider-export-site-b` is `Healthy` and run it again.

2. In `apps/ai-grid-spokes/manifests/values.yaml`, set the printed values and list every `combined`
   and `ingress` spoke in `consumerSites`:

   ```yaml
   siteDigests:
     site-a: "SITE_A_DIGEST"
     site-b: "SITE_B_DIGEST"
   providerGlobalIPs:
     site-b: "242.X.Y.Z"
   consumerSites:
     - site-a
   ```

3. Commit and push:

   ```bash
   git add apps/ai-grid-spokes/manifests/values.yaml
   git commit -m "site-b: inference wiring"
   git push
   ```

Argo CD then adds `site-b` as a backend to every other site's consumer, and `site-b`'s provider pins
the digests of the sites in `consumerSites`.

## Check the split

1. Run the checks with both spokes and their roles:

   ```bash
   export SPOKES="site-a site-b" SPOKE_ROLES="site-a=combined site-b=inference" SPOKE_LOCATIONS="site-a=us-east site-b=us-east"
   make verify
   ```

   `verify/05-grid.sh` checks the export's address against git, each consumer's backends, and
   `consumerSites` and `providerGlobalIPs` against the spokes' roles.

2. Optional: to see which site served a tenant request, run the tenant check with the site you
   expect:

   ```bash
   E2E_EXPECT_SITE=site-b bash verify/06-tenant-e2e.sh
   ```

   The check passes when the response's `X-Grid-Provider-Site` header names that site.

   > **Note:** Stopping only the model on `site-a` doesn't move traffic to `site-b`. The consumer
   > checks the health of the provider gateway, not the model, so it keeps choosing `site-a` and the
   > request fails with `502`. A tested way to prove that `site-b` serves a request that enters through
   > `site-a` is pending: either a model served only on the inference site, or the entry spoke's grid
   > provider stopped.

The sample model is plain vLLM, which reports no load, so this shows a request served on another
site, not the grid choosing a site by load. Load-based routing needs a model served with the llm-d
scheduler.

## Change a spoke's role

A spoke's role is set when you add it. To change it, remove the spoke, uninstall its AI Grid software,
and add it again with the new role.

1. Uninstall OpenShift AI, MaaS, and the grid from it with `make spoke-uninstall SPOKE=site-b`, as
   described in [Uninstall a spoke's software](7-remove-a-spoke.md#uninstall-a-spokes-software). Don't
   click **Remove from AI Grid** first; the uninstall removes the spoke itself.
2. Add it again with the new role, as in [Add the inference site](#add-the-inference-site).
3. Check that Argo CD's record of the cluster carries the new role:

   ```bash
   oc --kubeconfig .kube-cache/host get secret site-b-application-manager-cluster-secret -n openshift-gitops --show-labels
   ```

   ACM creates this Secret when it first registers the cluster with Argo CD. The labels include
   `ai-grid/role=inference`. ACM never updates the labels on this Secret after it creates it, so if the old role is still there, set it by hand (*not yet run on a cluster*):

   ```bash
   oc --kubeconfig .kube-cache/host label secret site-b-application-manager-cluster-secret -n openshift-gitops ai-grid/role=inference --overwrite
   ```

## What's next

[Sign in as a tenant](6-tenant-sign-in.md).
