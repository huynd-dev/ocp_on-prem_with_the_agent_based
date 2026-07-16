# 4. Installing the Cluster

## Download the OpenShift client tools

```bash
wget https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/4.19.0/openshift-client-linux-4.19.0.tar.gz
tar -xvf openshift-client-linux-4.19.0.tar.gz

wget https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/latest/oc-mirror.rhel9.tar.gz
tar -xzvf oc-mirror.rhel9.tar.gz

wget https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/4.19.0/openshift-install-linux-4.19.0.tar.gz
tar -xvf openshift-install-linux-4.19.0.tar.gz
```

## Create the pull secret

Get your base pull secret from
[console.redhat.com/openshift/downloads#tool-pull-secret](https://console.redhat.com/openshift/downloads#tool-pull-secret),
then merge in your disconnected registry's auth (see [2. Registry Setup](02-registry-setup.md))
before using it in `install-config.yaml`:

```bash
yum install jq -y
cat pull-secret.txt | jq > pull-secret.json
mkdir -p /root/.docker/
cp pull-secret.json /root/.docker/auth.json
```

## Create install-config.yaml and agent-config.yaml

```bash
mkdir -p openshift/install
cd openshift/install
cp ../../configs/install-config.yaml.example install-config.yaml
cp ../../configs/agent-config.yaml.example agent-config.yaml
```

Edit both files for your environment:

- [`configs/install-config.yaml.example`](../configs/install-config.yaml.example) — cluster
  name/domain, network CIDRs, API/Ingress VIPs, pull secret, SSH key, registry CA and
  `imageDigestSources` from [3. Disconnected Mirroring](03-disconnected-mirroring.md).
- [`configs/agent-config.yaml.example`](../configs/agent-config.yaml.example) — one entry per
  control-plane host with its MAC address and static network config, plus a `rendezvousIP`
  that all hosts can reach during bootstrap.

Both files are consumed together and deleted/rewritten by the installer, so keep your edited
copies only under `openshift/install/` (already covered by `.gitignore`).

## Create the install ISO

```bash
openshift-install agent create image --dir ./install
```

Boot each control-plane host from the resulting ISO, then watch progress:

```bash
openshift-install agent --dir install wait-for bootstrap-complete --log-level=debug
openshift-install agent --dir install wait-for install-complete --log-level=debug
```

---
Next: [5. Worker Node Addition](05-worker-node-addition.md)
