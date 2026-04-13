# pve-ansible-setup

Proxmox VE 9 – Unattended Install ISO Builder & Cluster Base Setup via Ansible

## Übersicht

Zwei Playbooks:

| Playbook | Zweck |
|---|---|
| `01_build_iso.yml` | Erzeugt eine unattended Installations-ISO via `proxmox-auto-install-assistant` |
| `02_base_setup.yml` | Basis-Einrichtung (Repos, Pakete, lldpd, Netzwerk, SDN) + Cluster-Init |

## Projektstruktur

```
pve-ansible-setup/
├── ansible.cfg
├── requirements.yml
├── 01_build_iso.yml
├── 02_base_setup.yml
├── inventory/
│   └── hosts.yml              # Cluster-Gruppen (pve_cluster_alpha, pve_cluster_beta)
├── host_vars/
│   ├── pve-alpha-01.yml       # IPs, Bond-Slaves, Storage-VLAN-Bridges pro Node
│   ├── pve-alpha-02.yml
│   ├── pve-alpha-03.yml
│   ├── pve-beta-01.yml
│   └── pve-beta-02.yml
├── group_vars/
│   ├── all.yml                # Globale Einstellungen
│   └── proxmox_nodes.yml      # Bond/Bridge-Defaults, lldpd, NTP, SDN
└── roles/
    ├── pve_iso_builder/       # answer.toml.j2, firstboot.sh.j2
    ├── pve_base_setup/        # repos, packages, lldpd, network, sdn
    └── pve_cluster_init/      # pvecm create + pvecm add (idempotent)
```

## Netzwerk-Topologie

| Bond | Slaves | `lacp_rate` | Bridge | Zweck |
|---|---|---|---|---|
| `bond0` | eno1 + eno2 | **fast** ⚠️ | `vmbr0` | Management + Corosync |
| `bond1` | ens1f0 + ens1f1 | **fast** | `vmbr10/11/12` | NFS, iSCSI-1, iSCSI-2 (via VLANs) |
| `bond2` | ens2f0 + ens2f1 | **fast** | `vmbrData` | VM-Traffic / SDN VLAN-aware |

> **Wichtig:** `bond-lacp-rate fast` ist für alle Bonds gesetzt.  
> Für Corosync-Bonds ist dies **Pflicht** – sonst 90s Failover → HA-Fencing!  
> Quelle: https://pve.proxmox.com/wiki/Cluster_Manager#pvecm_corosync_over_bonds  
> **Auch am Switch muss LACP fast rate aktiviert werden!**

## Schnellstart

```bash
# 1. Collections installieren
ansible-galaxy collection install -r requirements.yml

# 2. Root-Passwort-Hash erzeugen
mkpasswd --method=yescrypt
# → In group_vars/all.yml unter pve_root_password_hashed eintragen

# 3. Disk-Filter ermitteln
proxmox-auto-install-assistant device-info -t disk
# → In group_vars/all.yml anpassen (pve_disk_filter_*) oder
#   in host_vars/<node>.yml als pve_disk_list: ["sda", "sdb"]

# 4. SSH Public Key eintragen
# → group_vars/all.yml: pve_root_ssh_keys

# 5. ISO bauen
ansible-playbook 01_build_iso.yml

# 6. ISO auf USB/IPMI laden, Node booten (vollautomatisch)

# 7. Basis-Setup + Cluster
ansible-playbook 02_base_setup.yml
# oder einzelner Cluster:
ansible-playbook 02_base_setup.yml --limit pve_cluster_alpha
```

## Neuen Node zu einem Cluster hinzufügen

Node in `inventory/hosts.yml` unter der Cluster-Gruppe eintragen (`pve_cluster_primary: false`),  
entsprechende `host_vars/<node>.yml` anlegen, dann:

```bash
ansible-playbook 02_base_setup.yml --limit <neuer-node>
```

## Sicherheit

- `pve_api_password` und `pve_root_password_hashed` mit **ansible-vault** verschlüsseln:
  ```bash
  ansible-vault encrypt_string 'geheimesPasswort' --name 'pve_api_password'
  ```
- SSH-Keys statt Passwörter für root-Login (`pve_ssh_permit_root_login: prohibit-password`)
