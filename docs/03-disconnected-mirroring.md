# 3. Disconnected Registry Mirroring

> **Status: not yet documented.** The original notes for this repo jump straight from
> "registry is running" to "reference it in `install-config.yaml`" — the actual
> `oc-mirror` mirroring run (ImageSetConfiguration, `oc-mirror` invocation, mirrored
> tag/digest verification) was never written down. Fill this in with the exact
> commands used for this environment; until then, follow Red Hat's official guide:
> [Mirroring images for a disconnected installation](https://docs.openshift.com/container-platform/latest/installing/disconnected_install/index.html).

What this step needs to produce, at minimum:

1. An `ImageSetConfiguration` describing the OCP release (and any operators/catalogs) to mirror.
2. A completed `oc-mirror` run that pushes those images into the registry from
   [2. Registry Setup](02-registry-setup.md).
3. The resulting `imageDigestSources` / `imageContentSources` mapping, which feeds directly
   into `install-config.yaml` (already present in the template — see
   [4. Cluster Install](04-cluster-install.md)).

---
Next: [4. Cluster Install](04-cluster-install.md)
