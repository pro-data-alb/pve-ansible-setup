# pve-ansible-setup

Proxmox VE 9 – Vollautomatischer Unattended Install + Cluster-Setup via Ansible

## Übersicht

| Playbook | Zweck |
|---|---|
| `00_setup_distrobox.yml` | Debian-13-Distrobox als `iso_builder` vorbereiten (distributions-agnostisch) |
| `01_build_iso.yml` | Baut eine Unattended-ISO via `proxmox-auto-install-assistant` |
| `02_base_setup.yml` | Node-Setup (Repos, Netzwerk, SDN) + Cluster-Init |

---

## Voraussetzungen – `iso_builder` System

Der ISO-Builder läuft auf `localhost` (Ansible-Controller) mit
`ansible_connection: local`. Das bedeutet: **alle Abhängigkeiten müssen auf
dem Rechner installiert sein, von dem aus das Playbook gestartet wird** –
nicht auf den PVE-Nodes.

### Betriebssystem

| Anforderung | Details |
|---|---|
| **OS** | Debian 12 (Bookworm) oder Ubuntu 22.04/24.04 LTS empfohlen |
| **Arch** | `amd64` – `proxmox-auto-install-assistant` ist nur für x86_64 verfügbar |
| **Nicht unterstützt** | macOS, Windows (WSL2 geht, aber ungetestet) |

> **Hinweis:** Das Playbook prüft `ansible_os_family == "Debian"` und
> bricht bei anderen OS-Familien ab.

### Ansible-Version

```bash
ansible --version
# Mindestanforderung: core 2.15+
# Empfohlen:          core 2.17+
```

Collections werden automatisch aus `requirements.yml` installiert:

```bash
ansible-galaxy collection install -r requirements.yml
```

Installierte Collections nach dem Befehl:

| Collection | Wofür |
|---|---|
| `community.proxmox` | Vorbereitet für VM-Provisioning (noch nicht aktiv für Netzwerk) |
| `ansible.posix` | `authorized_key`-Modul für SSH-Key-Austausch im Cluster-Join |

### Systempakete auf dem iso_builder Host

Das Playbook installiert fehlende Pakete via `apt` automatisch (`become: true`).
Voraussetzung ist jedoch, dass das **APT-Repository für `proxmox-auto-install-assistant`**
erreichbar ist.

#### Auf einem Proxmox VE Node (empfohlen)

Das Paket ist im Standard-PVE-Repo enthalten – keine zusätzliche Konfiguration nötig.

#### Auf einem regulären Debian/Ubuntu System (Controller ist kein PVE-Node)

Das Paket `proxmox-auto-install-assistant` stammt aus dem Proxmox-Repo und ist
**nicht** in den Standard-Debian/Ubuntu-Quellen enthalten. APT-Quelle manuell
hinzufügen:

```bash
# Proxmox GPG-Key importieren
curl -fsSL https://enterprise.proxmox.com/debian/proxmox-release-bookworm.gpg \
  | sudo gpg --dearmor -o /etc/apt/trusted.gpg.d/proxmox-release-bookworm.gpg

# No-Subscription-Repo hinzufügen (kein Proxmox-Account nötig)
echo "deb http://download.proxmox.com/debian/pve bookworm pve-no-subscription" \
  | sudo tee /etc/apt/sources.list.d/pve-no-subscription.list

sudo apt update
sudo apt install proxmox-auto-install-assistant wget whois
```

> Für Ubuntu 24.04 (`noble`) analog mit `noble` statt `bookworm`.
> Prüfe die aktuellen Repo-URLs unter: https://pve.proxmox.com/wiki/Package_Repositories

#### Auf Fedora / openSUSE / Arch / beliebiger Distribution – via Distrobox

Für Nicht-Debian-Systeme bietet sich eine **Debian-13-Distrobox** als
`iso_builder`-Umgebung an. Ansible **und** `proxmox-auto-install-assistant`
laufen dann vollständig im Container – `ansible_connection: local` macht
SSH überflüssig. Das Setup-Playbook `00_setup_distrobox.yml` automatisiert
den gesamten Prozess.

> Distrobox mounted `$HOME` des Hosts automatisch in den Container –
> die fertige ISO ist ohne manuelles Volume-Mapping sofort auf dem Host sichtbar.

Details und Schritt-für-Schritt-Anleitung: siehe **[Distrobox-Setup](#distrobox-setup-auf-beliebiger-distribution)** am Ende dieses Dokuments.

#### Manuelle Vorabinstallation prüfen

```bash
# Prüfen ob alle benötigten Pakete vorhanden sind:
dpkg -l proxmox-auto-install-assistant wget whois 2>/dev/null | grep '^ii'

# Verify: Tool funktioniert
proxmox-auto-install-assistant --version
```

### Python-Anforderungen

```bash
# Mindestversion: Python 3.9+
python3 --version

# Benötigte Stdlib-Module (immer vorhanden):
# - json, os, subprocess, tempfile
# Keine zusätzlichen pip-Pakete erforderlich.
```

### Netzwerk / Connectivity

Das Playbook braucht folgende ausgehende Verbindungen vom iso_builder Host:

| Ziel | Port | Wofür | Pflicht? |
|---|---|---|---|
| `enterprise.proxmox.com` | 443/TCP | ISO-Download (`pve_iso_url`) | Nur wenn `pve_iso_download: true` |
| `download.proxmox.com` | 80/TCP | APT-Repo für Paketinstall | Nur wenn Pakete fehlen |
| DNS-Auflösung | 53/UDP+TCP | Hostnamen auflösen | Ja |

Wenn der Controller hinter einem **HTTP-Proxy** liegt:

```bash
# Vor dem Playbook-Aufruf setzen:
export http_proxy="http://proxy.example.local:3128"
export https_proxy="http://proxy.example.local:3128"
export no_proxy="localhost,127.0.0.1,192.168.0.0/16"
```

Wenn der ISO-Download nicht möglich ist (Air-Gap), ISO manuell ablegen und
Download deaktivieren:

```yaml
# group_vars/all.yml
pve_iso_download: false
pve_iso_output_dir:  "/srv/isos"
pve_iso_filename:    "proxmox-ve_9.1-1.iso"   # muss dort liegen
```

### Ausgabeverzeichnis

```bash
# Default: /tmp/pve-iso-build  (aus group_vars/all.yml → pve_iso_output_dir)
# Das Verzeichnis muss:
#   - existieren ODER durch den Ansible-User mit become: false anlegbar sein
#   - min. ~6 GB freien Speicher haben (Quell-ISO ~1.4 GB + Output-ISO pro Node)
df -h /tmp
```

---

### Variablen-Checkliste vor `01_build_iso.yml`

Das Playbook prüft Pflichtfelder in `pre_tasks` mit `assert` und bricht
mit einer klaren Fehlermeldung ab, wenn etwas fehlt.

#### Pflicht – Playbook bricht ohne diese Werte ab

| Variable | Datei | Beispielwert | Beschreibung |
|---|---|---|---|
| `pve_root_password_hashed` | `group_vars/all.yml` | `$y$j9T$...` | yescrypt-Hash, **nicht** Klartext |
| `pve_root_ssh_keys` | `group_vars/all.yml` | `["ssh-ed25519 AAAA..."]` | Mind. 1 SSH Public Key |
| `pve_iso_url` | `group_vars/all.yml` | `https://enterprise.proxmox.com/iso/proxmox-ve_9.1-1.iso` | Quell-ISO URL |
| `pve_iso_filename` | `group_vars/all.yml` | `proxmox-ve_9.1-1.iso` | Dateiname der Quell-ISO |

#### Pflicht pro Node – in `host_vars/<hostname>.yml`

| Variable | Beispielwert | Beschreibung |
|---|---|---|
| `pve_node_ip` | `192.168.10.11` | Statische Management-IP nach Install |
| `pve_node_cidr_prefix` | `24` | Prefix-Länge (nicht Netmask) |
| `pve_node_gw` | `192.168.10.1` | Default Gateway |
| `pve_node_dns` | `192.168.10.1` | DNS-Server |
| `pve_nic_aliases` | siehe unten | MAC → Alias → Bond-Rolle (bond0/bond1/bond2) |

```yaml
# Minimales host_vars-Beispiel für ISO-Bau:
pve_node_ip:          "192.168.10.11"
pve_node_cidr_prefix: "24"
pve_node_gw:          "192.168.10.1"
pve_node_dns:         "192.168.10.1"

pve_nic_aliases:
  - mac:   "aa:bb:cc:01:01:01"   # Management NIC 1 (aus IPMI/LLDP)
    alias: "node01-mgmt-a"
    role:  "bond0"
  - mac:   "aa:bb:cc:01:01:02"   # Management NIC 2
    alias: "node01-mgmt-b"
    role:  "bond0"
```

#### Optional – sinnvoll anzupassen

| Variable | Default | Beschreibung |
|---|---|---|
| `pve_disk_filesystem` | `zfs` | `zfs`, `ext4`, `xfs`, `btrfs` |
| `pve_disk_zfs_raid` | `raid1` | `raid0`, `raid1`, `raid10`, `raidz`, `raidz2`, `raidz3` |
| `pve_disk_filter_key` | `ID_MODEL` | udev-Attribut für Disk-Auswahl |
| `pve_disk_filter_value` | `Samsung*` | Glob-Pattern, z. B. `SAMSUNG*` oder `WD*` |
| `pve_disk_list` | nicht gesetzt | Explizite Liste: `["sda","sdb"]` – überschreibt Filter |
| `pve_iso_download` | `true` | `false` = ISO bereits vorhanden |
| `pve_iso_validate_answer` | `true` | `false` = answer.toml-Validierung überspringen |
| `pve_answer_fetch_from` | `iso` | `iso` \| `partition` \| `http` |
| `pve_timezone` | `Europe/Berlin` | Zeitzone des installierten Systems |
| `pve_keyboard` | `de` | Tastaturlayout |
| `pve_mailto` | `admin@example.local` | E-Mail für PVE-Benachrichtigungen |

#### Disk-Filter richtig setzen

Der `answer.toml`-Installer wählt Ziel-Disks anhand eines udev-Attributs aus.
Den korrekten Wert ermitteln **am laufenden System** (oder via Rescue-OS):

```bash
# Alle relevanten udev-Attribute einer Disk anzeigen:
udevadm info --query=all --name=/dev/sda | grep -E "ID_MODEL|ID_SERIAL|ID_VENDOR|ID_PATH"

# Beispielausgabe:
# E: ID_MODEL=Samsung_SSD_870
# E: ID_SERIAL=S5ESNX0R123456
# E: ID_VENDOR=Samsung

# Disk-Devices auf dem System:
proxmox-auto-install-assistant device-info -t disk
```

Bei **unbekannten oder gemischten Disks** ist die explizite Liste sicherer:

```yaml
# host_vars/<hostname>.yml
pve_disk_list: ["sda", "sdb"]   # Überschreibt pve_disk_filter_*
```

---

### Schritt-für-Schritt: ISO-Builder vorbereiten

```bash
# 1. Repo klonen
git clone https://github.com/pro-data-alb/pve-ansible-setup.git
cd pve-ansible-setup

# 2. Collections installieren
ansible-galaxy collection install -r requirements.yml

# 3. proxmox-auto-install-assistant verfügbar? (falls nicht: Repo oben hinzufügen)
proxmox-auto-install-assistant --version

# 4. Root-Passwort-Hash erzeugen
mkpasswd --method=yescrypt
# Ausgabe (Beispiel): $y$j9T$abcdef...
# → in group_vars/all.yml: pve_root_password_hashed: "$y$j9T$abcdef..."

# 5. SSH Public Key des Controllers eintragen
cat ~/.ssh/id_ed25519.pub
# → in group_vars/all.yml: pve_root_ssh_keys: ["ssh-ed25519 AAAA..."]

# 6. host_vars für jeden Node anlegen (IP + MACs)
cp host_vars/pve-alpha-01.yml host_vars/pve-alpha-02.yml
# ... anpassen

# 7. Sensitive Variablen verschlüsseln (Produktion)
ansible-vault encrypt_string '$y$j9T$...' --name 'pve_root_password_hashed'
# → Ausgabe direkt in group_vars/all.yml einfügen

# 8. Dry-Run (prüft Templates + Variablen, baut keine ISO)
ansible-playbook 01_build_iso.yml --check

# 9. ISO bauen
ansible-playbook 01_build_iso.yml
# Mit Vault-Passwort:
ansible-playbook 01_build_iso.yml --ask-vault-pass
```

---

### Troubleshooting – häufige Fehlerquellen

| Fehlermeldung | Ursache | Lösung |
|---|---|---|
| `pve_root_password_hashed fehlt oder ist noch Platzhalter` | `CHANGEME` noch im Hash | `mkpasswd --method=yescrypt` ausführen |
| `pve_root_ssh_keys enthält keinen SSH Public Key` | Liste leer | Public Key aus `~/.ssh/id_ed25519.pub` eintragen |
| `E: Package 'proxmox-auto-install-assistant' not found` | PVE-APT-Repo fehlt | Repo-Anleitung oben folgen |
| `Quell-ISO nicht gefunden` | Download fehlgeschlagen oder `pve_iso_download: false` ohne ISO | URL prüfen oder ISO manuell ablegen |
| `validate-answer: ERROR` | Ungültige `answer.toml` | Disk-Filter, IPs oder Pflichtfelder in `host_vars` prüfen |
| `No space left on device` | `/tmp` zu voll | `pve_iso_output_dir: "/srv/isos"` auf größere Partition umleiten |
| `proxmox-auto-install-assistant: command not found` | Paket fehlt trotz `apt install` | `which proxmox-auto-install-assistant` und `$PATH` prüfen |

---

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
├── 00_setup_distrobox.yml        # Distrobox iso-builder vorbereiten
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

Für jeden Node `host_vars/<hostname>.yml` anlegen:

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
# Erzeugt pro Node eine ISO unter /tmp/pve-iso-build/
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

---

## Distrobox-Setup auf beliebiger Distribution

Für alle Nicht-Debian-Systeme (Fedora, openSUSE, Arch, etc.) stellt
`00_setup_distrobox.yml` eine vollautomatische Einrichtung einer
Debian-13-Distrobox als `iso_builder`-Umgebung bereit.

### Funktionsweise

| Schritt | Was passiert |
|---|---|
| Container anlegen | `debian:trixie` via distrobox create |
| Basispakete | `wget`, `gpg`, `python3`, `ansible`, `git`, `whois` via apt |
| Proxmox GPG-Key | Download nach `/usr/share/keyrings/` im Container |
| Proxmox-Repo | `pve-no-subscription` (Trixie) via deb822-Format |
| `proxmox-auto-install-assistant` | apt install + Smoke-Test |
| inventory patch | `ansible_connection: local` + `ansible_python_interpreter` |
| Collections | `ansible-galaxy collection install -r requirements.yml` im Container |

### Voraussetzungen auf dem Host

```bash
# Fedora
sudo dnf install -y distrobox podman ansible

# openSUSE
sudo zypper install -y distrobox podman ansible

# Arch
sudo pacman -S distrobox podman ansible

# Debian/Ubuntu (falls native Umgebung nicht gewünscht)
sudo apt install -y distrobox podman ansible
```

### Ausführung

```bash
# 1. Repo klonen
git clone https://github.com/pro-data-alb/pve-ansible-setup.git
cd pve-ansible-setup

# 2. Distrobox + Proxmox-Umgebung vollautomatisch einrichten
ansible-playbook 00_setup_distrobox.yml

# 3. In die Box wechseln
distrobox enter iso-builder

# 4. ISO bauen (ab hier alles im Container)
ansible-playbook 01_build_iso.yml
```

> **Hinweis:** Distrobox mounted `$HOME` des Hosts automatisch.
> Die fertige ISO unter `pve_iso_output_dir` (Default: `/tmp/pve-iso-build/`)
> ist ohne weiteres Volume-Mapping direkt auf dem Host sichtbar.

### Annahmen des Playbooks

- Das Repo liegt im `$HOME` des ausführenden Users
- Der User hat im Distrobox-Container passwordless `sudo` (Distrobox-Default)
- Internet-Zugang für `download.proxmox.com` und `enterprise.proxmox.com` ist vorhanden
- `ansible_connection: local` ist nach dem Setup in `inventory/hosts.yml` gesetzt

### Troubleshooting Distrobox

| Fehlermeldung | Ursache | Lösung |
|---|---|---|
| `distrobox: command not found` | distrobox nicht installiert | `dnf/apt/zypper install distrobox` |
| `No container runtime found` | podman/docker fehlt | `dnf install podman` |
| `Error: image not found: debian:trixie` | Image noch nicht lokal | distrobox pulled automatisch; Netz prüfen |
| `sudo: command not found` | Container noch nicht initialisiert | `distrobox enter iso-builder -- true` einmal ausführen |
| `proxmox-auto-install-assistant: not found` | Proxmox-Repo nicht eingerichtet | `00_setup_distrobox.yml` erneut ausführen |

---

## Quick-Reference – Distrobox auf beliebiger Distribution

```bash
# Voraussetzungen (einmalig, je nach Distribution)
sudo dnf install -y distrobox podman ansible        # Fedora
# sudo zypper install -y distrobox podman ansible   # openSUSE
# sudo pacman -S distrobox podman ansible           # Arch

# Repo klonen
git clone https://github.com/pro-data-alb/pve-ansible-setup.git
cd pve-ansible-setup

# Distrobox + Proxmox-Umgebung einrichten (einmalig)
ansible-playbook 00_setup_distrobox.yml

# In die Box wechseln
distrobox enter iso-builder

# Root-Passwort-Hash erzeugen
mkpasswd --method=yescrypt
# → group_vars/all.yml: pve_root_password_hashed: "$y$j9T$..."

# SSH Public Key eintragen
cat ~/.ssh/id_ed25519.pub
# → group_vars/all.yml: pve_root_ssh_keys: ["ssh-ed25519 AAAA..."]

# host_vars für jeden Node anlegen
cp host_vars/pve-alpha-01.yml host_vars/<mein-node>.yml
# → IPs und MAC-Adressen eintragen

# ISO bauen
ansible-playbook 01_build_iso.yml
```
