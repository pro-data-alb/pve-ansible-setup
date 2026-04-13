# pve-ansible-setup

Proxmox VE 9 – Vollautomatischer Unattended Install + Cluster-Setup via Ansible

## Übersicht

| Playbook | Zweck |
|---|---|
| `01_build_iso.yml` | Baut eine Unattended-ISO via `proxmox-auto-install-assistant` |
| `02_base_setup.yml` | Node-Setup (Repos, Netzwerk, SDN) + Cluster-Init |

## Netzwerk-Strategie (dreistufig)

Interfacenamen (`eno1`, `ens1f0`, ...) sind hardware- und treiberabhängig und
im Vorfeld nicht zuverlässig bekannt. Daher werden **MAC-Adressen** als
stabile Referenz verwendet – auslesbar vor der Erstinstallation via IPMI oder
Switch-LLDP.

```
┌───────────────────────────────────────────────────────┐
│ Stufe 1 – Installer (answer.toml)                      │
│   source = "from-dhcp"                                 │
│   Erste erkannte NIC → DHCP → Installer hat Netz       │
└────────────────────────▼──────────────────────────────┘
                         │
┌────────────────────────▼──────────────────────────────┐
│ Stufe 2 – firstboot.sh (nach Install, vor 1. Reboot)   │
│   MAC → /sys/class/net → Interfacename                 │
│   udev-Regeln schreiben (stabiler Ifname nach Reboot)  │
│   bond0 (LACP fast) + vmbr0 (statische Mgmt-IP)       │
│   DHCP-Interface deaktivieren                         │
│   ⇒ Node per SSH auf pve_node_ip erreichbar            │
└────────────────────────▼──────────────────────────────┘
                         │ Reboot
┌────────────────────────▼──────────────────────────────┐
│ Stufe 3 – Ansible (02_base_setup.yml)                  │
│   bond1 (Storage), bond2 (VM-Traffic) via MAC          │
│   Storage-Bridges (VLAN), vmbrData                    │
│   SDN (VXLAN), Cluster-Init (pvecm)                   │
└───────────────────────────────────────────────────────┘
```

## Netzwerk-Topologie

| Bond | Slaves (via MAC) | `lacp_rate` | Bridge | Zweck |
|---|---|---|---|---|
| `bond0` | `bond0`-MACs aus `host_vars` | **fast** ⚠️ | `vmbr0` | Management + Corosync |
| `bond1` | `bond1`-MACs aus `host_vars` | **fast** | `vmbr10/11/12` | NFS, iSCSI (via VLANs) |
| `bond2` | `bond2`-MACs aus `host_vars` | **fast** | `vmbrData` | VM-Traffic / SDN VLAN-aware |

> **Wichtig:** `bond-lacp-rate fast` ist Pflicht für alle Corosync-Bonds.  
> Slow (default) = 90s Failover → HA-Fencing greift vor Link-Erkennung!  
> Quelle: https://pve.proxmox.com/wiki/Cluster_Manager#pvecm_corosync_over_bonds  
> **Switch-Ports müssen ebenfalls auf LACP fast konfiguriert sein!**

## Projektstruktur

```
pve-ansible-setup/
├── ansible.cfg
├── requirements.yml              # community.proxmox + ansible.posix
├── 01_build_iso.yml
├── 02_base_setup.yml
├── inventory/
│   └── hosts.yml
├── host_vars/
│   ├── pve-alpha-01.yml           # IPs, pve_nic_aliases (MAC+Alias+Role)
│   ├── pve-alpha-02.yml
│   └── pve-alpha-03.yml
├── group_vars/
│   ├── all.yml                    # Globale Einstellungen, Vault-Variablen
│   └── proxmox_nodes.yml          # Bond/Bridge-Defaults, lldpd, NTP, SDN
└── roles/
    ├── pve_iso_builder/           # answer.toml.j2, firstboot.sh.j2
    │   └── meta/main.yml
    ├── pve_base_setup/            # repos, packages, mac_resolve, network, sdn
    │   ├── meta/main.yml
    │   └── templates/
    │       └── 70-pve-nic-aliases.rules.j2
    └── pve_cluster_init/          # pvecm create + SSH-Key-basiertes pvecm add
        └── meta/main.yml
```

## Schnellstart

### 0. Voraussetzungen

```bash
# Collections installieren
ansible-galaxy collection install -r requirements.yml
```

### 1. MAC-Adressen vorab auslesen

Vor der Erstinstallation MACs aus IPMI oder Switch-LLDP auslesen:

```bash
# Via IPMI (remote, kein OS nötig):
ipmitool -H <bmc-ip> -U admin -P pass lan print 1 | grep 'MAC Address'

# Via Switch (LLDP-Tabelle):
lldpcli show neighbors  # auf einem bereits laufenden System

# Nach Erstinstall via DHCP (bevor bond0 aufgebaut ist):
ip link show | grep 'link/ether'
```

### 2. host_vars befüllen

Fur jeden Node `host_vars/<hostname>.yml` anlegen:

```yaml
pve_node_ip:          "192.168.10.11"
pve_node_cidr_prefix: "24"
pve_node_gw:          "192.168.10.1"
pve_node_dns:         "192.168.10.1"

pve_nic_aliases:
  - mac:   "aa:bb:cc:01:01:01"   # aus IPMI/LLDP
    alias: "node01-mgmt-a"        # frei wählbar
    role:  "bond0"                # bond0 | bond1 | bond2
  - mac:   "aa:bb:cc:01:01:02"
    alias: "node01-mgmt-b"
    role:  "bond0"
  # ... bond1 und bond2 NICs analog
```

### 3. Globale Einstellungen setzen

```bash
# Root-Passwort-Hash erzeugen (yescrypt, PVE 9 Standard)
mkpasswd --method=yescrypt
# → In group_vars/all.yml unter pve_root_password_hashed eintragen

# SSH Public Key des Ansible-Controllers eintragen
# → group_vars/all.yml: pve_root_ssh_keys
```

### 4. ISO bauen

```bash
ansible-playbook 01_build_iso.yml
# Erzeugt pro Node eine ISO unter /tmp/pve-<hostname>.iso
```

### 5. ISO flashen und Node booten

ISO via IPMI Virtual Media oder USB laden und booten.
Der Installer läuft vollautomatisch durch.
`firstboot.sh` baut danach bond0/vmbr0 auf (MAC-basiert).
Nach Reboot ist der Node auf `pve_node_ip` per SSH erreichbar.

### 6. Basis-Setup + Cluster

```bash
# Alle Nodes eines Clusters:
ansible-playbook 02_base_setup.yml --limit pve_cluster_alpha

# Einzelner Node (z. B. Nachzügler):
ansible-playbook 02_base_setup.yml --limit pve-alpha-03
```

## Neuen Node hinzufügen

1. `host_vars/<neuer-node>.yml` anlegen (IP + MAC-Aliase)
2. Node in `inventory/hosts.yml` unter der Cluster-Gruppe eintragen
3. ISO bauen, flashen, booten
4. Playbook ausführen:

```bash
ansible-playbook 02_base_setup.yml --limit <neuer-node>
```

## Sicherheit

### Ansible Vault (Pflicht für Produktion)

```bash
# Einzelnen Wert verschlüsseln:
ansible-vault encrypt_string 'GeheimesPasswort' --name 'pve_api_password'

# Ganze Datei verschlüsseln:
ansible-vault encrypt group_vars/all.yml

# Playbook mit Vault-Passwort ausführen:
ansible-playbook 02_base_setup.yml --ask-vault-pass
# oder:
ansible-playbook 02_base_setup.yml --vault-password-file ~/.vault_pass
```

Folgende Variablen müssen verschlüsselt werden:
- `pve_api_password`
- `pve_root_password_hashed`

### SSH

- Root-Login nur per SSH-Key (`pve_ssh_permit_root_login: prohibit-password`)
- Passwort-Auth deaktiviert (`pve_ssh_password_auth: no`)
- Cluster-Join via pre-shared SSH-Keys (kein `expect`, kein Passwort-Prompt)

## Bekannte Einschränkungen

- `community.proxmox` Collection wird in `requirements.yml` referenziert,
  aber die Netzwerk-Konfiguration erfolgt aktuell direkt via REST API
  (`ansible.builtin.uri`). Für VM-Provisioning kann `community.proxmox.proxmox_kvm`
  später ergänzt werden.
- udev-Regeln benötigen einen Reboot um auf den Interfacenamen wirksam zu sein.
  `mac_resolve.yml` liest die aktuellen Namen direkt aus `/sys/class/net`
  (kein Reboot nötig für den ersten Ansible-Lauf).
- Switch-Ports müssen vor dem Booten im LACP-Trunk-Mode sein,
  sonst bleibt bond0 im degraded-Mode bis zum nächsten LACPDU-Zyklus.
