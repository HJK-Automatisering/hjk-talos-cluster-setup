# HJK Talos Cluster Setup

Denne repository indeholder dokumentationen og konfigurationsfilerne til opsætning af en **single-node Kubernetes-klynge** baseret på:

- **Talos Linux**  
- **Cilium** (CNI)  
- **Rook Ceph** (single-node testmiljø)  
- **OS2AI**  
- **Memoctopus MVP**  
- Adgang fra **Ubuntu WSL2**  

Formålet er at sikre en ensartet, reproducérbar og veldokumenteret procedure for installation, drift og fejlfinding af klyngen i Hjørring Kommune.

---

# 📘 Dokumentation

Den fulde dokumentation ligger i mappen **docs/** og publiceres via GitHub Pages.

Du kan læse dokumentationen her:

👉 https://hjk-automatisering.github.io/hjk-talos-cluster-setup/

---

# 📁 Dokumentationsstruktur

| Fil | Indhold |
|-----|---------|
| `01-environment.md` | Forberedelse af hardware, netværk, WSL2 og værktøjer |
| `02-bootstrap.md` | Installation af Talos, patches, apply-scripts og bootstrap |
| `03-cilium.md` | Installation af Cilium via Helm |
| `04-rook-ceph.md` | Installation af Rook Ceph (single-node) |
| `05-wsl-access.md` | Adgang til klyngen via WSL2 |
| `90-troubleshooting.md` | Fejlfinding for Talos, Kubernetes, Cilium og Ceph |

---

# Scripts og konfigurationer

Mappen `Talos/Bash/` indeholder:

- Bash-scripts til installation, apply og bootstrap  
- Helm charts og values-filer til Cilium og Rook Ceph  
- YAML-patches til Talos-konfiguration  
- kubeconfig- og talosconfig-filer  
- Fejlfinding og historik (`history.txt` og `history2.txt`)

Disse scripts danner grundlaget for dokumentationens beskrivelser og workflows.

---

# Forudsætninger

Før installation skal følgende være på plads:

- Udviklermaskine med **Windows 11 + WSL2 (Ubuntu)**  
- Talos CLI (`talosctl`)  
- Kubernetes CLI (`kubectl`)  
- Helm  
- Cilium CLI  
- Netværk med adgang til serverens statiske IP  
- iDRAC adgang til at mounte Talos ISO  
- Grundlæggende kendskab til Kubernetes og YAML  

---

# Kom hurtigt i gang

1. Læs **01 – Miljøforberedelse**  
2. Følg **02 – Installation og bootstrap af Talos**  
3. Installér Cilium og Ceph  
4. Deploy OS2AI  
5. Deploy Memoctopus MVP  

Når clusteret kører, kan du administrere det via:

```bash
talosctl version
kubectl get nodes
helm ls -A
```

---

# Drift og fejlfinding

Talos giver stærke værktøjer til diagnosticering:

```bash
talosctl health
talosctl logs <service>
talosctl dmesg
```

Kubernetes:

```bash
kubectl get pods -A
kubectl describe pod <pod>
```

Se **90-troubleshooting.md** for detaljer.

