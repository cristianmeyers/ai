# 🖥️ Proxmox — Création de la VM `lnx`

Ce document détaille la création de la machine virtuelle **`lnx`** sur l'hyperviseur **Proxmox VE**. Cette VM sert de socle pour l'hébergement des services Docker (Traefik, n8n, Open WebUI).

---

## 📋 Prérequis

- Accès à l'interface web Proxmox (`https://<IP_PROXMOX>:8006`).
- Une image ISO ou template Ubuntu Server LTS disponible dans le stockage Proxmox.
- Un stockage (local-lvm, ZFS, etc.) avec suffisamment d'espace libre.
- Un réseau/bridge configuré (ex: `vmbr0`) avec accès au réseau local.

---

## 🛠️ 1. Création de la VM

Depuis l'interface Proxmox : **Datacenter → [Nœud] → Create VM**.

### 1.1 Général

| Paramètre | Valeur               |
| --------- | -------------------- |
| Node      | `<NOM_NODE_PROXMOX>` |
| VM ID     | `<VMID>`             |
| Name      | `lnx`                |

### 1.2 OS

| Paramètre            | Valeur                                             |
| -------------------- | -------------------------------------------------- |
| ISO image / Template | `<ubuntu-server-XX.XX.iso>` ou template cloud-init |
| Type                 | Linux                                              |
| Version              | 6.x - 2.6 Kernel                                   |

### 1.3 Système

| Paramètre       | Valeur                              |
| --------------- | ----------------------------------- |
| Machine         | `q35`                               |
| BIOS            | `OVMF (UEFI)` ou `SeaBIOS`          |
| Add EFI Disk    | Coché si UEFI                       |
| SCSI Controller | `VirtIO SCSI single`                |
| Qemu Agent      | Coché (activer le QEMU Guest Agent) |

### 1.4 Disques

| Paramètre     | Valeur                               |
| ------------- | ------------------------------------ |
| Bus/Device    | `SCSI` (VirtIO)                      |
| Storage       | `<STOCKAGE_PROXMOX>`                 |
| Disk size     | `<TAILLE_DISQUE_GO>` Go              |
| Cache         | `Default (No cache)` ou `Write back` |
| Discard       | Coché (si SSD/thin provisioning)     |
| SSD emulation | Coché si applicable                  |

### 1.5 CPU

| Paramètre | Valeur      |
| --------- | ----------- |
| Sockets   | `1`         |
| Cores     | `<NB_VCPU>` |
| Type      | `host`      |

### 1.6 Mémoire

| Paramètre    | Valeur                              |
| ------------ | ----------------------------------- |
| Memory (RAM) | `<RAM_GO>` Go                       |
| Ballooning   | Désactivé (ou min/max selon besoin) |

### 1.7 Réseau

| Paramètre | Valeur                      |
| --------- | --------------------------- |
| Bridge    | `vmbr0`                     |
| Model     | `VirtIO (paravirtualized)`  |
| VLAN Tag  | `<VLAN_ID>` (si applicable) |
| Firewall  | Coché (optionnel)           |

---

## 🚀 2. Démarrage et installation

1. Démarrer la VM depuis l'interface Proxmox (**Start**).
2. Ouvrir la console (**Console → noVNC** ou **xterm.js**).
3. Suivre l'installateur Ubuntu Server :
   - Configuration réseau (IP statique recommandée pour `lnx`).
   - Création de l'utilisateur administrateur.
   - Activer **OpenSSH server** pendant l'installation.
4. Une fois l'installation terminée, redémarrer la VM et retirer le support d'installation (ISO) des paramètres du matériel.

---

## ⚙️ 3. Post-installation

### 3.1 Installer le QEMU Guest Agent (si non fait à l'installation)

```bash
sudo apt update
sudo apt install -y qemu-guest-agent
sudo systemctl enable --now qemu-guest-agent
```

### 3.2 Vérifier l'accès SSH

```bash
ssh <utilisateur>@<IP_VM_LNX>
```

### 3.3 Mise à jour du système

```bash
sudo apt update && sudo apt upgrade -y
```

---

## 📝 4. Notes

- L'adresse IP attribuée à `lnx` doit être fixée (DHCP réservation ou configuration statique), car elle sera référencée par les autres services et par Traefik.
- Une fois la VM créée et accessible en SSH, la suite de la configuration (Docker, Traefik, n8n, Open WebUI) est documentée dans les sous-dossiers `lnx/`, `n8n/` et `open-webui/`.
