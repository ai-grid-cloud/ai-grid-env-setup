# Troubleshooting

This page lists the messages that the scripts print when they stop, grouped by step, with the cause and
the fix. Every script is safe to run again after you fix the cause.

`make verify` failures print their own likely cause under each `FAIL` line; start there.

## Any step

### `Unauthorized` or `error: You must be logged in to the server`

Your token from `make login` expired; tokens last a day. Run `make login` again.

### `WARNING: KUBEDIR is set in your shell` or `WARNING: HOST_KUBECONFIG is set in your shell`

A variable exported in your shell overrides the kubeconfigs in `.kube-cache/`, so the scripts use
another file. You can see one warning or both. Unset both, or open a new terminal:

```bash
unset KUBEDIR HOST_KUBECONFIG
```

### `... not created within ...s: is its Argo CD app synced?`

A script waited for an object that an Argo CD app creates. List the apps and find the one that isn't
`Synced` and `Healthy`:

```bash
oc --kubeconfig .kube-cache/host get applications.argoproj.io -n openshift-gitops
```

Common causes are a commit that Argo CD can't render, and a branch that you didn't push. Argo CD reads
the fork and branch in `clusters.local.yaml`, not your working tree.

## Registry login (`hack/quay-env.sh`)

### `no podman login found; run: podman login quay.io`

Run `podman login quay.io/rhoai` with the robot account, and then run the script again.

### `hack/quay-env.sh` can't find the login

The message is `... has no stored quay.io password (a credential helper holds it)`. A credential
helper, such as the macOS keychain, holds the password instead of podman's auth file. Sign in again
with an explicit auth file, as the message says:

```bash
podman login --authfile ~/.config/containers/auth.json quay.io
```

### `the login contains a single quote; set .env by hand`

Write `.env` yourself with two lines, `QUAY_USER='ROBOT_NAME'` and `QUAY_TOKEN='TOKEN'`, and run
`chmod 600 .env`.

## Sign-in (`make login`)

| Message | Cause | Fix |
|---|---|---|
| `no .../clusters.local.yaml` | You haven't created the file. | Run `cp clusters.example.yaml clusters.local.yaml` and fill it in. |
| `NAME: still has a <placeholder> in ...` | An entry still holds the example's values. | Fill it in, or delete the entry. |
| `NAME: needs server (https://<host>[:port]) and user in ...` | The `server` isn't a plain `https://` URL, or `user` is empty. | Use `https://api.CLUSTER_DOMAIN:6443`, with no path. |
| `NAME: no password in ...` | `password` is empty. | Fill it in, quoted. |
| `NAME: no TLS certificate from HOST:PORT` | The server's address is wrong, or the cluster is down or unreachable. | Check the `server` value and that the cluster runs. |
| `the saved CA for NAME no longer matches SERVER (rebuilt cluster?)` | The cluster presents a different certificate than at the first login. The usual reason is a new cluster under the same name. | If you rebuilt the cluster, delete the file the message names and run `make login` again. If you didn't, stop and find out why the certificate changed. |
| `NAME: the OAuth server's certificate no longer verifies; it did before` | The OAuth server's certificate verified through your system's trust store before and doesn't now. This can mean someone is intercepting the connection. | Check the certificate in a browser. Only if it's legitimately changed, delete the `.oauth-system-trust` file the message names. |
| `NAME: login as USER at SERVER failed: check the user and password` | Wrong user or password, or the user can't sign in with a password. | Check them in the cluster's web console. |
| `NAME: can't reach SERVER (curl exit N)` | A network error other than a certificate problem. | Check the address and your network. |
| `NAME: not in ...` | You named a cluster in `NAMES` that the file doesn't have. | Check the spelling; the hub is `host`. |

## Hub (`make hub`, `make spokes`, `make console`)

### `missing .env: run hack/quay-env.sh` or `QUAY_USER or QUAY_TOKEN empty`

See [Copy your registry login](1-prerequisites.md#copy-your-registry-login).

### `Keycloak admin login failed`

Keycloak isn't serving yet, or its admin Secret `maas-keycloak-initial-admin` changed. Wait for the
Argo CD app `keycloak` to be `Healthy` and run the script again.

### `realm ai-grid predates the broker ...`

The hub's Keycloak realm is from before organizations had their own identity providers, and the
Keycloak operator never re-imports an existing realm. Follow the steps in the message; the
[runbook](../runbook.md#upgrading-realm-ai-grid-to-the-broker) has them in full. Deleting the realm
drops its users and the console backend's user, which the listed scripts create again.

### `run bootstrap/35-keycloak.sh first`

`make spokes` or `make console` ran before Keycloak was set up. Run `make hub` to the end first.

### `route ai-grid-provider-console in ai-grid missing: is Argo CD app console synced?`

Wait for the Argo CD app `console` to be `Healthy`, and then run `make console` again.

## Spoke preparation (`make spoke-infra`, `make spoke-join`)

| Message | Cause | Fix |
|---|---|---|
| `no kubeconfig for SPOKE: expected ...` | `SPOKE` isn't in `clusters.local.yaml`, or you haven't run `make login`. | Add the spoke to the file and run `make login`. |
| `SPOKE is on PLATFORM: only AWS is supported` | `make spoke-infra` clones AWS MachineSets only. | Add the node another way. |
| `SPOKE has no worker MachineSet in ZONE to clone` | No worker MachineSet in the zone you set with `GPU_AZ` or `CPU_AZ`. | Unset the zone or pick one with workers. |
| `machine NAME failed: REASON: MESSAGE` | AWS refused the instance, for example `InsufficientInstanceCapacity`. | Run `make spoke-gpu-delete SPOKE=SPOKE`, then retry with `GPU_AZ` or `GPU_INSTANCE_TYPE`. |
| `SPOKE: timed out after 20 min` | The node didn't become Ready. | Look at the Machine with the command the message prints. |
| `SPOKE not available after 15m; deleted its auto-import-secret` | ACM's agent on the spoke didn't connect to the hub. The script deleted the Secret that held the spoke's admin kubeconfig. | Check that the spoke can reach the hub's API, then run `make spoke-join` again. |
| `couldn't add the invite to ...` | The script couldn't edit `apps/grid-enrollment/manifests/values.yaml`, for example because `invites:` is empty. | Add `SPOKE: { network: grid }` under `grid-enrollment.invites` by hand. |
| `local-cluster is the hub itself: nothing to join` | The hub is already in ACM. | To use the hub as a spoke, add `local-cluster` in the console. |

### `on branch BRANCH, but Argo CD reads GIT_REF`

`make spoke-join` and `make spoke-commit` commit only to the branch Argo CD reads. Check out that
branch, or set `git.ref` in `clusters.local.yaml` to the branch you're on, and run
`make spoke-commit SPOKE=SPOKE`.

### `git push` or `git pull --ff-only` fails with `Diverging branches`

Your local branch tracks a different remote branch than the one Argo CD reads. Point it at the right
one:

```bash
git branch --set-upstream-to=origin/GIT_REF
```

Replace `GIT_REF` with the branch in `clusters.local.yaml`, and then pull and push again.

## Adding to AI Grid

### The console shows **Provisioning** for a long time

The first Add takes about 11 minutes. If it takes much longer, check the spoke's apps on the hub:

```bash
oc --kubeconfig .kube-cache/host get applications.argoproj.io -n openshift-gitops | grep SPOKE
```

Then run `make verify` with the spoke in `SPOKES`; its failures name the stuck piece. A common cause is
an invite token older than 24 hours; see [Renew an invite](4-add-a-spoke.md#renew-an-invite).

### A new spoke waits about 20 minutes before it enrolls

For about 15 to 25 minutes after a fresh spoke's first Add, its operator subscriptions, such as
Submariner's and the Cluster Observability Operator's, show `ResolutionFailed` with a message similar
to the following:

```
error using catalogsource openshift-marketplace/rhoai-catalog-dev ... connection refused
```

Submariner's resources then fail for lack of its CRD, so the spoke's tunnel and grid enrollment wait.

The nightly OpenShift AI catalog is available to every namespace, and its pod takes about 6 minutes to
pull and start on a new node. While any such catalog is unreachable, OLM fails resolution for every
subscription on the cluster, and then waits out its retry backoff.

To avoid the wait, run `make spoke-catalog SPOKE=SPOKE` (part of `make spoke-infra`) before you add
the spoke. Otherwise, no action is needed: the operators install, ACM applies Submariner again, and the
tunnel and enrollment follow. If the spoke still hasn't enrolled after 30 minutes, check its apps and run
`make verify` with the spoke in `SPOKES`.

### Submariner doesn't connect

The Argo CD app `submariner-SPOKE` shows `Degraded`, or `verify/09-submariner.sh` fails.

- **With only one cluster on the network,** `Degraded` with `There are no connections on gateways` is
  expected until a second cluster joins.
- **With a second remote spoke,** the remote spokes can't reach each other's Submariner ports. (The
  hub and the first spoke connect without help.) ACM opens the ports only when it has AWS credentials,
  and this environment gives it none. Open IPsec (UDP `500` and `4500`) and Submariner's ports between
  the clusters' gateway nodes in their AWS security groups.

### `SPOKE hasn't enrolled yet: rerun after adding it to AI Grid`

`bootstrap/50-grid-hub-info.sh` found no grid identity on the spoke yet. Enrollment follows the
Submariner tunnel by about a minute; wait and run it again.

### `can't read SPOKE's GridNetwork: ...`

The script couldn't read the spoke for a reason other than the grid not being installed, such as an
expired token. Run `make login` and try again.

### Tenant calls through a spoke fail after the Add

- **Every call fails right after a first Add.** You haven't committed `siteDigests.SPOKE` yet, so the
  spoke's provider gateway pins zeros and refuses every caller. See
  [Commit the spoke's digest](4-add-a-spoke.md#commit-the-spokes-digest).
- **Calls get `500`.** The MaaS gateway's rate limits fail closed: when Limitador can't be reached, a
  call gets `500` instead of reaching the model uncounted. Check Limitador and its Redis with
  `verify/10-limitador-redis.sh`.

### Tenant calls fail after **Remove from AI Grid**, or after an Add that follows one

Without the `ai-grid/network` label, Remove took the spoke off Submariner, so its MaaS can't reach the
hub's database until the next Add brings the tunnel back, about a minute after the Add. Set the label
so it doesn't happen again (`make spoke-join` sets it, pending PR 18):

```bash
oc --kubeconfig .kube-cache/host label managedcluster SPOKE ai-grid/network=true
```

### A model shows an in-cluster address, or the portal lists no models

MaaS keeps a model's address from before the gateway got its public host name, and the portal caches
a failed lookup. Restart both on the spoke:

```bash
oc --kubeconfig .kube-cache/SPOKE get pods -n redhat-ods-applications -o name | grep -E 'maas-(controller|ui)' \
  | xargs oc --kubeconfig .kube-cache/SPOKE delete -n redhat-ods-applications
```

## Removal (`make spoke-uninstall`)

| Message | Cause | Fix |
|---|---|---|
| `this deletes OpenShift AI, MaaS and the grid from SPOKE; confirm with: ...` | A guard against uninstalling the wrong cluster. | Type the name when asked, or add `CONFIRM=SPOKE`. |
| `still present after Ns: OBJECT (finalizers and last condition above)` | An object kept a finalizer whose controller is gone or stuck. | Read the printed finalizers and condition, fix or remove the finalizer, and run the script again. |
| `SPOKE serves tenants, and so does OTHER: uninstalling SPOKE revokes every tenant API key ...` | Uninstalling would revoke tenant keys that another entry spoke uses. | See [Tenant API keys and other entry spokes](7-remove-a-spoke.md#tenant-api-keys-and-other-entry-spokes). |
| `SPOKE is in AI Grid but its apps aren't on the hub yet` | The spoke was added moments ago. | Wait until its apps are `Synced`, about 2 minutes, and run it again. |
| `SPOKE was already removed in the console, so its app list is gone ...` | You clicked **Remove from AI Grid** before the uninstall, so it can't see what Argo CD deployed. | Add the spoke again in the console, wait until its apps are `Synced` (about 2 minutes), and run `make spoke-uninstall` again. |
| `Placement ai-grid-network still selects SPOKE after 120s` | The spoke still has the `ai-grid/enabled` label. | Remove it in the console, then run the script again. |
| `the server doesn't have a resource type "externalmodels"`, then `can't list the MaaS objects` | An `inference` spoke has no MaaS CRDs, and the uninstall before PR 18 expected them. | Use PR 18's uninstall, which skips a kind that never arrived, or remove the spoke's `ai-grid/uninstalling` label as the message says. |

### The uninstall of a remote spoke stops at namespace `ai-tenants`

The message is `still present after 600s: ns ai-tenants`, or a wait on a MaaS subscription, tenant
configuration, or tenant. MaaS's cleanup jobs on the spoke revoke API keys in the hub's database over
Submariner, and the uninstall keeps the spoke on the network until they finish. If the tunnel itself is
down, they can't: run `verify/09-submariner.sh` with the spoke in `SPOKES`, and fix the tunnel as in
[Submariner doesn't connect](#submariner-doesnt-connect). Then run the uninstall again.

### A removed spoke's `submariner-SPOKE` app stays deleting

If the spoke can't be reached, ACM can't finish removing Submariner from it. On the hub, delete the
add-on without waiting, and if the app still stays, remove its finalizer:

```bash
oc --kubeconfig .kube-cache/host delete managedclusteraddon submariner -n SPOKE --wait=false
oc --kubeconfig .kube-cache/host patch applications.argoproj.io submariner-SPOKE -n openshift-gitops --type json -p '[{"op":"remove","path":"/metadata/finalizers"}]'
```
