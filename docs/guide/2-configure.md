# Configure your clusters and fork

The scripts reach every cluster through one file, `clusters.local.yaml`, and one command,
`make login`. This page sets both up.

**Field-tested:** `make login` with a hub and two spokes (2026-10-21).

## Fill in `clusters.local.yaml`

`clusters.local.yaml` holds each cluster's API URL, user, and password, and the fork and branch that
Argo CD reads. Git ignores it, and `make login` makes it readable only by you.

1. Copy the example file:

   ```bash
   cp clusters.example.yaml clusters.local.yaml
   ```

2. Open `clusters.local.yaml` and fill in the hub:

   ```yaml
   hub:
     server: https://api.HUB_CLUSTER_DOMAIN:6443
     user: HUB_USER
     password: "HUB_PASSWORD"
   ```

   Replace the following:

   - `HUB_CLUSTER_DOMAIN`: the cluster part of the hub's API host name. To find the whole URL, sign
     in to the hub's web console, click your user name, click **Copy login command**, and read the
     `--server` value. `oc whoami --show-server` prints it too.
   - `HUB_USER` and `HUB_PASSWORD`: a user with `cluster-admin`, such as `kubeadmin`.

   Keep the quotes around a password that contains `:`, `#`, `'`, or `"`.

3. Fill in the `git` entry with your fork and the branch that Argo CD reads:

   ```yaml
   git:
     repo: https://github.com/GITHUB_USER/ai-grid-env-setup.git
     ref: main
   ```

   Replace `GITHUB_USER` with your GitHub user name; the example file has `<you>` there, which
   `make login` doesn't check. Every commit that this guide asks you to make goes to this branch. Use
   a branch of your own if you want to keep `main` clean. Push it once with an upstream, so later
   `git push` commands know where to go:

   ```bash
   git push -u origin BRANCH
   ```

   Replace `BRANCH` with the branch in `ref`.

4. Under `spokes`, add an entry for each spoke. The key is the spoke's name in ACM, such as `site-a`.
   It must be lowercase letters, digits, and hyphens.

   ```yaml
   spokes:
     site-a:
       server: https://api.SITE_A_CLUSTER_DOMAIN:6443
       user: SITE_A_USER
       password: "SITE_A_PASSWORD"
   ```

   Delete the example's `site-b` entry if you don't have a second spoke yet. `make login` refuses an
   entry that still holds a `<placeholder>`.

Don't name a spoke `host` or `local-cluster`. `host` means the hub in the scripts, and
`local-cluster` is ACM's name for the hub itself.

## Sign in to every cluster

`make login` signs in to each cluster in `clusters.local.yaml` and writes one kubeconfig per cluster
to `.kube-cache/`. The hub's kubeconfig is `.kube-cache/host`; a spoke's is `.kube-cache/SPOKE_NAME`.
Every other script reads these files, so you don't export `KUBECONFIG`.

1. Open a new terminal in the repository. If you exported `HOST_KUBECONFIG` or `KUBEDIR` in an earlier
   session, unset them; otherwise they win over `.kube-cache/`, and every script prints a `WARNING`
   about it:

   ```bash
   unset HOST_KUBECONFIG KUBEDIR
   ```

2. Sign in:

   ```bash
   make login
   ```

   The output is similar to the following:

   ```
   ==> saved the CA of api.HUB_CLUSTER_DOMAIN:6443: subject=OU=openshift, CN=kube-apiserver-lb-signer sha256 Fingerprint=...
   ==> logged in to host as HUB_USER at https://api.HUB_CLUSTER_DOMAIN:6443 (.../.kube-cache/host)
   ==> saved the CA of api.SITE_A_CLUSTER_DOMAIN:6443: ...
   ==> logged in to site-a as SITE_A_USER at https://api.SITE_A_CLUSTER_DOMAIN:6443 (.../.kube-cache/site-a)
   ```

3. Confirm that a kubeconfig works:

   ```bash
   oc --kubeconfig .kube-cache/site-a whoami
   ```

   The output is the spoke's user name. For `kubeadmin`, OpenShift prints `kube:admin`, and so does
   `make login`.

### How `make login` trusts each cluster

The first time `make login` reaches a cluster, it saves the certificate authority that the cluster's
API presents and prints its fingerprint (*trust on first use*). Later logins accept only that
certificate. The cluster's OAuth server is checked against your system's trusted certificates first;
only a self-signed OAuth certificate is saved the same way. Your password goes to the cluster's OAuth
server through a pipe and is never printed, put on a command line, or stored outside
`clusters.local.yaml`.

> **Caution:** The first login trusts whatever certificate it's shown. Run it from a network that you
> trust. To check, compare the printed fingerprint with the one in the cluster's web console.

### Tokens expire

The tokens last a day. When a command fails with `Unauthorized`, run `make login` again. To sign in to
some clusters only, name them:

```bash
make login NAMES="host site-a"
```

## What's next

[Build the hub](3-build-the-hub.md).
