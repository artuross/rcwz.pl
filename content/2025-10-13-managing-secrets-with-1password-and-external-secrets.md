---
title: Managing secrets with 1Password and External Secrets
date: 2025-10-13T18:00:00+02:00
series: ["Talos cluster on Raspberry Pi 5"]
---

So far in this series, I've been putting secrets in 1Password vault and creating secrets with a `bootstrap.nu` script that reads them from 1Password and creates `Secret` resources in the cluster (using `kustomize`).

In this post, I'll add proper secret management to my Kubernetes cluster. I'll be using 2 components:

- [**1Password Connect**](https://developer.1password.com/docs/connect/) for syncing secrets from 1Password and exposing them via an in-cluster REST API,
- [**External Secrets Operator**](https://external-secrets.io/) for syncing secrets from 1Password Connect to Kubernetes secrets.

This setup will allow me to create `ExternalSecret` resources in the cluster that contain references to secrets in 1Password vault. From these, `Secret` resources will be automatically created and kept in sync.

## Prerequisites

As mentioned earlier, my cluster was created [in the first post](/2025-10-04-installing-talos-on-raspberry-pi-5/) of this series. While most steps are generic, I'll be using [`kubesource`](https://github.com/artuross/kubesource) for vendoring manifests into the Git repository.

As a reminder, to install `kubesource`, run:

```nu
go install github.com/artuross/kubesource/cmd/kubesource@latest
```

## Creating namespaces

Just like in the previous post, I'll be deploying each component into its own namespace. To do that, I need to create 2 files, one for each namespace:

```yaml
# kubernetes/cluster/namespaces/resources/core-1password-connect.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: core-1password-connect
```

```yaml
# kubernetes/cluster/namespaces/resources/core-external-secrets.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: core-external-secrets
```

As I already have an existing `kustomization.yaml` file (in `kubernetes/cluster/namespaces`) that includes Cilium namespace, I need to patch it to include the new namespaces:

```diff
  apiVersion: kustomize.config.k8s.io/v1beta1
  kind: Kustomization
  resources:
+   - resources/core-1password-connect.yaml
    - resources/core-cilium.yaml
+   - resources/core-external-secrets.yaml
```

Naturally, all resources could be created without the use of Kustomization file. However, as I'm going to be using ArgoCD in the future anyway, I'm trying to provision resources in a manner similar to how I'll be doing it with ArgoCD. This doesn't apply to the secrets, naturally, which I don't want to commit.
{.sidenote}

For now, you can either apply the namespaces or leave these files as is. They will be created with [`scripts/bootstrap.nu`](https://github.com/artuross/homelab/blob/main/scripts/bootstrap.nu) script created in the previous post.

## Installing 1Password Connect

Installing 1Password Connect is pretty straightforward. I like to start with a fairly minimal and mostly default configuration and patch it later, when necessary. I'll be using the official Helm chart as a base.

### Creating secrets

First, I need to create `1password-credentials.json` file as documented in [1Password docs](https://developer.1password.com/docs/connect/get-started?method=1password-cli#step-1). This can be done with `op` command:

```nu
op connect server create pl-rcwz-homelab --vaults homelab
```

Command above created a new Connect server with name `pl-rcwz-homelab` and connected it to `homelab` vault. It also created a file named `1password-credentials.json` in the working directory. I then saved this file as 1Password item with name `addons.1password.credentials` and deleted it from my disk.

While working on this post, I had some issues while trying to put tokens and files in the same 1Password item. Thus, I recommend to add documents to `Document` types and tokens to `Password` items. External Secrets documentation suggest that this shouldn't matter - perhaps I'm just doing something incorrectly.
{.sidenote}

With that, `scripts/bootstrap.nu` can be patched to create the secret file on disk for `kustomize` to pick it up:

```diff {lineNos=inline lineNoStart=6}
  # create Cilium secrets
  mkdir kubernetes/bootstrap/secrets/cilium
  op read --no-newline "op://homelab/addons.cilium.ca/ca.crt" | base64 --decode | save --force kubernetes/bootstrap/secrets/cilium/ca.crt
  op read --no-newline "op://homelab/addons.cilium.ca/ca.key" | base64 --decode | save --force kubernetes/bootstrap/secrets/cilium/ca.key
  op read --no-newline "op://homelab/addons.cilium.hubble-server-certs/tls.crt" | base64 --decode | save --force kubernetes/bootstrap/secrets/cilium/tls.crt
  op read --no-newline "op://homelab/addons.cilium.hubble-server-certs/tls.key" | base64 --decode | save --force kubernetes/bootstrap/secrets/cilium/tls.key

+ # create 1Password secrets
+ mkdir kubernetes/bootstrap/secrets/1password-connect
+ op read --no-newline "op://homelab/addons.1password.credentials/1password-credentials.json" | jq -c '.' | base64 | save --force kubernetes/bootstrap/secrets/1password-connect/1password-credentials.json

  # create namespaces
  kubectl apply --kustomize kubernetes/cluster/namespaces
```

I noticed that the Connect server is very sensitive to the content of `1password-credentials.json` which I accidently reformated while inspecting the file. To avoid issues, I just used `jq -c '.'` to ensure the file is minified and has no extra whitespace. You may also notice that I used `base64` before saving the file - Kubernetes will encode the content again, but Connect server expects to receive the file in `base64` format.

To complete this part, I need to update `kubernetes/bootstrap/kustomization.yaml` to create the new secret:

```diff
  apiVersion: kustomize.config.k8s.io/v1beta1
  kind: Kustomization
  secretGenerator:
+   # 1password-connect
+   - name: op-credentials
+     namespace: core-1password-connect
+     files:
+       - secrets/1password-connect/1password-credentials.json
+     options:
+       disableNameSuffixHash: true

    # cilium
    - name: cilium-ca
      namespace: core-cilium
      files:
        - secrets/cilium/ca.crt
        - secrets/cilium/ca.key
      options:
        disableNameSuffixHash: true

  # (rest of the file unchanged)
```

The Helm chart expects `op-credentials` by default. It can be customized, but I'm good with that for now.

### Creating base configuration

As mentioned before, I'll be using the official Helm chart as a base. First, create a `kustomization.yaml` file with the following content:

```yaml
# kubernetes/core/1password-connect/_source/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
helmCharts:
  - name: connect
    namespace: core-1password-connect
    repo: https://1password.github.io/connect-helm-charts/
    releaseName: connect
    includeCRDs: false
    version: 2.0.5
    valuesFile: values.yaml
```

Note that I set `includeCRDs` to `false` as this chart comes with their own CRD. I don't need it - I'll use External Secrets for that. The `values.yaml` file (in the same directory) just disables a health check hook:

```yaml
# kubernetes/core/1password-connect/_source/values.yaml
acceptanceTests:
  enabled: false
  healthCheck:
    enabled: false
```

To vendor the Helm chart content, `kubesource` needs a `kubesource.yaml` file (created in the parent directory):

```yaml
# kubernetes/core/1password-connect/kubesource.yaml
apiVersion: kubesource.rcwz.pl/v1alpha1
kind: Config
sourceDir: _source
targets:
  - directory: app/base
```

Then calling `kubesource` from the root of the repository will discover all `kubesource.yaml` files and save the rendered manifests.

```text {hl_Lines=["2-5"]}
❯ kubesource
Processing kubernetes/core/1password-connect
  Source directory: kubernetes/core/1password-connect/_source
  Saving to: kubernetes/core/1password-connect/app/base
  ✓ Successfully processed kubernetes/core/1password-connect
Processing kubernetes/core/cilium
  Source directory: kubernetes/core/cilium/_source
  Saving to: kubernetes/core/cilium/app/base
  ✓ Successfully processed kubernetes/core/cilium
```

After inspecting the content and trying to apply them, I realized that `Deployment` resource must be patched to comply with [Pod Security levels](https://kubernetes.io/docs/concepts/security/pod-security-admission/#pod-security-levels).

```yaml
# kubernetes/core/1password-connect/app/patches/Deployment--core-1password-connect--onepassword-connect.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: onepassword-connect
  namespace: core-1password-connect
spec:
  template:
    spec:
      containers:
        - name: connect-api
          securityContext:
            runAsNonRoot: true
            capabilities:
              drop:
                - ALL
            seccompProfile:
              type: RuntimeDefault
        - name: connect-sync
          securityContext:
            runAsNonRoot: true
            capabilities:
              drop:
                - ALL
            seccompProfile:
              type: RuntimeDefault
```

Finally, following the same pattern as I did for Cilium, a `kustomization.yaml` file stitches everything together:

```yaml
# kubernetes/core/1password-connect/app/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - base
patches:
  - path: patches/Deployment--core-1password-connect--onepassword-connect.yaml
```

The name of the patch file doesn't matter. I just use `[kind]--[namespace]--[name].yaml` convention to make it obvious what is being patched. This is also the same convention that `kubesource` uses for saved files.
{.sidenote}

### Deploying 1Password Connect

To deploy 1Password Connect, I need to create secrets in the cluster and apply the manifests. I already patched `scripts/bootstrap.nu` to create the secrets earlier, so I just need to instruct the bootstrap script to apply the `kustomize` manifest created above:

```diff {lineNos=inline lineNoStart=23}
  # deploy apps
+ kubectl apply --kustomize kubernetes/core/1password-connect/app
  kubectl apply --kustomize kubernetes/core/cilium/app
```

Now, execute `scripts/bootstrap.nu` to create everything. I've inspected logs of both containers in the created `Pod` and everything seems to be working fine.

The Connect server connects to 1Password service and caches the vault items locally. It also exposes the REST API in the cluster. External Secrets Operator, which I will install next, will use this API to create `Secret` resources in the cluster.

## Installing External Secrets Operator

Installing External Secrets Operator is very similar. Again, I'll be using official Helm chart as a base.

### Creating base configuration

Just like before, I need to create the source `kustomization.yaml` file:

```yaml
# kubernetes/core/external-secrets/_source/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
helmCharts:
  - name: external-secrets
    namespace: core-external-secrets
    repo: https://charts.external-secrets.io
    releaseName: external-secrets
    includeCRDs: true
    version: 0.20.2
```

I don't even need the `values.yaml` file this time.

Let's add `kubesource.yaml` for this component as well:

```yaml
# kubernetes/core/external-secrets/kubesource.yaml
apiVersion: kubesource.rcwz.pl/v1alpha1
kind: Config
sourceDir: _source
targets:
  - directory: app/base
```

After rendering it, I can see that `app/base` contains `CustomResourceDefinition`s among other resources. This is not good - I want to manage CRDs separately. Quick patch to `kubesource.yaml` solves this:

```diff
  apiVersion: kubesource.rcwz.pl/v1alpha1
  kind: Config
  sourceDir: _source
  targets:
    - directory: app/base
+     filter:
+       exclude:
+         - kind: CustomResourceDefinition
+   - directory: crds
+     filter:
+       include:
+         - kind: CustomResourceDefinition
```

Run `kubesource` once again and we've separated CRDs from the rest of the resources. `app/base` includes a single `Secret`, but it's empty (it will be automatically populated with TLS certificate once deployed), so there's no need to filter it out.

I also created `kustomization.yaml` in the `app` directory that just points to `base`:

```yaml
# kubernetes/core/external-secrets/app/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - base
```

### Deploying External Secrets Operator

To deploy External Secrets Operator, I need to patch `scripts/bootstrap.nu` once again:

```diff {lineNos=inline lineNoStart=17}
+ # create CRDs
+ kubectl apply --server-side --kustomize kubernetes/core/external-secrets/crds

  # create namespaces
  kubectl apply --kustomize kubernetes/cluster/namespaces

  # create secrets
  kubectl apply --kustomize kubernetes/bootstrap

  # deploy apps
  kubectl apply --kustomize kubernetes/core/1password-connect/app
  kubectl apply --kustomize kubernetes/core/cilium/app
+ kubectl apply --kustomize kubernetes/core/external-secrets/app
```

Notice the `--server-side` flag in line 18. This is hack to avoid Kubernetes adding `kubectl.kubernetes.io/last-applied-configuration` annotation to the CRD, which is too long for most CRDs.

Execute the script now to deploy the operator.

### Creating `ClusterSecretStore`

With both components installed, I still need to integrate them. This is done with `ClusterSecretStore` CRD provided by External Secrets Operator. Luckily, External Secrets documentation provides a good example:

```yaml
# kubernetes/core/external-secrets/app/resources/ClusterSecretStore--1password.yaml
apiVersion: external-secrets.io/v1
kind: ClusterSecretStore
metadata:
  name: 1password
spec:
  provider:
    onepassword:
      connectHost: http://onepassword-connect.core-1password-connect.svc.cluster.local:8080
      vaults:
        homelab: 1
      auth:
        secretRef:
          connectTokenSecretRef:
            name: onepassword-connect-token
            namespace: core-external-secrets
            key: token
```

This basically tells External Secrets Operator to use 1Password Connect server running in `core-1password-connect` namespace as a source of secrets. It also tells it to use `homelab` vault, which was configured when creating the Connect server. Finally, External Secrets Operator is instructed to use a `Secret` named `onepassword-connect-token` in `core-external-secrets` namespace to authenticate with the Connect server.

I don't have that secret yet, so let's create it now.

```nu
op connect token create kubernetes --server pl-rcwz-homelab --vault homelab
```

The output is then saved as as `token` in the newly created 1Password item `addons.1password.tokens`. Once again, I need to patch my trusty `scripts/bootstrap.nu` to create the secret on disk:

```diff {lineNos=inline lineNoStart=13}
  # create 1Password secrets
  mkdir kubernetes/bootstrap/secrets/1password-connect
  op read --no-newline "op://homelab/addons.1password.credentials/1password-credentials.json" | jq -c '.' | base64 | save --force kubernetes/bootstrap/secrets/1password-connect/1password-credentials.json
+ op read --no-newline "op://homelab/addons.1password.tokens/token" | save --force kubernetes/bootstrap/secrets/1password-connect/token
```

which is then picked up by `kubernetes/bootstrap/kustomization.yaml`:

```diff {lineNos=inline lineNoStart=21}
  - name: hubble-server-certs
    namespace: core-cilium
    files:
      - secrets/cilium/ca.crt
      - secrets/cilium/tls.crt
      - secrets/cilium/tls.key
    type: kubernetes.io/tls
    options:
      disableNameSuffixHash: true

+ # external-secrets
+ - name: onepassword-connect-token
+   namespace: core-external-secrets
+   files:
+     - secrets/1password-connect/token
+   options:
+     disableNameSuffixHash: true
```

The `ClusterSecretStore` must be applied as well. I will just add it to `kustomization.yaml` in `kubernetes/core/external-secrets/app`:

```diff
  apiVersion: kustomize.config.k8s.io/v1beta1
  kind: Kustomization
  resources:
    - base
+   - resources/ClusterSecretStore--1password.yaml
```

After running the `bootstrap.nu` script again, the status can be checked with `kubectl` command:

```text
❯ kubectl get css
NAME        AGE   STATUS   CAPABILITIES   READY
1password   78s   Valid    ReadWrite      True
```

## Replacing manually provisioned secrets

So far, all secrets were semi-manually created in the cluster with `kubernetes/bootstrap` and `scripts/bootstrap.nu`. It is time to replace them with `ExternalSecret` resources so they can be automatically synced from 1Password vault.

Starting with the token that External Secrets Operator is already using, I need to create an `ExternalSecret` resource for it. This secret is going to replace the secret `onepassword-connect-token` already in the cluster.

```yaml
# kubernetes/core/external-secrets/app/resources/ExternalSecret--core-external-secrets--onepassword-connect-token.yaml
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: onepassword-connect-token
  namespace: core-external-secrets
spec:
  secretStoreRef:
    kind: ClusterSecretStore
    name: 1password
  target:
    name: onepassword-connect-token
    creationPolicy: Owner
  data:
    - secretKey: token
      remoteRef:
        key: addons.1password.tokens
        property: token
```

Naturally, as External Secrets Operator needs to read this secret to authenticate with 1Password Connect, making a mistake here will results in broken integration. Our `bootstrap.nu` script is very useful here - it first creates the `Secret` (with `kubernetes/bootstrap`) and then creates the `ExternalSecret`, meaning that can be run multiple times to fix any issues.

Before I can run the script, `kustomization.yaml` in `kubernetes/core/external-secrets/app` must be patched again:

```diff
  apiVersion: kustomize.config.k8s.io/v1beta1
  kind: Kustomization
  resources:
    - base
    - resources/ClusterSecretStore--1password.yaml
+   - resources/ExternalSecret--core-external-secrets--onepassword-connect-token.yaml
```

Let's run the script and see if everything works. The secret can be inspected which should reveal a new label:

```text {hl_Lines=[11]}
❯ kubectl -n core-external-secrets get secret onepassword-connect-token -o yaml
apiVersion: v1
data:
  token: <redacted>
kind: Secret
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: <redacted>
  creationTimestamp: "2025-10-09T11:08:14Z"
  labels:
    reconcile.external-secrets.io/managed: "true"
  name: onepassword-connect-token
  namespace: core-external-secrets
  resourceVersion: "125318"
  uid: df394363-319c-439b-8d8a-03ce1f55eb6b
type: Opaque
```

The `ClusterSecretStore` is still ready:

```text
❯ kubectl get css
NAME        AGE   STATUS   CAPABILITIES   READY
1password   18m   Valid    ReadWrite      True
```

`ExternalSecret` status can be checked as well:

```text
❯ kubectl get externalsecret -A
NAMESPACE                NAME                        STORETYPE            STORE       REFRESH INTERVAL   STATUS         READY
core-1password-connect   op-credentials              ClusterSecretStore   1password   1h0m0s             SecretSynced   True
core-external-secrets    onepassword-connect-token   ClusterSecretStore   1password   1h0m0s             SecretSynced   True
```

Let's do the same for 1Password Connect credentials file. The content is fairly similar to the previous one (highlighted lines are different):

```yaml {hl_Lines=[5,6,12,15,17,18]}
# kubernetes/core/1password-connect/app/patches/Deployment--core-1password-connect--onepassword-connect.yaml
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: op-credentials
  namespace: core-1password-connect
spec:
  secretStoreRef:
    kind: ClusterSecretStore
    name: 1password
  target:
    name: op-credentials
    creationPolicy: Owner
  data:
    - secretKey: 1password-credentials.json
      remoteRef:
        key: addons.1password.credentials
        property: 1password-credentials.json
```

Just like before, patch `kustomization.yaml` in `kubernetes/core/1password-connect/app` to include the new resource:

```diff
  apiVersion: kustomize.config.k8s.io/v1beta1
  kind: Kustomization
  resources:
    - base
+   - resources/External-Secret--core-1password-connect--op-credentials.yaml
  patches:
    - path: patches/Deployment--core-1password-connect--onepassword-connect.yaml
```

Apply changes once again (with `bootstrap.nu`) and after that I restarted the `onepassword-connect` pod to make sure it picks up the new secret aaaand... it doesn't work! Remember that our `bootstrap.nu` script creates the secret like this:

```nu
op read --no-newline "op://homelab/addons.1password.credentials/1password-credentials.json" | jq -c '.' | base64 | save --force kubernetes/bootstrap/secrets/1password-connect/1password-credentials.json
```

`base64` is crucial here, as the Connect server expects the file to be encoded in `base64`. External Secrets Operator does not support encoding the secret before saving it to Kubernetes, so I need to store the file already encoded in 1Password. To fix this, I updated `bootstrap.nu` to:

```diff
- op read --no-newline "op://homelab/addons.1password.credentials/1password-credentials.json" | jq -c '.' | base64 | save --force kubernetes/bootstrap/secrets/1password-connect/1password-credentials.json
+ op read --no-newline "op://homelab/addons.1password.credentials/1password-credentials.json" | save --force kubernetes/bootstrap/secrets/1password-connect/1password-credentials.json
```

### Replacing Cilium secrets

If you're following along, Cilium secrets should also be replaced. I previously saved Cilium certificates as `base64`-encoded strings in 1Password. I had to decode and store them as attached files to `Document` type. (I just created a new item with the same name and later deleted the old one.)

[This commit](https://github.com/artuross/homelab/commit/9d1e88b0e9d14ef496927c739783ac566497d6f1) contains the code changes.

## Summary

With my relatively minimal config, secrets can now be automatically created and synced in the cluster. There's one more integration that is crucial to manage my cluster truly the GitOps way - ArgoCD. I'll cover that in the next post.
