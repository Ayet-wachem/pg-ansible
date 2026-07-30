# Corrections HA appliquees manuellement (a industrialiser dans les roles)

## Replication
- Le primary_conninfo du standby DOIT contenir application_name=<node_name>
  sinon repmgr ne reconnait pas le standby (apparait comme "walreceiver").
- .pgpass sur chaque noeud avec acces replication ET repmgr vers les 2 IPs.

## repmgr.conf (a ajouter au template)
- connection_check_type = 'connection'
- primary_visibility_consensus = false   (obligatoire en cluster 2 noeuds)

## Failover
- Promotion manuelle validee: repmgr standby promote -> SUCCESS
- node rejoin echoue sur "no password supplied" : a corriger
  Alternative: recloner l'ancien master via deploy-replication.yml
