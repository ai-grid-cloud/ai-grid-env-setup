# Check the prerequisites

Before you start, make sure you have the clusters, accounts, and tools on this page.

## Clusters

You need at least two OpenShift clusters: one hub and one spoke. Each additional spoke is another
cluster.

| Requirement | Why |
|---|---|
| OpenShift 4.20 or later, on AWS | The MaaS and grid charts need 4.20 or later. `make spoke-infra` adds nodes through AWS MachineSets only. |
| A standard control plane, not a hosted control plane | The pull secret is merged into each cluster's `openshift-config/pull-secret`, which a hosted control plane doesn't use. |
| At least 3 schedulable worker nodes | OpenShift AI, ACM, and the model need the room. Control-plane nodes with the worker role count. |
| Default storage class `gp3-csi` | The databases and grid enrollment use persistent volumes. |
| A user with `cluster-admin` and a password login | `make login` signs in with a user name and password, such as `kubeadmin` or `admin`. |
| The same region for all clusters | Tested in one AWS region only. |

The hub also needs the cert-manager operator and the Red Hat build of Keycloak operator installed;
the build doesn't install them. The demo.redhat.com catalog item **OpenShift Container Platform
Cluster (Multi-Cloud)** meets every requirement on this page and comes with both operators.

> **Note:** A second remote spoke needs Submariner's ports open between the clusters (IPsec on UDP
> `500` and `4500`, plus Submariner's own ports). ACM opens them automatically only when it has AWS
> credentials, which this environment doesn't give it. For more information, see
> [Submariner doesn't connect](troubleshooting.md#submariner-doesnt-connect).

## Accounts

- **A GitHub account and a fork of this repository.** Argo CD on the hub reads your fork, and you push
  commits to it during the build. Fork `ai-grid-cloud/ai-grid-env-setup`, then clone your fork:

  ```bash
  git clone https://github.com/GITHUB_USER/ai-grid-env-setup.git
  cd ai-grid-env-setup
  ```

  Replace `GITHUB_USER` with your GitHub user name. The fork must be public, or Argo CD can't read it.

- **A `quay.io/rhoai` login, from a read-only robot account.** Spokes pull the OpenShift AI nightly
  images from `quay.io/rhoai`. The login is copied into every spoke's pull secret, so use a robot
  account that can only read, not your personal login.

## Tools

Install these tools on the machine that you run the scripts from:

| Tool | Used for |
|---|---|
| `oc` | Every cluster operation |
| `make`, `bash`, `git` | Running the targets and committing to your fork |
| `jq` | Reading JSON from the clusters |
| `yq` (mikefarah, version 4) | Reading `clusters.local.yaml` and the values files |
| `curl`, `openssl` | `make login`, Keycloak setup, and the checks |
| `podman` | Your `quay.io` login, which `hack/quay-env.sh` copies |
| `envsubst` (from `gettext`) | `make hub` |
| `helm` | `make spoke-infra` and `make spoke-catalog`, and some checks in `make verify` |
| `perl` | One check in `make verify` |

## Copy your registry login

`hack/quay-env.sh` copies your `quay.io` login from podman into the file `.env` in the repository.
Git ignores `.env`, and the scripts read it when they create the hub's pull-secret Secret.

1. Sign in to `quay.io` with the robot account, saving the login where the script reads it:

   ```bash
   podman login --authfile ~/.config/containers/auth.json quay.io/rhoai
   ```

   Enter the robot account's name and token when podman asks for them. The script reads
   `~/.config/containers/auth.json`, or the file that `REGISTRY_AUTH_FILE` names. Without
   `--authfile`, podman on Linux saves the login elsewhere and the script doesn't find it.

2. Copy the login into `.env`:

   ```bash
   bash hack/quay-env.sh
   ```

   The output is similar to the following:

   ```
   wrote QUAY_USER=ROBOT_NAME and QUAY_TOKEN to /path/to/ai-grid-env-setup/.env
   ```

   The script prints the robot's name only, never the token. You do this once per machine.

If the script says that a credential helper holds the password, see
[`hack/quay-env.sh` can't find the login](troubleshooting.md#hackquay-envsh-cant-find-the-login).

## What's next

[Configure your clusters and fork](2-configure.md).
