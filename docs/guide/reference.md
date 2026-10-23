# Reference

## Make targets

| Target | What it does | Runs against |
|---|---|---|
| `make login` | Signs in to every cluster in `clusters.local.yaml` and writes `.kube-cache/`. `NAMES="host site-a"` limits it. | All clusters |
| `make hub` | Installs OpenShift GitOps, points Argo CD at your fork, and adds the hub's Secrets, Keycloak users, and digest. | Hub |
| `make spokes` | Applies the pull secret if `pullSecretMode` is `script`, and rewrites Keycloak's portal callbacks: every spoke invited in git, plus `local-cluster` while it's in AI Grid or in `SPOKES`, less `inference` spokes. | Hub, spokes in `SPOKES` |
| `make all` | `make hub`, then `make spokes`. Adds no cluster to AI Grid. | Hub |
| `make console` | Creates the provider console's OAuth client and Keycloak credentials. | Hub |
| `make spoke-infra SPOKE=NAME` | Adds a GPU node; with `CPU=1`, a CPU worker too; then runs `make spoke-catalog`. | Spoke |
| `make spoke-catalog SPOKE=NAME` | Merges your `quay.io/rhoai` login into the spoke's pull secret and waits for the nightly catalog to be `READY`, before the Add. | Spoke |
| `make spoke-gpu SPOKE=NAME` | Adds a GPU MachineSet. | Spoke |
| `make spoke-gpu-delete SPOKE=NAME` | Deletes it, with its machines and nodes. | Spoke |
| `make spoke-cpu SPOKE=NAME` | Adds a CPU worker MachineSet. | Spoke |
| `make spoke-cpu-delete SPOKE=NAME` | Deletes it. | Spoke |
| `make spoke-join SPOKE=NAME` | Imports the spoke into ACM, labels it `ai-grid/network=true` (pending PR 18), adds its invite and models directory, rewrites the portal callbacks, and offers to commit and push. | Hub, git |
| `make spoke-commit SPOKE=NAME` | Commits and pushes only the spoke's invite and models directory, without asking. | git |
| `make spoke-uninstall SPOKE=NAME` | Removes the spoke from AI Grid and uninstalls OpenShift AI, MaaS, and the grid from it, then takes a remote spoke off the Submariner network. With `KEEP_PREREQS=1` (pending PR 18), keeps the network, catalog, GPU layer, and grid identity. Asks for the name at a terminal; otherwise needs `CONFIRM=NAME`. | Hub, spoke |
| `make verify` | Runs every check in `verify/`. | Hub, spokes in `SPOKES` |
| `make teardown` | Stops delivery to the spokes in `SPOKES` and detaches remote ones; leaves software installed. | Hub |

Scripts without a target:

| Script | What it does |
|---|---|
| `bash hack/quay-env.sh` | Copies your `quay.io` login from podman into `.env`. |
| `bash bootstrap/50-grid-hub-info.sh` | Prints the values to commit: `hubSiteDigest`; for each remote spoke in `SPOKES`, `siteDigests`; and for each `inference` spoke, `providerGlobalIPs`. |
| `bash bootstrap/36-portal-clients.sh` | Rewrites Keycloak's list of portal callbacks, as `make spokes` does. It reads each spoke's apps domain from the hub, so it needs no spoke kubeconfig. |

## Variables

Set these in your shell, or before a command, such as `SPOKES=site-a make verify`. A variable that you
set wins over `clusters.local.yaml`.

| Variable | Default | Meaning |
|---|---|---|
| `SPOKES` | `local-cluster` | The spokes that `make verify`, `make spokes`, `make teardown`, and `50-grid-hub-info.sh` act on, separated by spaces. |
| `SPOKE_ROLES` | `local-cluster=combined` | Each spoke's role, as `NAME=ROLE` pairs. Used by `make verify`. |
| `SPOKE_LOCATIONS` | `local-cluster=us-east` | Each spoke's location, as `NAME=LOCATION` pairs. Used by `make verify`. |
| `GIT_REPO`, `GIT_REF` | `git` in `clusters.local.yaml`, else `ai-grid-cloud/ai-grid-env-setup` `main` | The fork and branch that Argo CD reads. Set them before `make hub`. |
| `KUBEDIR` | `.kube-cache` when `clusters.local.yaml` exists | Where the kubeconfigs are. |
| `HOST_KUBECONFIG` | `KUBEDIR/host` | The hub's kubeconfig. |
| `CLUSTERS_FILE` | `clusters.local.yaml` | The cluster file. |
| `GPU_INSTANCE_TYPE` | `g6.xlarge` | The GPU node's AWS instance type. |
| `GPU_REPLICAS` | `1` | How many GPU nodes. |
| `GPU_AZ` | The cloned MachineSet's zone | The GPU node's availability zone. |
| `GPU_VOLUME_SIZE` | `120` | The GPU node's disk, in GB. |
| `CPU_INSTANCE_TYPE`, `CPU_REPLICAS`, `CPU_AZ`, `CPU_VOLUME_SIZE` | `m6a.4xlarge`, `1`, the clone's zone, `120` | The same, for a CPU worker. |
| `CPU` | Unset | `1` on `make spoke-infra` adds a CPU worker as well as the GPU node. |
| `DRY_RUN` | Unset | `1` prints a MachineSet instead of creating it. |
| `TENANT_USER` | `alice` | The user `verify/06-tenant-e2e.sh` signs in as. |
| `E2E_EXPECT_SITE` | Unset | The site that must serve the tenant check's request. |
| `CONFIRM` | Unset | Names the spoke for `make spoke-uninstall` when there's no terminal to ask. |
| `REVOKE_TENANT_KEYS` | Unset | `1` lets `make spoke-uninstall` remove an entry spoke while another one serves tenants, revoking every tenant API key. |
| `KEEP_PREREQS` | Unset | Pending PR 18. `1` makes `make spoke-uninstall` remove only AI Grid's layer from a remote spoke and keep its network, GPU layer, nightly catalog, and grid identity. See [Keep a spoke's prerequisites](7-remove-a-spoke.md#keep-a-spokes-prerequisites). |

## Files you edit

| File | In git | What you change |
|---|---|---|
| `clusters.local.yaml` | No | Each cluster's API URL, user, and password; your fork and branch. |
| `.env` | No | Written by `hack/quay-env.sh`; edit by hand only if the script can't. |
| `apps/ai-grid-spokes/manifests/values.yaml` | Yes | `hubSiteDigest`, `siteDigests`, `providerGlobalIPs`, and `consumerSites`, from the values the scripts print. |
| `apps/grid-enrollment/manifests/values.yaml` | Yes | `invites`; `make spoke-join` adds entries. |
| `spokes/clusters/NAME/models/` | Yes | The models a spoke serves; `make spoke-join` creates it. |

## Files you don't edit

| Path | What it is |
|---|---|
| `.kube-cache/` | Kubeconfigs and saved certificates from `make login`. Delete a file only when a message tells you to. |
| `out/` | Saved inventories from `98-uninstall-spoke.sh`. |
| `env.sh` | Defaults that every script reads. Override with variables instead. |
| `apps/*/` except the values files named earlier | The hub's apps, which Argo CD applies. |
| `spokes/` except `spokes/clusters/` | The software that ACM and Argo CD deliver to spokes. |
| `bootstrap/`, `hack/`, `verify/` | The scripts. |

## Labels on a ManagedCluster

| Label | Set by | Meaning |
|---|---|---|
| `ai-grid/enabled=true` | The console's **Add to AI Grid** | The cluster is an AI Grid spoke. Its removal stops delivery. |
| `ai-grid/role` | The console | `combined`, `ingress`, or `inference`. |
| `ai-grid/network=true` | `make spoke-join` (pending PR 18), or by hand | Keeps the spoke on the Submariner network with the hub, also after **Remove from AI Grid**. `make spoke-uninstall` removes it unless `KEEP_PREREQS=1`. |
| `ai-grid/uninstalling=true` | `make spoke-uninstall` | Keeps the spoke on the Submariner network while MaaS cleans up. |
| `location` | The console | Where the cluster is, used for model placement. |

Node Feature Discovery and the NVIDIA GPU operator follow the role: every `combined` and `inference`
spoke gets them, and `ingress` spokes don't.
