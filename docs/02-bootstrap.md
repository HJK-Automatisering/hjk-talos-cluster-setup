---

## docs/02-bootstrap.md

```markdown
---
layout: default
title: Bootstrap af cluster
nav_order: 3
---

# Bootstrap af cluster

## Generering af secrets og maskinkonfigurationer

1. Opret mappe til konfigurationer:
```bash
mkdir -p ~/k8s-configs/cluster2/secrets
```

2. Generér secrets:
```bash
talosctl gen secrets --output secrets.yaml
```

3. Gem secrets sikkert (fx Proton Pass). Slet dem fra arbejdsmappen efter brug.


4. Generér maskinkonfiguration:
```bash
talosctl gen config cluster1 https://<control-plane-ip>:6443 --output machine-config.yaml
```

5. Tilpas konfigurationen med patches efter behov.