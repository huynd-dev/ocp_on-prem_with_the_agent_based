# 2. Disconnected Registry Setup

An air-gapped/disconnected OpenShift install needs a local image registry to mirror release
and operator images into. Pick **one** of the two options below.

## Option A — Harbor

1. Download a release from the [Harbor GitHub releases page](https://github.com/goharbor/harbor/releases).
2. Configure `harbor.yml` from the template at [`../manifests/harbor.yml.example`](../manifests/harbor.yml.example):

   ```bash
   cp manifests/harbor.yml.example harbor.yml
   # edit hostname, harbor_admin_password, database.password, and the TLS
   # certificate/key paths before installing
   ```
3. Run Harbor's `install.sh` per the release instructions.

## Option B — Red Hat Quay (mirror-registry)

1. Download and extract the mirror-registry installer:

   ```bash
   wget https://mirror.openshift.com/pub/cgw/mirror-registry/latest/mirror-registry-amd64.tar.gz
   tar -xzvf mirror-registry-amd64.tar.gz
   ```
2. Install it, pointing storage/db at persistent disks:

   ```bash
   export REGISTRY_SERVER="registry.<domain>.com"
   ./mirror-registry install \
     --quayHostname "$REGISTRY_SERVER" \
     --quayRoot /data/quay \
     --quayStorage /data/quay/storage \
     --sqliteStorage /data/quay/sqlitedb \
     --initUser <admin-user> \
     --initPassword <admin-password>
   ```
3. Trust the generated root CA so `https://$REGISTRY_SERVER` works without `--tls-verify=false`:

   ```bash
   cp /data/quay/quay-rootCA/rootCA.pem /etc/pki/ca-trust/source/anchors/
   update-ca-trust extract
   ```

The same root CA (or Harbor's certificate) is what goes into `additionalTrustBundle` in
`install-config.yaml` — see [4. Cluster Install](04-cluster-install.md).

---
Next: [3. Disconnected Mirroring](03-disconnected-mirroring.md)
