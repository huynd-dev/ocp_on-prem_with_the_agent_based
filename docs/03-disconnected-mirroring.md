# 3. Disconnected Registry Mirroring

Goal: pull the OpenShift release and the operator catalogs from the internet onto a
staging host, then re-mirror them from that staging host into the private/air-gapped
registry from [2. Registry Setup](02-registry-setup.md). The result feeds
`imageDigestSources` in `install-config.yaml` (see [4. Cluster Install](04-cluster-install.md)).

## 3.1 Prepare the staging host (internet-connected)

This is the machine that pulls the OpenShift bits from the internet before they're
copied into the air-gapped environment.

Minimum spec:

| Resource        | Requirement          |
|-----------------|-----------------------|
| CPU / RAM       | 16 vCPU / 32 GB RAM   |
| OS              | Ubuntu 24.04 LTS (or RHEL 9.x) |
| `/data`         | 1 TB                  |
| `/mirror-cache` | 1 TB                  |
| Network         | Internet access       |
| Software        | Docker installed      |

### Install oc-mirror

```bash
# Stable build (recommended)
curl -LO https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/stable-4.20/oc-mirror.tar.gz

# RHEL 9 build
curl -LO https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/stable-4.20/oc-mirror.rhel9.tar.gz

tar -xvf oc-mirror.rhel9.tar.gz
chmod +x oc-mirror
sudo mv oc-mirror /usr/local/bin/

oc-mirror version
```

### Create the pull secret

Get your pull secret from
[console.redhat.com/openshift/downloads#tool-pull-secret](https://console.redhat.com/openshift/downloads#tool-pull-secret)
and save it as `pull-secret.txt`, then:

```bash
apt install jq -y
cat pull-secret.txt | jq > pull-secret.json
mkdir -p /root/.docker/
cp pull-secret.json /root/.docker/auth.json
```

## 3.2 Mirror from the internet to local disk

Each of the three catalogs below (platform release, Red Hat operators, certified
operators) has its own `ImageSetConfiguration`, checked in under
[`configs/imageset/`](../configs/imageset/), and its own local directory under `/data`.

### 3.2.1 OpenShift platform release

`ImageSetConfiguration`: [`configs/imageset/imageset_platform_release_4_20.yaml`](../configs/imageset/imageset_platform_release_4_20.yaml)

```bash
mkdir -p /data/release
cd /data
cp /path/to/repo/configs/imageset/imageset_platform_release_4_20.yaml .

oc-mirror -c ./imageset_platform_release_4_20.yaml file:///data/release \
  --v2 --log-level debug --retry-delay 15s --retry-times 10
```

A successful run ends with an `INFO` completion log.

### 3.2.2 Red Hat Operator catalog

`ImageSetConfiguration`: [`configs/imageset/imageset_redhat_operator_index.yaml`](../configs/imageset/imageset_redhat_operator_index.yaml)
— trim the `packages` list to what your cluster actually needs.

```bash
mkdir -p /data/redhat-operator
cd /data
cp /path/to/repo/configs/imageset/imageset_redhat_operator_index.yaml .

oc-mirror -c ./imageset_redhat_operator_index.yaml file:///data/redhat-operator \
  --v2 --log-level debug --retry-delay 15s --retry-times 10
```

### 3.2.3 Certified Operator catalog

`ImageSetConfiguration`: [`configs/imageset/imageset_certified_4_20.yaml`](../configs/imageset/imageset_certified_4_20.yaml)
— trim the `packages` list to what your cluster actually needs.

```bash
mkdir -p /data/certified-operator
cd /data
cp /path/to/repo/configs/imageset/imageset_certified_4_20.yaml .

oc-mirror -c ./imageset_certified_4_20.yaml file:///data/certified-operator \
  --v2 --log-level debug --retry-delay 15s --retry-times 10
```

## 3.3 Mirror from local disk to the private registry

Once `/data/release`, `/data/redhat-operator` and `/data/certified-operator` have been
copied from the internet-connected staging host into the air-gapped environment (e.g.
onto the bastion), push them into the private registry.

Build an auth file for the private registry:

```bash
cd /data
export REGISTRY='<private-registry-host>'
export USER='<registry-user>'
export PASS='<registry-password>'
echo "{\"auths\":{\"$REGISTRY\":{\"auth\":\"$(echo -n "$USER:$PASS" | base64)\"}}}" > auth.json
```

### 3.3.1 Push the platform release

```bash
oc-mirror -c ./imageset_platform_release_4_20.yaml \
  docker://$REGISTRY/redhat/release -a ./auth.json \
  --v2 --log-level debug --retry-delay 15s --retry-times 10
```

### 3.3.2 Push the Red Hat Operator catalog

```bash
oc-mirror -c ./imageset_redhat_operator_index.yaml \
  docker://$REGISTRY/redhat/redhat-operator -a ./auth.json \
  --v2 --log-level debug --retry-delay 15s --retry-times 10
```

### 3.3.3 Push the Certified Operator catalog

```bash
oc-mirror -c ./imageset_certified_4_20.yaml \
  docker://$REGISTRY/redhat/certified-operator -a ./auth.json \
  --v2 --log-level debug --retry-delay 15s --retry-times 10
```

Each `oc-mirror` push writes an `ImageDigestMirrorSet`/`imageDigestSources` mapping under
its working directory — that mapping is what goes into `install-config.yaml`'s
`imageDigestSources` (see [4. Cluster Install](04-cluster-install.md)).

---
Next: [4. Cluster Install](04-cluster-install.md)
