# 5. Adding Worker Nodes (Day 2)

Control-plane replicas are fixed at install time (`compute.replicas: 0` in
`install-config.yaml`); workers are added afterwards by generating a per-host boot
image with `oc adm node-image`.

## Describe the worker hosts

Copy the template and add one entry per worker (hostname, MAC address, static IP):

```bash
cp configs/nodes-config.yaml.example nodes-config.yaml
```

See [`configs/nodes-config.yaml.example`](../configs/nodes-config.yaml.example) for the
full schema — it mirrors the `networkConfig` block from `agent-config.yaml`.

## Generate and boot the worker image

```bash
oc adm node-image create nodes-config.yaml --registry-config=/root/.docker/config.json
```

Boot each worker host from the generated image, then track its bootstrap:

```bash
oc adm node-image monitor --ip-addresses <ip-node>
```

Once a worker joins, approve its CSRs if needed and confirm it shows up with `oc get nodes`.

---
Next: [6. ODF Storage](06-odf-storage.md)
