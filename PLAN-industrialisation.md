# Plan - Industrialiser les correctifs HA dans les roles

Objectif : mettre dans le code Ansible ce qui a ete repare a la main,
pour que le deploiement soit reproductible (test : reconstruire le cluster
de zero sans intervention manuelle).

## CORRECTIF 1 - application_name
Probleme : le standby se replique sous "walreceiver" au lieu de son node_name
-> repmgr ne le reconnait pas -> failover bloque.
Solution : dans role replication (clone-standby.yml), apres pg_basebackup :
  ALTER SYSTEM SET primary_conninfo = '... application_name=<node_name>';
puis restart.

## CORRECTIF 2 - .pgpass complet
Probleme : repmgr echoue sur "no password supplied".
Solution : role replication, tache pgpass.yml + template pgpass.j2 avec
acces replication ET repmgr vers les 2 IPs, mode 0600, owner postgres,
sur TOUS les noeuds HA.
Lignes :
  <master_ip>:5432:repmgr:repmgr:<pass>
  <slave_ip>:5432:repmgr:repmgr:<pass>
  <master_ip>:5432:replication:repl:<pass>
  <slave_ip>:5432:replication:repl:<pass>
  *:5432:replication:repl:<pass>
  *:5432:repmgr:repmgr:<pass>

## CORRECTIF 3 - parametres repmgr.conf
Probleme : repmgrd detecte la panne mais ne promeut pas.
Solution : dans template repmgr.conf.j2, ajouter :
  connection_check_type = 'connection'
  primary_visibility_consensus = false   (OBLIGATOIRE en cluster 2 noeuds)

## CORRECTIF 4 - shared_preload_libraries
Probleme : mettre 'repmgr' dans postgresql.conf AVANT d'installer le paquet
empeche PostgreSQL de demarrer.
Solution : c'est le role repmgr (pas postgresql) qui ajoute cette ligne,
APRES l'install du paquet, puis restart. Ordre : paquet -> shared_preload
-> restart -> register.

## Ordre de travail
1. Template repmgr.conf.j2 (correctif 3)
2. Tache pgpass.yml + pgpass.j2 (correctif 2)
3. application_name dans clone-standby.yml (correctif 1)
4. Verifier ordre role repmgr (correctif 4)
5. TEST : detruire cluster Alma, redeployer via playbooks SANS correction
   manuelle -> si failover marche, industrialisation reussie.

## Etat valide a ce jour
- 7 VMs, reseau pgnet, SSH, Ansible agentless, Vault AES-256
- PostgreSQL 17 multi-OS idempotent, version configurable (pg_version)
- Replication streaming 2 clusters
- repmgr cluster propre + failover MANUEL demontre (standby promote OK)
- GitLab : pg-ansible-lab

## Reste apres industrialisation
- OBJ-7 Keepalived (VIP) + complete OBJ-4
- OBJ-1 role preflight (dont detection versions PG existantes)
- OBJ-8 role validation + rapport
- OBJ-11 air-gapped
- Refaire cycle HA complet sur Ubuntu (.185/.186)
