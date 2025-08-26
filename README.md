
# ansible-secure-multivm

Petit labo Ansible pour automatiser un parc de 3 VMs (Ubuntu 24.04) :

* **Logs centralisés** (rsyslog)
* **Mises à jour sécurité automatiques**
* **Scans hebdo** sécurité **OVAL** (vulnérabilités Canonical) + **XCCDF/SSG** (CIS) avec rapports HTML rapatriés

## 1) Objectif

Avoir une base “ops + sécu” reproductible : visibilité (logs), patching automatique, preuves de conformité (rapports HTML) et scripts de remédiation.

## 2) Architecture

* **Controller** : ta machine (Ansible)
* **logserver** : `log1` (réception des logs)
* **servers** : `srv1`, `srv2` (envoient les logs, patch + scans)

Inventaire minimal (`inventaire.ini`) :

```ini
[logserver]
log1 ansible_host=<IP_LOG1> ansible_user=<SSH_USER> ansible_ssh_private_key_file=~/.ssh/id_ed25519

[servers]
srv1 ansible_host=<IP_SRV1> ansible_user=<SSH_USER> ansible_ssh_private_key_file=~/.ssh/id_ed25519
srv2 ansible_host=<IP_SRV2> ansible_user=<SSH_USER> ansible_ssh_private_key_file=~/.ssh/id_ed25519
```

## 3) Prérequis

* Ansible installé sur le controller
* Accès SSH par clé vers les VMs
* Python présent sur les VMs (pour `ansible -m ping`)

## 4) Démarrage rapide

Depuis la racine du dépôt :

```bash
ansible-inventory --graph

# Module C : centralisation des logs
ansible-playbook 20-logging.yml
# test
ansible srv1 -m shell -a "logger 'hello-central-log'"
ansible log1 -b -m shell -a "grep -R 'hello-central-log' /var/log/remote || echo not-found"

# Module D : mises à jour sécurité automatiques (Ubuntu)
ansible-playbook 30-updates.yml

# Module E : scans OVAL + XCCDF (installe un job weekly + exécute une fois + rapatrie HTML)
ansible-playbook 40-openscap.yml
```

## 5) Modules (aperçu)

* **C — rsyslog**

  * `log1` écoute TCP/514 et écrit sous `/var/log/remote/<host>/messages.log`
  * `srv1/srv2` envoient `*.*` vers `@@log1:514` (TCP)
* **D — Auto updates**

  * `unattended-upgrades` activé (Ubuntu)
* **E — Scans sécurité**

  * **OVAL** : télécharge le flux Canonical (codename) et génère `oval-<codename>-YYYY-MM-DD.html`
  * **XCCDF/SSG** : télécharge **SSG v0.1.77 (zip)** sous `/opt/ssg`, sélectionne le datastream Ubuntu (**2404 → 2204 → 2004**), lance le profil **CIS Level 2 Server**, produit `xccdf-YYYY-MM-DD.html`, résultats XML (ARF) et un script de **remédiation** `fix-YYYY-MM-DD.sh`
  * Un **cron weekly** (`/etc/cron.weekly/openscap-both`) exécute OVAL + XCCDF chaque semaine

## 6) Rapports (où regarder)

* **Sur les VMs** : `/var/log/openscap/`

  * `oval-<codename>-YYYY-MM-DD.html` + `oval-latest.html`
  * `xccdf-YYYY-MM-DD.html` + `xccdf-latest.html`, `xcssf-results*.xml`, `xcssf-arf*.xml`, `fix-*.sh`
* **Sur le controller** : `reports/` (copie des `*-latest.html`)

> Le répertoire `reports/` est ignoré par Git (sauf `.gitkeep`).

## 7) Arborescence

```
ansible-secure-multivm/
├─ ansible.cfg
├─ inventaire.ini
├─ roles/
│  ├─ rsyslog_server/…
│  ├─ rsyslog_client/…
│  ├─ updates/…
│  └─ openscap/
│     └─ tasks/main.yml   # OVAL + XCCDF weekly (+ run now + fetch)
├─ 20-logging.yml
├─ 30-updates.yml
├─ 40-openscap.yml
└─ reports/.gitkeep
```

## 8) Dépannage rapide

* **Rien dans `/var/log/remote`** : vérifier port 514/tcp sur `log1`, noms des fichiers dans `/etc/rsyslog.d/`, relancer rsyslog.
* **Pas de rapport OVAL** : tester la connexion Internet depuis la VM (`curl -I https://security-metadata.canonical.com/oval/`), exécuter `sudo /etc/cron.weekly/openscap-both`.
* **Pas de rapport XCCDF** : vérifier la présence du datastream sous `/opt/ssg/scap-security-guide-0.1.77/` (fichier `ssg-ubuntu* -ds.xml`) et que le **profil** existe :

  ```bash
  oscap info /opt/ssg/.../ssg-ubuntuXXXX-ds.xml | sed -n '1,120p'
  ```

  (adapter `openscap_profile` si besoin, ex. `...cis_level1_server`)

## 9) Remarques

* Ce repo sert de **base** : à adapter pour la prod (rotation logs, timers systemd si requis, stockage centralisé des rapports, etc.).
* Les versions d’OS et des contenus SSG peuvent évoluer : le rôle choisit automatiquement le datastream Ubuntu dispo (24.04 → 22.04 → 20.04).


