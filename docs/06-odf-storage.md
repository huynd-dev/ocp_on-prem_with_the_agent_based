# 6. Installing ODF (OpenShift Data Foundation)

## Label the storage nodes

```bash
oc label node <node> node-role.kubernetes.io/infra=""
oc label node <node> cluster.ocs.openshift.io/openshift-storage=""
```

## Inspect available disks

```bash
NODE_NAME=node/compute01.hub.bca.gov.vn
cat <<EOF | oc debug $NODE_NAME
chroot /host
lsblk -o NAME,ROTA,SIZE,TYPE
EOF
```

```bash
NODE_NAME=node/compute01.hub.bca.gov.vn
cat <<EOF | oc debug $NODE_NAME
chroot /host
ls -aslc /dev/disk/by-path/
EOF
```

## Workaround: force virtual disks to report as non-rotational

ODF's disk auto-detection skips disks it thinks are spinning HDDs. On virtualized
infrastructure the backing disk is often reported as rotational even though it's
flash-backed — this MachineConfig flips a matching disk's `rotational` flag at boot
so ODF picks it up.

Source (Butane): [`../manifests/99-fake-nonrotational-mc.bu`](../manifests/99-fake-nonrotational-mc.bu)
Rendered manifest: [`../manifests/99-fake-nonrotational-mc.yaml`](../manifests/99-fake-nonrotational-mc.yaml)

> The script matches disks by exact size (`500G` by default) — adjust `target_disks`
> in the `.bu` source and re-render with `butane` if your storage disks differ.

```bash
oc apply -f manifests/99-fake-nonrotational-mc.yaml
```

Verify:

```bash
NODE_NAME=node/compute01.hub.bca.gov.vn
cat <<EOF | oc debug $NODE_NAME
cat /sys/block/vdb/queue/rotational
EOF
```

```bash
NODE_NAME=node/compute01.hub.bca.gov.vn
cat <<EOF | oc debug $NODE_NAME
chroot /host
journalctl | grep -e fake-nonrotational
EOF
```

`rotational` should read `0`, and the journal should show the `fake-nonrotational.service`
unit having run successfully.
