---
title: Adding Cilium to a Talos cluster
date: 2025-10-08T23:00:00+02:00
series: ["Talos cluster on Raspberry Pi 5"]
---

In today's post, I will document how to install [Cilium](https://cilium.io/) on a Talos-managed Kubernetes cluster. Cilium is going to replace the default [Flannel](https://github.com/flannel-io/flannel) and [`kube-proxy`](https://github.com/kubernetes/kubernetes/tree/master/cmd/kube-proxy).

As always, you can find all code in my [homelab Git repository](https://github.com/artuross/homelab) and all changes in this post are in [this commit](https://github.com/artuross/homelab/commit/e4ae409b9d0071ac437450751e78ce8b84a537bb).

## Prerequisites

My cluster was created in the previous post. While most steps should be applicable to any Talos cluster, some steps may reuse files created in the previous post. If you haven't set up a Talos cluster yet, please refer to the [first post in the series](/2025-10-04-installing-talos-on-raspberry-pi-5/).

### `kubesource`

I prefer to vendor rendered 3rd party manifests into my Git repository. I wrote a small tool called [`kubesource`](https://github.com/artuross/kubesource) to help with that. This post will use `kubesource` everywhere, but just know that you could use `kustomize` and some clever patches to achieve the same effect.

To install `kubesource`, run:

```nu
go install github.com/artuross/kubesource/cmd/kubesource@latest
```

## Preparing Cilium manifests

### Creating namespace

I will deploy Cilium into `core-cilium` namespace. While working with Kubernetes, I found that I prefer to create namespaces beforehand and separate them from application manifests.

Let's create a file `kubernetes/cluster/namespaces/resources/core-cilium.yaml`:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  labels:
    pod-security.kubernetes.io/enforce: privileged
    pod-security.kubernetes.io/audit: privileged
    pod-security.kubernetes.io/warn: privileged
  name: core-cilium
```

I will eventually add more namespaces to this directory. To make it easier to manage, I will create a `kustomization.yaml` file in `kubernetes/cluster/namespaces`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - resources/core-cilium.yaml
```

To create the namespace, run:

```nu
kubectl apply --kustomize kubernetes/cluster/namespaces
```

### Creating base config

To create Cilium manifests, I will use the [official Helm chart](https://github.com/cilium/cilium/tree/v1.18.0/install/kubernetes/cilium). For now, let's create a minimal config in `kubernetes/core/cilium/_source/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
helmCharts:
  - name: cilium
    namespace: core-cilium
    repo: https://helm.cilium.io/
    releaseName: cilium
    includeCRDs: false
    version: 1.18.0
```

I'm using version `1.18.0`. This can be render with `kustomize`:

```nu
kustomize build --enable-helm ./kubernetes/core/cilium/_source/
```

However I don't just want to print it to the console. This is where `kubesource` comes in. For now, create file `kubernetes/core/cilium/kubesource.yaml` with following content:

```yaml
apiVersion: kubesource.rcwz.pl/v1alpha1
kind: Config
sourceDir: _source
targets:
  - directory: app/base
```

Long story short, this will run `kustomize build` command with the target of `kubernetes/core/cilium/_source` and save the output to `kubernetes/core/cilium/app/base`. Now, in the root of the repo, run:

```nu
kubesource
```

As I'm writing this, `kubesource` has a very minimalistic output:

```text
Processing kubernetes/core/cilium
  Source directory: kubernetes/core/cilium/_source
  Saving to: kubernetes/core/cilium/app/base
  ✓ Successfully processed kubernetes/core/cilium
```

The result is saved in `kubernetes/core/cilium/app/base` and looks like this:

```text {hl_Lines=[12,"19-20"]}
❯ ls -1 kubernetes/core/cilium/app/base/
ClusterRole--cilium-operator.yaml
ClusterRole--cilium.yaml
ClusterRoleBinding--cilium-operator.yaml
ClusterRoleBinding--cilium.yaml
ConfigMap--core-cilium--cilium-config.yaml
ConfigMap--core-cilium--cilium-envoy-config.yaml
DaemonSet--core-cilium--cilium-envoy.yaml
DaemonSet--core-cilium--cilium.yaml
Deployment--core-cilium--cilium-operator.yaml
kustomization.yaml
Namespace--cilium-secrets.yaml
Role--cilium-secrets--cilium-operator-tlsinterception-secrets.yaml
Role--cilium-secrets--cilium-tlsinterception-secrets.yaml
Role--core-cilium--cilium-config-agent.yaml
RoleBinding--cilium-secrets--cilium-operator-tlsinterception-secrets.yaml
RoleBinding--cilium-secrets--cilium-tlsinterception-secrets.yaml
RoleBinding--core-cilium--cilium-config-agent.yaml
Secret--core-cilium--cilium-ca.yaml
Secret--core-cilium--hubble-server-certs.yaml
Service--core-cilium--cilium-envoy.yaml
Service--core-cilium--hubble-peer.yaml
ServiceAccount--core-cilium--cilium-envoy.yaml
ServiceAccount--core-cilium--cilium-operator.yaml
ServiceAccount--core-cilium--cilium.yaml
```

Each file in the directory is a separate manifest. This is our base config that we can further customize with Helm values and then with Kustomize patches. Before I do this, there's the `Namespace`, highlighted in the output above and 2 `Secret` manifests that I don't want to commit to Git.

I'll change the Cilium config later, so let's take a look at the secrets first:

```yaml
# Secret--core-cilium--cilium-ca.yaml
apiVersion: v1
data:
  ca.crt: <redacted>
  ca.key: <redacted>
kind: Secret
metadata:
  name: cilium-ca
  namespace: core-cilium
```

and

```yaml
# Secret--core-cilium--hubble-server-certs.yaml
apiVersion: v1
data:
  ca.crt: <redacted>
  tls.crt: <redacted>
  tls.key: <redacted>
kind: Secret
metadata:
  name: hubble-server-certs
  namespace: core-cilium
type: kubernetes.io/tls
```

### Extracting secrets

I don't want to commit secrets to Git, but I need them to deploy Cilium. I've simply copy pasted each secret value (in their `base64`-encoded form) into 1Password items. Finally, I've created `scripts/bootstrap.nu` file to programatically create the secrets from 1Password and apply them to the cluster later.

```nu
#!/usr/bin/env nu

# create the secrets directory
mkdir kubernetes/bootstrap/secrets

# create Cilium secrets
mkdir kubernetes/bootstrap/secrets/cilium
op read --no-newline "op://homelab/addons.cilium.ca/ca.crt" | base64 --decode | save --force kubernetes/bootstrap/secrets/cilium/ca.crt
op read --no-newline "op://homelab/addons.cilium.ca/ca.key" | base64 --decode | save --force kubernetes/bootstrap/secrets/cilium/ca.key
op read --no-newline "op://homelab/addons.cilium.hubble-server-certs/tls.crt" | base64 --decode | save --force kubernetes/bootstrap/secrets/cilium/tls.crt
op read --no-newline "op://homelab/addons.cilium.hubble-server-certs/tls.key" | base64 --decode | save --force kubernetes/bootstrap/secrets/cilium/tls.key
```

This script creates the necessary directories and then reads each secret value from 1Password and saves it to a separate file. Values need to be decoded as `kustomize` expects values to be in plain text.

**Make sure to add `/kubernetes/bootstrap/secrets/` to `.gitignore` to avoid spilling secrets.**

Now it's time to create `kubernetes/bootstrap/kustomization.yaml` to generate the secrets from files:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
secretGenerator:
  - name: cilium-ca
    namespace: core-cilium
    files:
      - secrets/cilium/ca.crt
      - secrets/cilium/ca.key
    options:
      disableNameSuffixHash: true

  - name: hubble-server-certs
    namespace: core-cilium
    files:
      - secrets/cilium/ca.crt
      - secrets/cilium/tls.crt
      - secrets/cilium/tls.key
    type: kubernetes.io/tls
    options:
      disableNameSuffixHash: true
```

I could apply this with `kubectl`, but since I already have a bootstrap script, I've updated it to be self sufficient:

```diff
  #!/usr/bin/env nu

  # create the secrets directory
  mkdir kubernetes/bootstrap/secrets

  # create Cilium secrets
  mkdir kubernetes/bootstrap/secrets/cilium
  op read --no-newline "op://homelab/addons.cilium.ca/ca.crt" | base64 --decode | save --force   kubernetes/bootstrap/secrets/cilium/ca.crt
  op read --no-newline "op://homelab/addons.cilium.ca/ca.key" | base64 --decode | save --force   kubernetes/bootstrap/secrets/cilium/ca.key
  op read --no-newline "op://homelab/addons.cilium.hubble-server-certs/tls.crt" | base64 --decode | save   --force kubernetes/bootstrap/secrets/cilium/tls.crt
  op read --no-newline "op://homelab/addons.cilium.hubble-server-certs/tls.key" | base64 --decode | save   --force kubernetes/bootstrap/secrets/cilium/tls.key

+ # create namespaces
+ kubectl apply --kustomize kubernetes/cluster/namespaces
+
+ # create secrets
+ kubectl apply --kustomize kubernetes/bootstrap
```

Finally, `kubesource` can filter out the secrets from the rendered manifests. Update `kubesource.yaml`:

```diff
  apiVersion: kubesource.rcwz.pl/v1alpha1
  kind: Config
  sourceDir: _source
  targets:
    - directory: app/base
+     filter:
+       exclude:
+         - kind: Secret
```

After running `kubesource` again, the output should not contain any secrets.

```diff
  ❯ ls -1 kubernetes/core/cilium/app/base/
  ClusterRole--cilium-operator.yaml
  ClusterRole--cilium.yaml
  ClusterRoleBinding--cilium-operator.yaml
  ClusterRoleBinding--cilium.yaml
  ConfigMap--core-cilium--cilium-config.yaml
  ConfigMap--core-cilium--cilium-envoy-config.yaml
  DaemonSet--core-cilium--cilium-envoy.yaml
  DaemonSet--core-cilium--cilium.yaml
  Deployment--core-cilium--cilium-operator.yaml
  kustomization.yaml
  Namespace--cilium-secrets.yaml
  Role--cilium-secrets--cilium-operator-tlsinterception-secrets.yaml
  Role--cilium-secrets--cilium-tlsinterception-secrets.yaml
  Role--core-cilium--cilium-config-agent.yaml
  RoleBinding--cilium-secrets--cilium-operator-tlsinterception-secrets.yaml
  RoleBinding--cilium-secrets--cilium-tlsinterception-secrets.yaml
  RoleBinding--core-cilium--cilium-config-agent.yaml
  Secret--core-cilium--cilium-ca.yaml
  Secret--core-cilium--hubble-server-certs.yaml
- Service--core-cilium--cilium-envoy.yaml
- Service--core-cilium--hubble-peer.yaml
  ServiceAccount--core-cilium--cilium-envoy.yaml
  ServiceAccount--core-cilium--cilium-operator.yaml
  ServiceAccount--core-cilium--cilium.yaml
```

Let's also create `kubernetes/core/cilium/app/kustomization.yaml` and include our base config:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - base
```

### Customizing Cilium

So far, I've been using default values from the Helm chart. Talos has a [documentation page](https://www.talos.dev/v1.10/kubernetes-guides/network/deploying-cilium/) that documents the installation process. Converting the values from Cilium CLI to Helm values, we end up with `kubernetes/core/cilium/_source/values.yaml`:

```yaml
cgroup:
  autoMount:
    enabled: false
  hostRoot: /sys/fs/cgroup

ipam:
  mode: kubernetes

k8sServiceHost: localhost
k8sServicePort: 7445
kubeProxyReplacement: true

securityContext:
  capabilities:
    ciliumAgent:
      - CHOWN
      - DAC_OVERRIDE
      - FOWNER
      - IPC_LOCK
      - KILL
      - NET_ADMIN
      - NET_RAW
      - SETGID
      - SETUID
      - SYS_ADMIN
      - SYS_RESOURCE
    cleanCiliumState:
      - NET_ADMIN
      - SYS_ADMIN
      - SYS_RESOURCE
```

This `values.yaml` file is currently not used, the `kustomization.yaml` file in the same directory must be patched to include it:

```diff
  apiVersion: kustomize.config.k8s.io/v1beta1
  kind: Kustomization
  helmCharts:
    - name: cilium
      namespace: core-cilium
      repo: https://helm.cilium.io/
      releaseName: cilium
      includeCRDs: false
      version: 1.18.0
+     valuesFile: values.yaml
```

Once again, run `kubesource` to regenerate the manifests.

I still have to remove the `Namespace` that the Cilium chart creates. While fixing this, I will also update deployment to start only 1 replicas as I currently have only 2 nodes in the cluster anyway.

`kubernetes/core/cilium/_source/values.yaml` has following changes:

```diff
  cgroup:
    autoMount:
      enabled: false
  hostRoot: /sys/fs/cgroup

  ipam:
    mode: kubernetes

  k8sServiceHost: localhost
  k8sServicePort: 7445
  kubeProxyReplacement: true

+ # run only 1 replica
+ operator:
+   replicas: 1
+
+ # reuse core-cilium namespace
+ tls:
+   secretsNamespace:
+     create: false
+     name: core-cilium

  securityContext:
    capabilities:
      ciliumAgent:
        - CHOWN
        - DAC_OVERRIDE
        - FOWNER
        - IPC_LOCK
        - KILL
        - NET_ADMIN
        - NET_RAW
        - SETGID
        - SETUID
        - SYS_ADMIN
        - SYS_RESOURCE
      cleanCiliumState:
        - NET_ADMIN
        - SYS_ADMIN
        - SYS_RESOURCE
```

With my minimal changes done, I can run `kubesource` one last time to generate the final manifests.

One more patch to `bootstrap.nu` to apply the Cilium manifests:

```diff
  #!/usr/bin/env nu

  # create the secrets directory
  mkdir kubernetes/bootstrap/secrets

  # create Cilium secrets
  mkdir kubernetes/bootstrap/secrets/cilium
  op read --no-newline "op://homelab/addons.cilium.ca/ca.crt" | base64 --decode | save --force kubernetes/bootstrap/secrets/cilium/ca.crt
  op read --no-newline "op://homelab/addons.cilium.ca/ca.key" | base64 --decode | save --force kubernetes/bootstrap/secrets/cilium/ca.key
  op read --no-newline "op://homelab/addons.cilium.hubble-server-certs/tls.crt" | base64 --decode | save --force kubernetes/bootstrap/secrets/cilium/tls.crt
  op read --no-newline "op://homelab/addons.cilium.hubble-server-certs/tls.key" | base64 --decode | save --force kubernetes/bootstrap/secrets/cilium/tls.key

  # create namespaces
  kubectl apply --kustomize kubernetes/cluster/namespaces

  # create secrets
  kubectl apply --kustomize kubernetes/bootstrap

+ # deploy apps
+ kubectl apply --kustomize kubernetes/core/cilium/app
```

## Removing Flannel and `kube-proxy` from Talos

### Preparing Talos config

With Cilium config prepared, I need to remove Flannel and `kube-proxy` from Talos before I can deploy Cilium. I already have `talconfig.yaml` filed [prepared in the previous post](https://github.com/artuross/homelab/blob/main/talos/talconfig.yaml). In the `patches` section, a new value must be added to disable the default CNI and proxy:

```diff
  patches:
    # use custom installer for Raspberry Pi 5, based on the work of:
    # https://github.com/siderolabs/sbc-raspberrypi/issues/23
    # https://github.com/talos-rpi5/talos-builder/releases/tag/v1.10.7-rpi5
    - |
      machine:
        install:
          image: ghcr.io/artuross/talos-installer:v1.10.7-rpi5

+   # disable CNI and proxy to use Cilium
+   - |-
+     cluster:
+       network:
+         cni:
+           name: none
+       proxy:
+         disabled: true
```

To render Talos manifests for each node, I will use the same command from previous post:

```nu
(
    op run
        --env-file .env
        --no-masking
        --
        talhelper genconfig
            --config-file talos/talconfig.yaml
            --no-gitignore
            --out-dir talos/.rendered
            --secret-file talos/talsecret.yaml
)
```

Each node must be patched with its own config:

### Deploying Cilium

```nu
# control plane
talosctl apply-config --nodes "192.168.0.82" --endpoints "192.168.0.82" --file "./talos/.rendered/pl-rcwz-homelab-rpi51.yaml"

# worker
talosctl apply-config --nodes "192.168.0.94" --endpoints "192.168.0.94" --file "./talos/.rendered/pl-rcwz-homelab-rpi52.yaml"
```

In my case, nodes did not need to reboot. We should now expect that Talos will not have Flannel or `kube-proxy` running, however Talos **does not** remove resources automatically. We need to delete each `DaemonSet`, which can be done with a single command:

```nu
kubectl -n kube-system delete ds kube-flannel kube-proxy
```

It's now time to run previously created bootstrap script. As a reminder, the script will create all necessary resources in the cluster, including Cilium:

```nu
chmod 740 scripts/bootstrap.nu
./scripts/bootstrap.nu
```

It's gonna take a minute for all pods to become healthy, but eventually everything should be running:

```text
❯ kubectl get pods -A
NAMESPACE     NAME                               READY   STATUS    RESTARTS        AGE
core-cilium   cilium-envoy-6kghh                 1/1     Running   0               69s
core-cilium   cilium-envoy-qmxsx                 1/1     Running   0               69s
core-cilium   cilium-operator-5d7f7c9dd9-l2t9t   1/1     Running   0               69s
core-cilium   cilium-tbqn7                       1/1     Running   0               69s
core-cilium   cilium-wm5bf                       1/1     Running   0               69s
kube-system   coredns-8477467d67-687gr           1/1     Running   0               58s
kube-system   coredns-8477467d67-jrbf8           1/1     Running   0               42s
kube-system   kube-apiserver-rpi51               1/1     Running   0               4h49m
kube-system   kube-controller-manager-rpi51      1/1     Running   2 (4h49m ago)   4h49m
kube-system   kube-scheduler-rpi51               1/1     Running   3 (4h49m ago)   4h49m
```

### Verifying Cilium installation

Cilium has its own CLI tool that is often used for installation. We're gonna use it to verify whether Cilium was deployed correctly. To install it, run:

```nu
brew install cilium-cli
```

Now, check the status:

```nu
cilium status -n core-cilium
```

The output should look similar to mine:

```text
❯ cilium status -n core-cilium
    /¯¯\
 /¯¯\__/¯¯\    Cilium:             OK
 \__/¯¯\__/    Operator:           OK
 /¯¯\__/¯¯\    Envoy DaemonSet:    OK
 \__/¯¯\__/    Hubble Relay:       disabled
    \__/       ClusterMesh:        disabled

...
```

## Summary

Cilium is now successfully deployed to the Talos cluster. This is certainly an overkill for my tiny cluster, but knowing Cilium will be useful in the future and in larger clusters.

In the next post, I'll tackle secret management in my cluster.
