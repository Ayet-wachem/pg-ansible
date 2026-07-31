# pg-ansible — Framework Ansible de déploiement PostgreSQL (Standalone & Haute Disponibilité)

Framework Ansible modulaire pour déployer et gérer PostgreSQL 17 de façon
**reproductible**, **multi-distributions** et **idempotente**, en architecture
autonome (standalone) ou en **haute disponibilité** (réplication + failover
automatique + adresse virtuelle flottante).

Projet réalisé dans le cadre d'un stage chez PROXYM.

---

## Sommaire

1. [Fonctionnalités](#1-fonctionnalités)
2. [Architecture](#2-architecture)
3. [Prérequis](#3-prérequis)
4. [Structure du projet](#4-structure-du-projet)
5. [Installation pas à pas](#5-installation-pas-à-pas)
6. [Guide de tests](#6-guide-de-tests)
7. [Dépannage](#7-dépannage)
8. [Limites connues](#8-limites-connues)

---

## 1. Fonctionnalités

| # | Fonctionnalité | Description |
|---|----------------|-------------|
| 1 | Vérifications préalables (preflight) | Contrôle OS, RAM, disque, ports, résolution de noms avant tout déploiement |
| 2 | Version PostgreSQL configurable | Variable `pg_version` (17 par défaut ; testé avec 16) |
| 3 | Architecture Standalone | Instance PostgreSQL autonome |
| 4 | Architecture Haute Disponibilité | Master + Standby + VIP |
| 5 | Réplication streaming | Réplication en flux temps réel master → standby |
| 6 | Failover automatique | Promotion automatique du standby via repmgrd |
| 7 | Adresse virtuelle (VIP) | VIP flottante gérée par Keepalived (VRRP) |
| 8 | Validation post-déploiement | Rapport de conformité horodaté |
| 9 | Secrets chiffrés | Mots de passe gérés par Ansible Vault (AES-256) |
| 10 | Multi-distributions | AlmaLinux 9 (RHEL) et Ubuntu 22.04 (Debian) |
| 11 | Installation air-gapped | Déploiement sans accès Internet via dépôt local |
| 12 | Architecture extensible | Rôles modulaires réutilisables |

---

## 2. Architecture

### Topologie du laboratoire

Réseau interne : **172.16.50.0/24** (interface `enp0s8`).

| Rôle | Hôte | IP | OS |
|------|------|----|----|
| Contrôleur Ansible | pilote | 172.16.50.10 | AlmaLinux 9 |
| Master (cluster Alma) | pg-alma-master | 172.16.50.180 | AlmaLinux 9 |
| Standby (cluster Alma) | pg-alma-slave | 172.16.50.181 | AlmaLinux 9 |
| Standalone Alma | pg-alma-standalone | 172.16.50.182 | AlmaLinux 9 |
| Master (cluster Ubuntu) | pg-ubuntu-master | 172.16.50.185 | Ubuntu 22.04 |
| Standby (cluster Ubuntu) | pg-ubuntu-slave | 172.16.50.186 | Ubuntu 22.04 |
| Standalone Ubuntu | pg-ubuntu-standalone | 172.16.50.187 | Ubuntu 22.04 |
| **VIP cluster Alma** | — | **172.16.50.190** | — |
| **VIP cluster Ubuntu** | — | **172.16.50.191** | — |

### Schéma d'un cluster HA

```
                 VIP 172.16.50.190
                        │
              ┌─────────┴─────────┐
              │  (suit le master) │
              ▼                   ▼
     ┌────────────────┐  ┌────────────────┐
     │  MASTER (.180) │  │  STANDBY (.181)│
     │  PostgreSQL 17 │─▶│  PostgreSQL 17 │
     │  primary       │  │  recovery      │
     │  repmgrd       │  │  repmgrd       │
     │  keepalived    │  │  keepalived    │
     └────────────────┘  └────────────────┘
       réplication streaming (temps réel)

  En cas de panne du master :
  1. repmgrd (standby) détecte l'absence du master
  2. après 3 tentatives, promeut le standby en nouveau master
  3. Keepalived déplace la VIP vers le nouveau master
```

### Composants logiciels

- **PostgreSQL 17** — moteur de base de données
- **repmgr / repmgrd** — gestion de la réplication et failover automatique
- **Keepalived** — VIP flottante via le protocole VRRP
- **Ansible** — orchestration (installé uniquement sur le pilote, mode agentless)
- **Ansible Vault** — chiffrement des secrets

---

## 3. Prérequis

### Sur le contrôleur (pilote)

- AlmaLinux 9 (ou toute distribution avec Python 3)
- Ansible Core ≥ 2.15 (`python3 -m pip install --user ansible`)
- Collection `community.postgresql` :
  ```bash
  ansible-galaxy collection install community.postgresql
  ```
- Accès SSH par clé vers toutes les cibles (utilisateur `ansible` avec sudo NOPASSWD)

### Sur les cibles

- AlmaLinux 9 ou Ubuntu 22.04
- Utilisateur `ansible` avec privilèges sudo sans mot de passe
- Deux interfaces réseau : NAT (Internet) + réseau interne `pgnet` (IP statique)
- Ressources minimales : 1,5 Go RAM, 10 Go disque disponible (seuils de lab ;
  recommandé en production : 2 Go / 20 Go)

### Sur le module Python des cibles (masters)

- `python3-psycopg2` (installé automatiquement, ou manuellement) :
  ```bash
  ansible primary -m package -a "name=python3-psycopg2 state=present" -b
  ```

---

## 4. Structure du projet

```
pg-ansible/
├── ansible.cfg                      # Configuration Ansible (inventaire, vault, roles_path)
├── inventories/
│   └── lab/
│       ├── hosts.yml                # Inventaire : groupes ha_alma, ha_ubuntu, standalone, primary, standby
│       └── group_vars/
│           └── all/
│               ├── vars.yml         # Variables (pont vers le Vault)
│               └── vault.yml        # Secrets chiffrés (AES-256)
├── roles/
│   ├── preflight/                   # OBJ-1 : vérifications préalables
│   ├── postgresql/                  # OBJ-2,3 : installation PostgreSQL (online/air-gap, multi-OS)
│   ├── replication/                 # OBJ-5 : réplication streaming + .pgpass + application_name
│   ├── repmgr/                      # OBJ-6 : repmgr + failover automatique (repmgrd)
│   ├── keepalived/                  # OBJ-7 : VIP flottante (VRRP)
│   └── validation/                  # OBJ-8 : validation post-déploiement + rapport
└── playbooks/
    ├── preflight.yml
    ├── deploy-standalone.yml
    ├── deploy-ha.yml
    ├── deploy-replication.yml
    ├── deploy-repmgr.yml
    ├── deploy-keepalived.yml
    └── validate-cluster.yml
```

### Secrets gérés par le Vault

| Variable Vault | Usage |
|----------------|-------|
| `vault_pg_postgres_password` | Mot de passe superutilisateur postgres |
| `vault_pg_repl_password` | Utilisateur de réplication `repl` |
| `vault_pg_repmgr_password` | Utilisateur `repmgr` |
| `vault_keepalived_auth_password` | Authentification VRRP |

Le fichier `vault.yml` est chiffré ; le mot de passe du Vault est stocké dans
`/root/.vault_pass.txt` (référencé par `ansible.cfg`, exclu de Git via `.gitignore`).

---

## 5. Installation pas à pas

Toutes les commandes se lancent **depuis le pilote**, dans `~/pg-ansible`.

### Étape 0 — Vérifier la connectivité

```bash
cd ~/pg-ansible
ansible all -m ping
```

Les 6 cibles doivent répondre `SUCCESS / pong`.

### Étape 1 — Vérifications préalables (preflight)

```bash
ansible-playbook playbooks/preflight.yml --limit ha_alma
```

Le playbook s'arrête avec un message explicite si un prérequis manque
(OS non supporté, RAM ou disque insuffisant). C'est le comportement attendu.

### Étape 2 — Déploiement Standalone

```bash
ansible-playbook playbooks/deploy-standalone.yml --limit standalone
```

Installe PostgreSQL sur les instances autonomes (AlmaLinux + Ubuntu).

Pour choisir une autre version :
```bash
ansible-playbook playbooks/deploy-standalone.yml --limit pg-alma-standalone -e pg_version=16
```

### Étape 3 — Déploiement Haute Disponibilité

Le déploiement HA s'effectue en quatre temps, dans cet ordre.

**3.1 — PostgreSQL en mode HA (sur master + standby)**
```bash
ansible-playbook playbooks/deploy-ha.yml --limit ha_alma
```

**3.2 — Réplication streaming**
```bash
ansible-playbook playbooks/deploy-replication.yml --limit ha_alma
```
Crée les utilisateurs `repl` et `repmgr`, clone le standby depuis le master
(`pg_basebackup`), et force l'`application_name` du standby (requis par repmgr).

**3.3 — repmgr + failover automatique**
```bash
ansible-playbook playbooks/deploy-repmgr.yml --limit ha_alma
```
Installe repmgr, enregistre les nœuds, et démarre le démon `repmgrd`
(surveillance + promotion automatique).

**3.4 — Keepalived (VIP)**
```bash
ansible-playbook playbooks/deploy-keepalived.yml --limit ha_alma
```
Installe Keepalived et monte la VIP `172.16.50.190` sur le master.

### Étape 4 — Validation

```bash
ansible-playbook playbooks/validate-cluster.yml --limit ha_alma
```

Affiche un bilan par nœud (service, connexion, rôle, réplication, VIP) et écrit
un rapport dans `/var/log/pg-validation-report.txt`.

### Déploiement sur le cluster Ubuntu

Les mêmes commandes s'appliquent en remplaçant `--limit ha_alma` par
`--limit ha_ubuntu`. Les rôles étant multi-OS, aucune adaptation n'est nécessaire.

### Installation air-gapped (sans Internet)

**Préparation du dépôt local** (sur un nœud disposant d'Internet, ex. .180) :
```bash
# Sur la machine source
sudo dnf install -y createrepo_c yum-utils httpd
sudo dnf download --resolve --alldeps --destdir=/var/www/html/pgrepo \
     postgresql17-server postgresql17-contrib repmgr_17
sudo createrepo_c /var/www/html/pgrepo
sudo systemctl enable --now httpd
sudo firewall-cmd --add-service=http --permanent && sudo firewall-cmd --reload
```

**Déploiement en mode air-gap** (la cible n'accède qu'au dépôt local) :
```bash
ansible-playbook playbooks/deploy-standalone.yml \
    --limit pg-alma-standalone -e pg_install_mode=airgap
```

Le paramètre `pg_install_mode=airgap` configure un dépôt local
(`http://172.16.50.180/pgrepo`) et installe avec `disablerepo="*"`,
garantissant qu'aucun paquet ne provient d'Internet.

---

## 6. Guide de tests

### Test 1 — Idempotence

Relancer un playbook ne doit rien casser ni modifier.

```bash
ansible-playbook playbooks/deploy-standalone.yml --limit standalone
```

**Attendu :** `changed=0` au second passage (aucune modification).

### Test 2 — Version configurable

```bash
ansible-playbook playbooks/deploy-standalone.yml --limit pg-alma-standalone -e pg_version=16
ssh ansible@172.16.50.182 "sudo -u postgres psql -tAc 'SHOW server_version;'"
```

**Attendu :** version `16.x` affichée.

### Test 3 — Réplication en temps réel

Écrire sur le master, vérifier la propagation sur le standby.

```bash
# Écriture sur le master
ssh ansible@172.16.50.180 "sudo -u postgres psql -c 'CREATE TABLE t_test(id int); INSERT INTO t_test VALUES(42);'"
# Lecture sur le standby
ssh ansible@172.16.50.181 "sudo -u postgres psql -tAc 'SELECT * FROM t_test;'"
```

**Attendu :** le standby renvoie `42`.

Vérifier l'état de la réplication côté master :
```bash
ssh ansible@172.16.50.180 "sudo -u postgres psql -x -c 'SELECT application_name, state FROM pg_stat_replication;'"
```
**Attendu :** `application_name = pg-alma-slave`, `state = streaming`.

### Test 4 — État du cluster repmgr

```bash
ssh ansible@172.16.50.180 "sudo -u postgres /usr/pgsql-17/bin/repmgr -f /etc/repmgr/17/repmgr.conf cluster show"
```

**Attendu :**
```
 1 | pg-alma-master | primary | * running |
 2 | pg-alma-slave  | standby |   running | pg-alma-master
```
(sans avertissement, sans `!` devant l'upstream)

### Test 5 — Failover automatique (test clé)

**Important :** le failover se déclenche sur une panne réelle (nœud injoignable),
pas sur un simple arrêt de service. Il faut **éteindre la VM master**.

1. Vérifier que repmgrd tourne sur les deux nœuds :
   ```bash
   for ip in 180 181; do echo -n ".$ip: "; ssh ansible@172.16.50.$ip "sudo systemctl is-active repmgr-17"; done
   ```
   Les deux doivent être `active`.

2. Éteindre la VM `pg-alma-master` dans VirtualBox (Machine → Fermer → Extinction).

3. Confirmer la panne :
   ```bash
   ping -c3 172.16.50.180
   ```
   **Attendu :** `100% packet loss`.

4. Observer le failover (jusqu'à ~45 s) :
   ```bash
   ssh ansible@172.16.50.181 "sudo journalctl -u repmgr-17 --no-pager | tail -20"
   ```
   **Attendu dans les logs :**
   ```
   this node is the only available candidate and will now promote itself
   promote_command is: "/usr/pgsql-17/bin/repmgr standby promote ..."
   STANDBY PROMOTE successful
   ```

5. Vérifier la promotion :
   ```bash
   ssh ansible@172.16.50.181 "sudo -u postgres psql -tAc 'SELECT pg_is_in_recovery();'"
   ```
   **Attendu :** `f` (le standby est devenu master).

### Test 6 — Adresse virtuelle (VIP)

```bash
# La VIP répond au ping
ping -c2 172.16.50.190
# La VIP est montée sur le master
ssh ansible@172.16.50.180 "ip a show enp0s8 | grep 172.16.50.190"
```

**Attendu :** la VIP répond et apparaît sur le master
(`inet 172.16.50.190/24 scope global secondary enp0s8`).

### Test 7 — Installation air-gapped

```bash
# Isoler la cible (couper la route par défaut)
ssh ansible@172.16.50.182 "sudo ip route del default"
# Vérifier l'absence d'Internet
ssh ansible@172.16.50.182 "ping -c2 8.8.8.8 || echo 'ISOLÉ'"
# Réinstaller depuis le dépôt local
ansible-playbook playbooks/deploy-standalone.yml --limit pg-alma-standalone -e pg_install_mode=airgap
# Vérifier
ssh ansible@172.16.50.182 "sudo -u postgres psql -tAc 'SELECT version();'"
```

**Attendu :** PostgreSQL installé et fonctionnel alors que la machine n'a pas
d'accès Internet.

### Test 8 — Validation globale

```bash
ansible-playbook playbooks/validate-cluster.yml --limit ha_alma
ssh ansible@172.16.50.180 "sudo cat /var/log/pg-validation-report.txt"
```

**Attendu :** rapport complet indiquant service actif, connexion OK,
rôles corrects, réplication active, VIP accessible.

---

## 7. Dépannage

| Symptôme | Cause probable | Solution |
|----------|----------------|----------|
| `Invalid data directory for cluster` (Ubuntu) | `postgresql.conf` écrasé sans `data_directory` | Réinitialiser le cluster : `sudo pg_dropcluster 17 main ; sudo pg_createcluster 17 main` |
| repmgr : `node not attached` / `!` dans cluster show | `application_name` du standby incorrect | Vérifier que `primary_conninfo` contient `application_name=<node_name>` |
| repmgrd : `repmgr: command not found` lors du failover | Chemin non absolu dans `promote_command` | Utiliser `{{ pg_bindir }}/repmgr` dans `repmgr.conf.j2`, puis redémarrer repmgrd |
| repmgrd ne bascule pas | Le master n'est pas réellement en panne (arrêt de service ≠ panne) | Éteindre la VM entière pour simuler une vraie panne |
| Deux masters (split-brain) | Ancien master redémarré comme primary après failover | Recloner l'ancien master en standby via `deploy-replication.yml` |
| `no password supplied` (repmgr) | `.pgpass` incomplet | Vérifier `.pgpass` (accès `repmgr` et `replication` vers les deux IP) |
| Connexion refusée port 5432 entre nœuds | Firewall | Ouvrir le port : `firewall-cmd --add-port=5432/tcp --permanent` (Alma) ou `ufw allow 5432` (Ubuntu) |

---

## 8. Limites connues

- **Bascule automatique de la VIP** : le failover PostgreSQL (promotion du
  standby) est automatique et fonctionnel. En revanche, la migration automatique
  de la VIP vers le nouveau master via Keepalived reste sensible sur réseau
  interne VirtualBox (comportement VRRP de l'environnement de virtualisation).
  La VIP se monte correctement sur le master ; sa bascule automatique lors d'un
  failover est à fiabiliser en environnement réseau standard.

- **Réintégration de l'ancien master** (`node rejoin`) : après un failover,
  l'ancien master doit être réintégré comme standby. La méthode fiable utilisée
  est le reclonage via `deploy-replication.yml`.

- **Seuils preflight** : abaissés pour l'environnement de lab
  (1,5 Go RAM / 10 Go disque) ; les valeurs de production recommandées sont
  2 Go / 20 Go, configurables via `preflight_min_ram_mb` et
  `preflight_min_disk_gb`.

---

## Auteur

Projet développé par **Ayet** dans le cadre d'un stage chez PROXYM.
Framework Ansible pour le déploiement automatisé de PostgreSQL en haute
disponibilité, multi-distributions et air-gapped.
