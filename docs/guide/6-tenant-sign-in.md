# Sign in as a tenant

Tenants use the MaaS Portal and the MaaS API on an entry spoke (`combined` or `ingress`). They sign
in through Keycloak on the hub, never through a cluster's OpenShift login. This page explains how
sign-in works and walks you through it with the test users that `make hub` creates.

**Status:** tenant sign-in through the hub's Keycloak and the portal on a `combined` spoke passes
`make verify` on the test environment (2026-10-21). The browser sign-in through an organization's own
identity provider on a newly added spoke is *not yet run* as a hands-on test.

> **Note:** This page describes the broker setup from PR 10, which isn't merged yet. On `main`, realm
> `ai-grid` has no identity providers: the users `alice`, `dave`, and `bob` sign in to it directly, in
> the portal and on the command line, and the groups are `org1-group1` and `org2-group1`.

## How tenant sign-in works

Keycloak on the hub has three realms:

- **`ai-grid`**, the broker. The MaaS gateway and portal on every spoke trust tokens from this realm.
  Each organization (`org1`, `org2`) is a Keycloak organization with an email domain.
- **`org1-idp`** and **`org2-idp`**, which stand in for each organization's own identity provider.

When a tenant enters an email address, realm `ai-grid` sends them to their organization's identity
provider by the address's domain: `org1.example.com` goes to `org1-idp`. After they sign in there,
`ai-grid` issues the token that MaaS reads, with the organization in the claim
`https://redhat.com/maas/orgs`. A tenant sees only their own organization's models.

Realm `ai-grid` also has local users for the command line and for `make verify`, who sign in with a
password grant.

## Test users

`bootstrap/35-keycloak.sh` creates these users and keeps their passwords in Secret
`ai-grid-test-users` in namespace `keycloak` on the hub.

| Sign in as | Realm | Organization | Group | Password key |
|---|---|---|---|---|
| `alice@org1.example.com` | `org1-idp`, through the portal | `org1` | `group1` (subscribed to the sample model) | `org1-idp-alice` |
| `dave@org1.example.com` | `org1-idp`, through the portal | `org1` | none, so no models | `org1-idp-dave` |
| `bob@org2.example.com` | `org2-idp`, through the portal | `org2` | `group1` (no subscription yet, so no models) | `org2-idp-bob` |
| `alice` | `ai-grid`, command line | `org1` | `group1` | `alice` |
| `dave` | `ai-grid`, command line | `org1` | none | `dave` |
| `bob` | `ai-grid`, command line | `org2` | `group1` | `bob` |

To print a password in your terminal, run the following command:

```bash
oc --kubeconfig .kube-cache/host extract secret/ai-grid-test-users -n keycloak --keys=PASSWORD_KEY --to=-
```

Replace `PASSWORD_KEY` with the key from the table.

## Sign in to the portal

1. Find the entry spoke's portal address:

   ```bash
   echo "https://maas.$(oc --kubeconfig .kube-cache/site-a get ingresses.config cluster -o jsonpath='{.spec.domain}')/maas-consumer-portal/"
   ```

2. Open the address in a private browser window. The portal sends you to Keycloak.
3. Enter `alice@org1.example.com` and continue. Keycloak sends you to realm `org1-idp` without
   showing a list of providers.
4. Enter alice's password from the key `org1-idp-alice`.

The portal opens and lists `org1`'s models. Alice's user name in the portal is `org1-idp.` followed by
a fixed ID, because brokered users are named after their identity provider.

To see the difference between users, sign out and sign in as `dave@org1.example.com`, who has no
group and sees no models, and as `bob@org2.example.com`, who belongs to `org2` and sees no models,
because the sample subscription is `org1`'s.

A portal session lasts one hour. Then the tenant signs in again.

## Call a model from the command line

`verify/06-tenant-e2e.sh` does the whole path as `alice`: it gets a token from realm `ai-grid`, creates
a MaaS API key, and calls the sample model through the spoke's MaaS gateway. Run it on its own to see
the result:

```bash
SPOKES=site-a SPOKE_ROLES=site-a=combined bash verify/06-tenant-e2e.sh
```

To run it as another user, set `TENANT_USER`, such as `TENANT_USER=bob`.

## Known limits

- The portal's pages that call the Kubernetes API, such as the user and namespace views, answer `401`
  or `500`, because the cluster doesn't trust Keycloak tokens.
- A valid user with no group gets `500 AUTH_FAILURE` from MaaS rather than `403`.

## What's next

[Remove a spoke or tear down](7-remove-a-spoke.md).
