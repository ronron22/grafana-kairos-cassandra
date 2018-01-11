# Grafana et kairosdb avec cluster cassandra en backend

L'idée est d'utiliser Cassandra pour y stocker des quantités énorme de données, car Prometheus et Influxdb ne semblent pas proposer les fonctions de clustering dans leur version community.

4 sections :

1. cassandra - le stockage 
2. kairosdb - le traitement
3. grafana - la présentation
4. collectd - l'alimentation

## Cassandra

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

### Configuration

#### la résolution des noms

Pensez à déclarer les trois noeuds dans /etc/hosts pour que les membres du cluster puisse aisement se trouver

#### changer le nom du cluster

```bash
cluster_name: 'Kairos cluster'
```

#### déclarer les seeds provider

Dans la section **seed_provider**  puis **class_name** et enfin **parameters**, déclarer les **seeds** (les noeuds)

exemple

```bash
- seeds: "10.0.145.202,10.0.145.201,10.0.145.203"
```

#### définir l'adresse d'écoute

```bash
listen_address: 10.0.145.202
```

#### Activer les rpc

Pour permettre à Kairos de s'adresser à tous les noeuds

```bash
start_rpc: true
rpc_address: 10.0.145.202
rpc_port: 9160
```

relancer

```bash
service cassandra restart
```

### vérification et troubleshooting

Consulter les logs ici **/var/log/cassandra/**

#### obtenir le status du cluster

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

## Kairosdb

### pré-requis

* une machines en Debian Stretch

(cette machine hébergera Kairosdb et Grafana)

### installation 

apt install kairosdb

### configuration

La configuration se trouve ici : /opt/kairosdb/conf/

Voici les lignes à activer/configurer (les lignes les plus importantes sont les deux premières)

```bash
kairosdb.service.datastore=org.kairosdb.datastore.cassandra.CassandraModule
kairosdb.datastore.cassandra.cql_host_list=10.0.145.201:9160,10.0.145.202:9160,10.0.145.203:9160
kairosdb.datastore.cassandra.keyspace=kairosdb
kairosdb.datastore.cassandra.simultaneous_cql_queries=20
kairosdb.datastore.cassandra.query_reader_threads=6
kairosdb.datastore.cassandra.row_key_cache_size=50000
kairosdb.datastore.cassandra.string_cache_size=50000
kairosdb.datastore.cassandra.read_consistency_level=ONE
kairosdb.datastore.cassandra.write_consistency_level=QUORUM
kairosdb.datastore.cassandra.connections_per_host.local.core=5
kairosdb.datastore.cassandra.connections_per_host.local.max=100
kairosdb.datastore.cassandra.connections_per_host.remote.core=1
kairosdb.datastore.cassandra.connections_per_host.remote.max=10
kairosdb.datastore.cassandra.max_requests_per_connection.local=128
kairosdb.datastore.cassandra.max_requests_per_connection.remote=128
kairosdb.datastore.cassandra.max_queue_size=500
kairosdb.datastore.cassandra.use_ssl=false
```

Pensez à commenter le **datastore h2**

relancer

```bash
service kairosdb restart
```

### vérification

#### sur cassandra, vérifiez que le keyspace kairosdb est créé

retenez le nom du **kairosdb.datastore.cassandra.keyspace** 

```bash
root@cassandra2:~ # cqlsh 10.0.145.202 9042                                                                            
Connected to Kairos cluster at 10.0.145.202:9042.
[cqlsh 5.0.1 | Cassandra 3.9 | CQL spec 3.4.2 | Native protocol v4]
Use HELP for help.
cqlsh> SELECT * FROM system_schema.keyspaces;

 keyspace_name      | durable_writes | replication
--------------------+----------------+-------------------------------------------------------------------------------------
           kairosdb |           True | {'class': 'org.apache.cassandra.locator.SimpleStrategy', 'replication_factor': '1'}
...
```

### consultation des métriques sur Kairosdb

Connectez-vous sur http://adresse-ip:8080/ puis selectionner les métriques dans la section **metric**

Les métriques sont présentées au format **Graphite**

### vérification et troubleshooting

Consulter les logs ici **/opt/kairosdb/log/**

## Grafana

Installons **grafana** sur le même serveur que **kairosdb**

### installation

#### mettons directement la version de l'éditeur

```bash
cat /etc/apt/sources.list.d/grafana.list 
deb https://packagecloud.io/grafana/stable/debian/ stretch main
```

#### installation du plugin kairosdb

```bash
grafana-cli plugins install grafana-kairosdb-datasource 
```

relancer

```bash
service grafana restart
```

### Configuration 

#### ajout de la Datasources

Dans **HTTP settings** bien choisir les paramètres suivants

* Type : KairosDB
* Url : http://localhost:8080/
* Access : **Proxy** (pas direct !!)

#### Dashboard

Utilisez la notation Graphite (.) ex "collectd.cpu.0.cpu.idle.value_rate"

Les métriques peuvent être récupérées du dashboard Kairos

### vérification et troubleshooting

Consulter les logs ici **/var/log/grafana/**

## Collectd

L'alimentation des datas

### installation

```bash
apt install collectd
```

#### plugin d'alimentation de kairosdb

récupérer ce plugin (pas un autre)

```bash
git clone https://github.com/kairosdb/collectd-kairosdb.git
```

Installation du plugin

```bash
cp -v kairosdb_writer.py /usr/lib/collectd/
```

Copie de la configuration

```bash
cp -v  kairosdb.conf /etc/collectd/collectd.conf.d/
```

Adaptez la configuration (adresse IP de KairosDBURI et ModulePath "/usr/lib/collectd/") 

```bash
mv -v  kairosdb.conf /etc/collectd/collectd.conf.d/
```

relancer

```bash
service grafana restart
```

### vérification et troubleshooting

Consulter les logs ici **/var/log/syslog**

## Si cela ne marche pas

Contactez-moi ou cherchez
