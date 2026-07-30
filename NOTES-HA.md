# Corrections HA appliquées manuellement (à industrialiser dans les roles)

## Réplication
- Le primary_conninfo du standby DOIT contenir application_name=<node_name>
  sinon repmgr ne reconnaît pas le standby (apparaît comme "walreceiver").
  Corrigé via: ALTER SYSTEM SET primary_conninfo = '... application_name=pg-alma-slave'
- .pgpass sur chaque noeud avec accès replication ET repmgr vers les 2 IPs.

## repmgr.conf (à ajouter au template)
- connection_check_type = 'connection'
- primary_visibility_consensus = false   (obligatoire en cluster 2 noeuds)

## Failover
- Promotion manuelle validée: repmgr standby promote -> SUCCESS
- node rejoin échoue sur "no password supplied" : à corriger (pgpass/replication auth)
  Alternative: recloner l'ancien master via deploy-replication.yml
