---
title: Installing Talos on Raspberry Pi 5
date: 2025-10-04T18:10:00+02:00
series: ["Talos cluster on Raspberry Pi 5"]
---

A few months ago, [some smart folks](https://github.com/siderolabs/sbc-raspberrypi/issues/23#issue-2563309502) have figured out patches required to make [Talos](https://www.talos.dev/) work on Raspberry Pi 5. As Talos is my favorite Kubernetes distro, I've been waiting for this moment for a long time. In this post, I'll share all steps required to get Talos up and running on Raspberry Pi 5.

Before we begin, you can find all files from this blog post in my [`homelab`](https://github.com/artuross/homelab) repository, specifically commits [`b0f7125`](https://github.com/artuross/homelab/commit/b0f712519992e7cd3b2ce97bae838415cda750e7) and [`2485555`](https://github.com/artuross/homelab/commit/24855553b9aeef4f25c3d6320477c6781426df27).

## Hardware

I’m running a 2-node Talos cluster using Raspberry Pi 5 boards (8 GB RAM, 512 GB NVMe drives). As I only have 2 free boards, my setup will have only 1 control plane and 1 worker.

## Preparing Talos image

To install Talos on my Raspberry Pis with SSDs, I need to prepare a custom installer. Normally, I'd use [Talos Image Factory](https://factory.talos.dev/), but since I also need to use a custom kernel, this step must be done manually.

**Note:** I use [Nushell](https://www.nushell.sh/) as my shell, so all commands will be in Nushell's syntax unless noted otherwise. Most of these commands should work in `bash`. Commands wrapped in parentheses `()` are multi-line. In `bash`, you can use backslash `\` at the end of each line instead.
{.sidenote}

I'll be installing Talos `v1.10.2` (I will update nodes to a newer Talos version at the end of this blog post).

The command below will build the installer image archive and place it in `./talos/.build` directory. It also includes extensions [`iscsi-tools`](https://github.com/siderolabs/extensions/pkgs/container/iscsi-tools) and [`util-linux-tools`](https://github.com/siderolabs/extensions/pkgs/container/util-linux-tools) that I'll need in the future to set up [Longhorn](https://longhorn.io/) and an overlay.

```nu {hl_Lines=[6,9,"11-12"]}
(
    docker run
        --rm
        --tty
        --volume ./talos/.build:/out
        "ghcr.io/talos-rpi5/imager:v1.10.2-1-g8f0ce1e0b"
        installer
        --arch arm64
        --overlay-image "ghcr.io/talos-rpi5/sbc-raspberrypi5:7d04484-v1.10.0-1-gf7d2f72"
        --overlay-name rpi5
        --system-extension-image "ghcr.io/siderolabs/iscsi-tools:v0.2.0"
        --system-extension-image "ghcr.io/siderolabs/util-linux-tools:2.40.4"
)
```

The command above will create `installer-arm64.tar`. This needs to be packaged in a container image and pushed to a registry. I'm using [`crane`](https://github.com/google/go-containerregistry/tree/main/cmd/crane) for this:

```nu
crane push ./talos/.build/installer-arm64.tar "ghcr.io/artuross/talos-installer:v1.10.2-rpi5"
```

## Creating Talos configuration files

To manage my config files in a GitOps way, I have [`talhelper`](https://budimanjojo.github.io/talhelper/latest/) and [`talosctl`](https://www.talos.dev/v1.11/learn-more/talosctl/) installed.

### Preparing Talos secrets

Instead of using `talosctl` to generate manifests, I will use `talhelper gensecret` to generate a new set of secrets and print them to the console. The output should look something like this:

```yaml
cluster:
  id: <redacted>
  secret: <redacted>
secrets:
  bootstraptoken: <redacted>
  secretboxencryptionsecret: <redacted>
trustdinfo:
  token: <redacted>
certs:
  etcd:
    crt: <redacted>
    key: <redacted>
  k8s:
    crt: <redacted>
    key: <redacted>
  k8saggregator:
    crt: <redacted>
    key: <redacted>
  k8sserviceaccount:
    key: <redacted>
  os:
    crt: <redacted>
    key: <redacted>
```

I don't want to commit these to my Git repository, so I've extracted all secret values into a single 1Password item and created an `.env` file with following content:

```env
TALOS_CERTS_ETCD_CRT="op://homelab/talos/certs.etcd/crt"
TALOS_CERTS_ETCD_KEY="op://homelab/talos/certs.etcd/key"
TALOS_CERTS_K8SAGGREGATOR_CRT="op://homelab/talos/certs.k8saggregator/crt"
TALOS_CERTS_K8SAGGREGATOR_KEY="op://homelab/talos/certs.k8saggregator/key"
TALOS_CERTS_K8SSERVICEACCOUNT_KEY="op://homelab/talos/certs.k8sserviceaccount/key"
TALOS_CERTS_K8S_CRT="op://homelab/talos/certs.k8s/crt"
TALOS_CERTS_K8S_KEY="op://homelab/talos/certs.k8s/key"
TALOS_CERTS_OS_CRT="op://homelab/talos/certs.os/crt"
TALOS_CERTS_OS_KEY="op://homelab/talos/certs.os/key"
TALOS_CLUSTER_ID="op://homelab/talos/cluster/id"
TALOS_CLUSTER_SECRET="op://homelab/talos/cluster/secret"
TALOS_SECRETS_BOOTSTRAPTOKEN="op://homelab/talos/secrets/bootstraptoken"
TALOS_SECRETS_SECRETBOXENCRYPTIONSECRET="op://homelab/talos/secrets/secretboxencryptionsecret"
TALOS_TRUSTDINFO_TOKEN="op://homelab/talos/trustdinfo/token"
```

Finally, I can create the `talsecret.yaml` file with the secret values replaced with variables that will be substituted during rendering (this is placed in `talos/talsecret.yaml`):

```yaml
cluster:
  id: $TALOS_CLUSTER_ID
  secret: $TALOS_CLUSTER_SECRET
secrets:
  bootstraptoken: $TALOS_SECRETS_BOOTSTRAPTOKEN
  secretboxencryptionsecret: $TALOS_SECRETS_SECRETBOXENCRYPTIONSECRET
trustdinfo:
  token: $TALOS_TRUSTDINFO_TOKEN
certs:
  etcd:
    crt: $TALOS_CERTS_ETCD_CRT
    key: $TALOS_CERTS_ETCD_KEY
  k8s:
    crt: $TALOS_CERTS_K8S_CRT
    key: $TALOS_CERTS_K8S_KEY
  k8saggregator:
    crt: $TALOS_CERTS_K8SAGGREGATOR_CRT
    key: $TALOS_CERTS_K8SAGGREGATOR_KEY
  k8sserviceaccount:
    key: $TALOS_CERTS_K8SSERVICEACCOUNT_KEY
  os:
    crt: $TALOS_CERTS_OS_CRT
    key: $TALOS_CERTS_OS_KEY
```

`talhelper` can expand env variables in this file when generating manifests allowing me to keep secrets out of Git.

### Creating node configuration

Now that I have the secrets, I can create the Talos configuration files. Once again, `talhelper` is going to be useful. As mentioned earlier, I'll be running a 2-node cluster with 1 control plane (`rpi51` with IP `192.168.0.82`) and 1 worker (`rpi52` with IP `192.168.0.94`). For now, I will install Kubernetes `v1.30.0` and upgrade it later.

Finally, below you may notice configuration for custom volumes and user volumes. This configuration will create two partitions on the NVMe drive: `EPHEMERAL` partition for Talos files (such as downloaded container images, etc) and `persistent-data` partition - this will eventually be used by Longhorn.

Put together, `talos/talconfig.yaml` file looks like this:

```yaml
clusterName: pl-rcwz-homelab
endpoint: https://192.168.0.82:6443
kubernetesVersion: v1.30.0
talosVersion: v1.10.2
allowSchedulingOnMasters: true
nodes:
  - hostname: rpi51
    ipAddress: 192.168.0.82
    installDisk: /dev/nvme0n1
    controlPlane: true
    userVolumes: &user-volumes
      - name: persistent-data
        provisioning:
          diskSelector:
            match: disk.transport == "nvme"
          minSize: 200GB
          grow: true
    volumes: &volumes
      - name: EPHEMERAL
        provisioning:
          diskSelector:
            match: disk.transport == "nvme"
          minSize: 50GiB
          maxSize: 50GiB
  - hostname: rpi52
    ipAddress: 192.168.0.94
    installDisk: /dev/nvme0n1
    controlPlane: false
    userVolumes: *user-volumes
    volumes: *volumes
patches:
  - |
    machine:
      install:
        image: ghcr.io/artuross/talos-installer:v1.10.2-rpi5
```

### Generating Talos manifests

With all files created, I can now generate manifests that Talos can understand. I'm using [1Password CLI](https://developer.1password.com/docs/cli/) (`op`) to replace env variables in `talsecret.yaml` file with 1Password secrets.

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

Command above will create `talos/.rendered` directory with a couple of files:

- `pl-rcwz-homelab-rpi51.yaml` which contains configuration for the control plane node,
- `pl-rcwz-homelab-rpi52.yaml` which contains configuration for the worker node,
- `talosconfig` which is the Talos configuration file that `talosctl` will use.

## Installing Talos on Raspberry Pis

To install Talos on Raspberry Pi, I have an older SD card with Raspbian that I can use to boot the boards. I'll skip the instructions how to prepare the SD card as I'm reusing the one I already have.

The first step is to insert an SD card into the board, connect it to the network and power it on. Once the board is up, I can SSH into it.

```nu
ssh rpi@192.168.0.82
```

When logged in, writing the Talos image to the NVMe drive is a matter of a few commands:

As you can see below, I use a different installer than the one we just built. This is because we pushed to registry only a container image and not the raw archive. `talosctl upgrade` command can be used to "upgrade" the node to our version, however I am intentionally skipping this step as both nodes will be upgraded to `v1.10.7` at the end of this blog post.
{.sidenote}

1. Download and unpack installer.

    ```nu
    wget https://github.com/talos-rpi5/talos-builder/releases/download/v1.10.2-rpi5-pre3/metal-arm64.raw.zst
    unzstd metal-arm64.raw.zst
    ```

2. Write the image to the NVMe drive. **Running the command below will destroy all data on the drive!**

    ```nu
    sudo dd if=metal-arm64.raw of=/dev/nvme0n1 bs=4M status=progress conv=fsync
    ```

    At this point you may also want to verify whether your Pi is configured to boot from SD card first, NVMe second.

3. Shutdown the board.

    ```nu
    sudo shutdown now
    ```

4. Remove the card and boot the board from NVMe.

These steps must be repeated for the second board. Note that at this point, neither node has a role assigned.

## Bootstrapping Talos cluster

It is time to bootstrap the cluster. This section is just a short summary of the Talos docs for completeness. To make it easier to run `talosctl` commands, let's just copy the new `talosconfig` file to `~/.talos/config`. (You may want to backup an old config if you have one.)

```nu
cp talos/.rendered/talosconfig ~/.talos/config
```

Then apply the configuration to both nodes:

```nu
# apply config to control plane
talosctl apply-config --nodes "192.168.0.82" --endpoints "192.168.0.82" --file "./talos/.rendered/pl-rcwz-homelab-rpi51.yaml" --insecure

# apply config to worker
talosctl apply-config --nodes "192.168.0.94" --endpoints "192.168.0.94" --file "./talos/.rendered/pl-rcwz-homelab-rpi52.yaml" --insecure
```

After this, you should be able to monitor the nodes. For example, I can check the control plane node:

```nu
talosctl dashboard --nodes "192.168.0.82"
```

Finally, to bootstrap the cluster, `talosctl bootstrap` must be executed once:

```nu
talosctl bootstrap --nodes "192.168.0.82"
```

After a short while, the cluster should be up and running. Kubernetes config file can be retrieved with:

```nu
talosctl kubeconfig --nodes "192.168.0.82"
```

You can verify that everything is working by checking if pods start:

```nu
kubectl ctx admin@pl-rcwz-homelab
kubectl get pods -A
```

Once again, it may take a couple of minutes for the the Kubernetes to bootstrap completely. Some pods may not show up initially or be in `ContainerCreating`/`Pending` state. This is normal.

## Upgrading Talos

As a bonus, I want to update Talos version to `v1.10.7`. Before starting, let's verify that nodes are at the expected versions:

```txt
❯ kubectl get nodes -o wide
NAME    STATUS   ROLES           AGE     VERSION   INTERNAL-IP    EXTERNAL-IP   OS-IMAGE                       KERNEL-VERSION   CONTAINER-RUNTIME
rpi51   Ready    control-plane   2m50s   v1.30.0   192.168.0.82   <none>        Talos (v1.10.2-1-g8f0ce1e0b)   6.12.25-talos    containerd://2.0.5
rpi52   Ready    <none>          2m48s   v1.30.0   192.168.0.94   <none>        Talos (v1.10.2-1-g8f0ce1e0b)   6.12.25-talos    containerd://2.0.5
```

We can see in the `OS-IMAGE` column that both nodes are running Talos `v1.10.2`.

Just like before, the installer image must be built and pushed to a registry. In this case I update the imager version and `util-linux-tools` extension version (however it should be safe to skip this last one).

```diff
(
     docker run
         --rm
         --tty
         --volume ./talos/.build:/out
-        "ghcr.io/talos-rpi5/imager:v1.10.2-1-g8f0ce1e0b"
+        "ghcr.io/talos-rpi5/imager:v1.10.7-1-g5893396e8"
         installer
         --arch arm64
         --overlay-image "ghcr.io/talos-rpi5/sbc-raspberrypi5:7d04484-v1.10.0-1-gf7d2f72"
         --overlay-name rpi5
         --system-extension-image "ghcr.io/siderolabs/iscsi-tools:v0.2.0"
-        --system-extension-image "ghcr.io/siderolabs/util-linux-tools:2.40.4"
+        --system-extension-image "ghcr.io/siderolabs/util-linux-tools:2.41.1"
)
```

The archive must be packaged and pushed again, this time with a new tag:

```nu
crane push ./talos/.build/installer-arm64.tar "ghcr.io/artuross/talos-installer:v1.10.7-rpi5"
```

Update the `talos/talconfig.yaml` file to use the new version:

```diff
  clusterName: pl-rcwz-homelab
  endpoint: https://192.168.0.82:6443
  kubernetesVersion: v1.30.0
- talosVersion: v1.10.2
+ talosVersion: v1.10.7
  allowSchedulingOnMasters: true
    nodes:
    - hostname: rpi51
      ipAddress: 192.168.0.82
      installDisk: /dev/nvme0n1
      controlPlane: true
      userVolumes: &user-volumes
        - name: persistent-data
          provisioning:
            diskSelector:
              match: disk.transport == "nvme"
            minSize: 200GB
            grow: true
      volumes: &volumes
        - name: EPHEMERAL
          provisioning:
            diskSelector:
              match: disk.transport == "nvme"
            minSize: 50GiB
            maxSize: 50GiB
    - hostname: rpi52
      ipAddress: 192.168.0.94
      installDisk: /dev/nvme0n1
      controlPlane: false
      userVolumes: *user-volumes
      volumes: *volumes
  patches:
    - |
      machine:
        install:
-         image: ghcr.io/artuross/talos-installer:v1.10.2-rpi5
+         image: ghcr.io/artuross/talos-installer:v1.10.7-rpi5
```

Render the final manifests (this command has not changed):

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

And apply configuration to each node. I apply config to both nodes as I already tested these steps before, but a smarter way would be to update 1 node at a time. Notice that this time, I use `--mode no-reboot` flag.

```nu
# control plane
talosctl apply-config --nodes "192.168.0.82" --endpoints "192.168.0.82" --file "./talos/.rendered/pl-rcwz-homelab-rpi51.yaml" --mode no-reboot

# worker
talosctl apply-config --nodes "192.168.0.94" --endpoints "192.168.0.94" --file "./talos/.rendered/pl-rcwz-homelab-rpi52.yaml" --mode no-reboot
```

Finally, trigger the upgrade with:

```nu
# control plane
talosctl upgrade --image "ghcr.io/artuross/talos-installer:v1.10.7-rpi5" --nodes "192.168.0.82" --endpoints "192.168.0.82" --stage

# worker
talosctl upgrade --image "ghcr.io/artuross/talos-installer:v1.10.7-rpi5" --nodes "192.168.0.94" --endpoints "192.168.0.94" --stage
```

Listing nodes once again, I can see the version updated:

```txt
❯ kubectl get nodes -o wide
NAME    STATUS   ROLES           AGE   VERSION   INTERNAL-IP    EXTERNAL-IP   OS-IMAGE                       KERNEL-VERSION   CONTAINER-RUNTIME
rpi51   Ready    control-plane   27m   v1.30.0   192.168.0.82   <none>        Talos (v1.10.7-1-g5893396e8)   6.12.25-talos    containerd://2.0.5
rpi52   Ready    <none>          27m   v1.30.0   192.168.0.94   <none>        Talos (v1.10.7-1-g5893396e8)   6.12.25-talos    containerd://2.0.5
```

## Summary

Talos is now running on a Raspberry Pi 5 cluster! There are a bit more steps compared to running Talos on a fully supported hardware, but compare that to running other Kubernetes on other Linux distros and no wonder why so many people enjoy using Talos.

In the future blog posts, I'll document adding operators to build a fully functional cluster.
