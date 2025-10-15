---
title: Setting up ArgoCD for continuous deployment
date: 2025-10-15T20:00:00+02:00
series: ["Talos cluster on Raspberry Pi 5"]
---

In the [previous blog post in this series](2025-10-13-managing-secrets-with-1password-and-external-secrets/), I added secret management with [External Secrets Operator](https://external-secrets.io/latest/) and [1Password Connect](https://developer.1password.com/docs/connect/). This was a necessary step before introducing [ArgoCD](https://argo-cd.readthedocs.io/en/stable/), which will automate the deployment of applications to my Kubernetes cluster. Without ESO, I would either have to commit secrets in Git (not ideal, unless you plan to use [Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets) or [SOPS](https://github.com/getsops/sops)) or to manually create secrets in the cluster.

Today, I will set up ArgoCD and migrate all components deployed so far to it. ArgoCD is quite powerful, but I'm going to keep it simple for now, since I wouldn't be able to use all the features just yet.

You can find all code in [my Git repository](https://github.com/artuross/homelab), with all changes in this post [in these commits](https://github.com/artuross/homelab/compare/9d1e88b0e9d14ef496927c739783ac566497d6f1...46cb90fbc82ac7936565ce291b12091900c1ee62).

## Prerequisites

This post builds on top of the previous posts in the series. Steps described here should work on most clusters, but in case of issues, please refer to the previous posts.

## Preparing namespace

As usual, I will start by creating a namespace for ArgoCD. Following the pattern established in previous posts, I will create a namespace `core-argocd` and create it with `kustomization.yaml` along with all other namespaces.

```yaml
# kubernetes/cluster/namespaces/resources/core-argocd.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: core-argocd
```

Then add it to `kubernetes/cluster/namespaces/kustomization.yaml`:

```diff
  apiVersion: kustomize.config.k8s.io/v1beta1
  kind: Kustomization
  resources:
    - resources/core-1password-connect.yaml
+   - resources/core-argocd.yaml
    - resources/core-cilium.yaml
    - resources/core-external-secrets.yaml
```

## Adding base ArgoCD config

As the base, I'll use [ArgoCD's Helm chart](https://github.com/argoproj/argo-helm/tree/main/charts/argo-cd) `v8.5.10` from the official Helm repository.

```yaml
# kubernetes/core/argocd/_source/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
helmCharts:
  - name: argo-cd
    namespace: core-argocd
    repo: https://argoproj.github.io/argo-helm
    releaseName: argocd
    includeCRDs: true
    version: 8.5.10
    valuesFile: values.yaml
```

I also added an empty `values.yaml` in the same directory. To render manifests with [`kubesource`](https://github.com/artuross/kubesource), I need to add a `kubesource.yaml` file. I already know that ArgoCD needs to create CRDs, so I will create a separate target for them.

```yaml
# kubernetes/core/argocd/kubesource.yaml
apiVersion: kubesource.rcwz.pl/v1alpha1
kind: Config
sourceDir: _source
targets:
  - directory: app/base
    filter:
      exclude:
        - kind: CustomResourceDefinition
  - directory: crds
    filter:
      include:
        - kind: CustomResourceDefinition
```

I've also added `kustomization.yaml` to the `app` directory:

```yaml
# kubernetes/core/argocd/app/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - base
```

## Customizing ArgoCD

After rendering manifests and inspecting output, I need to make a few adjustments:

1. **Disable Dex Server and authentication.**

    As I'm going to be the only user right now, I can just disable the authentication entirely. Later, I plan to use something else than Dex, which is quite basic anyway.

2. **Disable Notifications Controller.**

    ArgoCD Notifications are a more advanced feature. I don't need them right now and disabling this component will simplify the setup.

3. **Disable Redis Secret Init job.**

    When ArgoCD is first installed, it creates a job to initialize a Redis password. This job just creates a new secret, however the Helm chart creates a bunch of resources to manage permissions. Since I already have ESO managing secrets, I can just create the secret in 1Password and disable this job.

4. **Remove the `Secret`.**

    Similarly to the Redis password, when ArgoCD first starts, it populates this secret with some required fields. Again, I will just move everything to 1Password to have repeatable environment.

Let's address them all now by modifiying `values.yaml`:

```yaml {lineNos=inline}
# kubernetes/core/argocd/_source/values.yaml
configs:
  params:
    server.disable.auth: true

  secret:
    createSecret: false

dex:
  enabled: false

notifications:
  enabled: false

redisSecretInit:
  enabled: false

```

I think the changes are self-explanatory, but for clarity:

- lines **4** and **10** disable authentication and Dex Server,
- line **7** disables creation of the `Secret` resource,
- line **13** disables the Notifications Controller,
- line **16** disables the Redis Secret Init job.

## Deploying ArgoCD

With the initial config prepared, I can deploy ArgoCD to the cluster. As a reminder, I've removed Redis password job and the `Secret` resource, so I expect the application to simply not start. Let's deploy everything first and see what happens:

```nu
kubectl apply --kustomize kubernetes/cluster/namespaces
kubectl apply --server-side --kustomize kubernetes/core/argocd/crds
kubectl apply --kustomize kubernetes/core/argocd/app
```

After checking the status, I can see the issue, just like anticipated:

```text
❯ k get pods -n core-argocd
NAME                                                READY   STATUS                       RESTARTS      AGE
argocd-application-controller-0                     1/1     Running                      0             3m57s
argocd-applicationset-controller-66cbcfb664-7drcj   1/1     Running                      0             3m57s
argocd-redis-794bcb9584-s742p                       0/1     CreateContainerConfigError   0             3m57s
argocd-repo-server-6588d49677-qqm6v                 1/1     Running                      0             3m57s
argocd-server-6b6c67d9c5-qbh8q                      0/1     CrashLoopBackOff             5 (58s ago)   3m57s
```

### Fixing Redis

After inspecting details and a short search, it turns out I need to create `argocd-redis` secret with a key `auth` containing the password. I will use ESO to create this secret from 1Password.

First, I need to add a new item to 1Password vault. I just added a new `Password` item with the name `addons.argocd.redis` and a random password generated by 1Password.

This resource must be created in the cluster, so a new `ExternalSecret` resource just references it:

```yaml
# kubernetes/core/argocd/app/resources/External-Secret--core-argocd--argocd-redis.yaml
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: argocd-redis
  namespace: core-argocd
spec:
  secretStoreRef:
    kind: ClusterSecretStore
    name: 1password
  target:
    name: argocd-redis
    creationPolicy: Owner
  data:
    - secretKey: auth
      remoteRef:
        key: addons.argocd.redis
        property: password
```

then I just need to add it to `kustomization.yaml` (in `kubernetes/core/argocd/app`):

```diff
  apiVersion: kustomize.config.k8s.io/v1beta1
  kind: Kustomization
  resources:
    - base
+   - resources/External-Secret--core-argocd--argocd-redis.yaml
```

One apply later, I can see the secret created and Redis pod has started.

### Fixing ArgoCD Server

The last issue is with the ArgoCD server itself. It has a couple of required fields. ArgoCD docs have a good [example](https://argo-cd.readthedocs.io/en/stable/operator-manual/argocd-secret-yaml/) of the content of the secret.

I was a bit lazy and just temporarily re-enabled the creation of the secret in `values.yaml` to get the certificates. The certificates landed in a new 1Password item `addons.argocd.certificates` (type: `Document`).

I added one more 1Password item with `addons.argocd.server` and a random password saved in the `secretKey` field. This will be the `server.secretkey` property in the `Secret` file.

All of that is then put in another `ExternalSecret` resource:

```yaml
# kubernetes/core/argocd/app/resources/External-Secret--core-argocd--argocd-secret.yaml
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: argocd-secret
  namespace: core-argocd
spec:
  secretStoreRef:
    kind: ClusterSecretStore
    name: 1password
  target:
    name: argocd-secret
    creationPolicy: Owner
  data:
    - secretKey: server.secretkey
      remoteRef:
        key: addons.argocd.server
        property: secretKey

    - secretKey: tls.crt
      remoteRef:
        key: addons.argocd.certificates
        property: tls.crt

    - secretKey: tls.key
      remoteRef:
        key: addons.argocd.certificates
        property: tls.key
```

and added to the same `kustomization.yaml` file as before:

```diff
  apiVersion: kustomize.config.k8s.io/v1beta1
  kind: Kustomization
  resources:
    - base
    - resources/External-Secret--core-argocd--argocd-redis.yaml
+   - resources/External-Secret--core-argocd--argocd-secret.yaml
```

Apply again, and voila! All pods are running:

```text
❯ k get pods -n core-argocd
NAME                                                READY   STATUS    RESTARTS   AGE
argocd-application-controller-0                     1/1     Running   0          32m
argocd-applicationset-controller-66cbcfb664-7drcj   1/1     Running   0          32m
argocd-redis-794bcb9584-s742p                       1/1     Running   0          32m
argocd-repo-server-6588d49677-qqm6v                 1/1     Running   0          32m
argocd-server-6b6c67d9c5-2cwt8                      1/1     Running   0          105s
```

## Verifying ArgoCD

To access the ArgoCD UI, I need to set up port forwarding:

```nu
kubectl port-forward -n core-argocd service/argocd-server 8080:80
```

Then I can access the UI at `http://localhost:8080`. This is annoying - ArgoCD redirects me to HTTPS. I don't want to deal with it right now. A quick patch to `values.yaml` disables TLS:

```diff
  configs:
    params:
      server.disable.auth: true
+     server.insecure: true

    secret:
      createSecret: false
```

Reapplying the manifests (you will need to run `kubesource` first) fixes the issue.

## Migrating existing components to ArgoCD

To make ArgoCD manage my existing components, I only really need 4 resources:

1. An `Application` to sync `kubernetes/cluster/namespaces`.
2. An `ApplicationSet` to sync components in `kubernetes/core/*/app`.
3. An `ApplicationSet` to sync components in `kubernetes/core/*/crds`.
4. An `Application` to sync all 3 resources above since I want to manage those with ArgoCD as well.

I should say that 2. and 3. could easily be combined into one `ApplicationSet`, but I prefer to keep them separated.

I will also not focus on the more advanced features of ArgoCD, such as sync waves, etc. I want to keep it simple for now.

### Bootstrapping ArgoCD

Let's start with the last `Application` resource:

```yaml
# kubernetes/bootstrap/resources/Application--core-argocd--bootstrap.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: bootstrap
  namespace: core-argocd
spec:
  project: default
  source:
    repoURL: git@github.com:artuross/homelab
    targetRevision: HEAD
    path: kubernetes/cluster/apps
  destination:
    name: in-cluster
    namespace: core-argocd
  syncPolicy:
    automated:
      prune: false
      selfHeal: true
      allowEmpty: true
```

I've put this in `kubernetes/bootstrap`, because it's pretty much the only resource **not** managed by ArgoCD.

It will create a new application that will sync everything in `kubernetes/cluster/apps` (doesn't exist yet). The repo URL is set to my repo, but you should set it to your own.

I also want to add it to `kustomization.yaml` in the parent directory. In case I ever need to recreate the cluster, I want it to bootstrap automatically:

```diff
  apiVersion: kustomize.config.k8s.io/v1beta1
  kind: Kustomization
+ resources:
+   - Application--core-argocd--bootstrap.yaml

  secretGenerator:
    # the rest of the document ommited
```

`bootstrap` application expects `kubernetes/cluster/apps` to exist. For now, empty `kustomization.yaml` will be enough:

```yaml
# kubernetes/cluster/apps/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources: []
```

Finally, the `Application` can be just applied:

```nu
kubectl apply --filename kubernetes/bootstrap/resources/Application--core-argocd--bootstrap.yaml
```

After checking the state in ArgoCD's UI, I can see an error

```text
Failed to load target state: failed to generate manifest for source 1 of 1: rpc error: code = Unknown desc = failed to list refs: error creating SSH agent: "SSH agent requested but SSH_AUTH_SOCK not-specified"
```

This just says that ArgoCD cannot access my Git repo (which is public). To fix this, I need to add my SSH key to ArgoCD.

I added a new `services.github.ssh` with `SSH Key` type to 1Password, then copied the public key to my GitHub account.

Then I created another `ExternalSecret` resource:

```yaml
# kubernetes/core/argocd/app/resources/External-Secret--core-argocd--github-credentials.yaml
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: github-credentials
  namespace: core-argocd
spec:
  secretStoreRef:
    name: 1password
    kind: ClusterSecretStore
  target:
    name: github-credentials
    creationPolicy: Owner
    template:
      engineVersion: v2
      mergePolicy: Merge
      metadata:
        labels:
          argocd.argoproj.io/secret-type: repository
      data:
        type: git
        url: git@github.com:artuross/homelab
  data:
    - secretKey: sshPrivateKey
      remoteRef:
        key: services.github.ssh
        property: private key
```

This one is a bit different than the previous ones. Not every part here is the secret, some of it is just static data. The important part is the `argocd.argoproj.io/secret-type: repository` label, which tells ArgoCD that this secret contains repository credentials.

Final patch to ArgoCD's `kustomization.yaml`:

```diff
  apiVersion: kustomize.config.k8s.io/v1beta1
  kind: Kustomization
  resources:
    - base
    - resources/External-Secret--core-argocd--argocd-redis.yaml
    - resources/External-Secret--core-argocd--argocd-secret.yaml
+   - resources/External-Secret--core-argocd--github-credentials.yaml
```

After applying, the repository should be visible in `Repository` → `Settings` in the UI. I also had to remove the `bootstrap` application and re-create it, since ArgoCD was stuck in a weird sync state.

Checking again, everything looks good.

### Adding `kubernetes/cluster/apps` resources

`Application` to sync `kubernetes/cluster/namespaces` is very similar to the `bootstrap` one:

```yaml
# kubernetes/cluster/apps/resources/Application--core-argocd--cluster-namespaces.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: cluster-namespaces
  namespace: core-argocd
spec:
  project: default
  source:
    repoURL: git@github.com:artuross/homelab
    targetRevision: HEAD
    path: kubernetes/cluster/namespaces
  destination:
    name: in-cluster
    namespace: core-argocd
  syncPolicy:
    automated:
      prune: false
      selfHeal: true
      allowEmpty: true
```

`ApplicationSet` to sync CRDs is a bit different. `ApplicationSet` generates `Application` resources based on a template and discovered resources. It's a bit similar to how e.g. `Deployment` has a template for `Pod`. I've highlighted the most importants parts, but in a gist, discovered directory paths are stored in the `.path` variable, which is then used both to generate the name of the `Application` and to set the `path` field in the `source` section.

```yaml {hl_Lines=[16,19,25]}
# kubernetes/cluster/apps/resources/ApplicationSet--core-argocd--core-crds.yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: core-crds
  namespace: core-argocd
spec:
  goTemplate: true
  goTemplateOptions:
    - missingkey=error
  generators:
    - git:
        repoURL: git@github.com:artuross/homelab
        revision: HEAD
        directories:
          - path: kubernetes/core/*/crds
  template:
    metadata:
      name: core-{{ index .path.segments 2 }}-crds
    spec:
      project: default
      source:
        repoURL: git@github.com:artuross/homelab
        targetRevision: HEAD
        path: "{{ .path.path }}"
      destination:
        name: in-cluster
        namespace: core-argocd
      syncPolicy:
        automated:
          prune: false
          selfHeal: true
          allowEmpty: true
        syncOptions:
          - ServerSideApply=true
```

You may also notice `ServerSideApply=true` in `syncOptions`. This is the same hack that I used earlier with `kubectl apply --server-side`. It makes ArgoCD use server-side to avoid issues with too long `kubectl.kubernetes.io/last-applied-configuration`.

Resource for `kubernetes/core/*/app` is almost identical (highlighted are the differences). `ServerSideApply=true` is not needed here.

```yaml {hl_Lines=[5,16,19]}
# kubernetes/cluster/apps/resources/ApplicationSet--core-argocd--core-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: core-app
  namespace: core-argocd
spec:
  goTemplate: true
  goTemplateOptions:
    - missingkey=error
  generators:
    - git:
        repoURL: git@github.com:artuross/homelab
        revision: HEAD
        directories:
          - path: kubernetes/core/*/app
  template:
    metadata:
      name: core-{{ index .path.segments 2 }}-app
    spec:
      project: default
      source:
        repoURL: git@github.com:artuross/homelab
        targetRevision: HEAD
        path: "{{ .path.path }}"
      destination:
        name: in-cluster
        namespace: core-argocd
      syncPolicy:
        automated:
          prune: false
          selfHeal: true
          allowEmpty: true
```

To make ArgoCD aware of those new resources, I need to add them to `kustomization.yaml` in `kubernetes/cluster/apps`:

```diff
  apiVersion: kustomize.config.k8s.io/v1beta1
  kind: Kustomization
- resources: []
+ resources:
+  - resources/Application--core-argocd--cluster-namespaces.yaml
+  - resources/ApplicationSet--core-argocd--core-app.yaml
+  - resources/ApplicationSet--core-argocd--core-crds.yaml
```

Once commited and pushed to remote repository, ArgoCD should pick up the changes (within couple of minutes) and sync everything.

## Summary

After setting up ArgoCD, I no longer need to manually deploy anything to the cluster. In the next post, I'll begin work towards exposing services in the cluster to my tailnet.
