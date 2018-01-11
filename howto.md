# Cassandra


### Ressources

* https://www.digitalocean.com/community/tutorials/how-to-run-a-multi-node-cluster-database-with-cassandra-on-ubuntu-14-04

### pré-requis

* Trois machines en Debian Stretch

### installation des paquets debian

```bash
cat >/etc/apt/sources.list.d/cassandra.list<<EOF
deb http://www.apache.org/dist/cassandra/debian 30x main
deb http://www.apache.org/dist/cassandra/debian 39x main
EOF

apt update && apt install -y cassandra cassandra-tools
```

## Configuration

### la résolution des noms

Pensez à déclarer les trois noeuds dans /etc/hosts pour que les membres du cluster puisse aisement se trouver

### changer le nom du cluster

```bash
cluster_name: 'Kairos cluster'
```

### déclarer les seeds provider

avec la commande nodetool **status**

```bash
nodetool status
Datacenter: datacenter1
=======================
Status=Up/Down
|/ State=Normal/Leaving/Joining/Moving
--  Address       Load       Tokens       Owns (effective)  Host ID                               Rack
UN  10.0.145.201  314.14 KiB  256          34.2%             41e091d4-5df3-421c-b7b2-c85919588109  rack1
UN  10.0.145.202  196.86 KiB  256          32.6%             0674c1dd-4dfe-4539-a7c4-88fea05cfe1e  rack1
UN  10.0.145.203  326.17 KiB  256          33.3%             2528cbd6-e5c9-401c-8832-aa1d10b1fcbe  rack1
```
Dans la section **seed_provider** puis **class_name** et enfin parameters, déclarer les seeds (les noeuds)

exemple
```bash
- seeds: "10.0.145.202,10.0.145.201,10.0.145.203"
```

#### définir l'adresse d'écoute

L'interface "externe"

```bash
listen_address: 10.0.145.202
```

### Activer les rpc

Pour permettre à Kairos de s'adresser à tous les noeuds

```bash
start_rpc: true
rpc_address: 10.0.145.202
rpc_port: 9160
```
### obtenir le status du cluster

