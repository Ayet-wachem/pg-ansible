# Notes HA + version-configurabilite (a industrialiser)

## Failover repmgr
- application_name=<node_name> OBLIGATOIRE dans primary_conninfo du standby
- .pgpass complet (replication + repmgr) sur chaque noeud
- repmgr.conf: connection_check_type='connection', primary_visibility_consensus=false
- Promotion manuelle validee: repmgr standby promote -> SUCCESS
- node rejoin echoue sur "no password supplied" (a corriger)

## Version configurable (pg_version)
- -e pg_version=16 installe bien PG16 (paquets, chemins, service OK)
- LIMITE: coexistence 2 versions sur meme VM = conflit de port 5432
- A gerer par preflight (detection versions existantes) ou ports differents
