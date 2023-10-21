# MongoDB High availability cluster

Tested on:
- Platform: Windows10
- Minikube: 1.30.1
- Kubectl: 1.27
- Addons: dashboard, default-storageclass, ingress, metrics-server, storage-provisioner

Before all, Create Namespaces.

```bash
kubectl create namespace mongo
```

## Replica Set

- 1 master
- 2 secondary

```bash
cd ./database/mongo/rs/
kubectl apply -f rbac.yml
kubectl apply -f configmap.yml
kubectl apply -f certs.yml # tls
kubectl apply -f statefulset.yml
```

Waiting all nodes become healthy, then run initial script

```bash
# mongodb-rs-0 bash
mongosh /data/cert/nodes.mongodb
```

### Expose Connection to public

- S1
  - set URI with `directConnection=true`
  - Use HAproxy to replace k8s LoadBalancer
- S2
  - shard replica set, it has `mongos` as router

I use S1 and setup HAproxy deployment, run command to deploy it. You must set more or equal to 2 replicas. So when master goes down and switch master node, i can ensure HAproxy is high available.

```bash
kubectl apply -f haproxy.yml
```

Then i can connect to master and secondary directly

```sh
# master
mongodb://tom-rs:jerry@localhost:27000/?authMechanism=DEFAULT&tlsInsecure=true&directConnection=true
# secondary
mongodb://tom-rs:jerry@localhost:27017/?authMechanism=DEFAULT&tlsInsecure=true&directConnection=true
```

HAproxy enabled stats monitor, but you should be careful security problem in production

```
# stats
# usr: admin  pwd: admin@rserve
http://127.0.0.1:8000/haproxy_stats
```

## Sharded Replica Set

Pods:

- sh0-0, sh0-1, sh0-2
- sh1-0, sh1-1, sh1-2
- configdb-1, configdb-2
- mongos-1


1. Setup basic configuration

```bash
kubectl apply -f rbac.yml
kubectl apply -f certs.yml # tls
```

2. Deploy two sharded replica set and create superuser

```bash
kubectl apply -f sh.yml
# mongodb-sh0-0 and mongodb-sh1-0 bash
mongosh /data/cert/nodes.mongodb
```

3. Deploy two configdb pods and create user `tom-mongos`

```bash
kubectl apply -f configdb.yml
# configdb-0 bash
mongosh /data/cert/nodes.mongodb
```

4. Create one mongos and login as `tom-mongos` to configure cluster settings

```bash
kubectl apply -f mongos.yml
# mongos-0 bash
mongosh /data/cert/nodes.mongodb
```

5. Deploy HAproxy for load balancing and proxy

```bash
kubectl apply -f haproxy.yml
```

6. Access mongos with url: `mongodb://tom-mongos:jerry@localhost:27000/?directConnection=true&authMechanism=DEFAULT` and run `sh.status()`
   1. you can try to run `db.getUsers()` to check if they are using different user data

7. Run command at mongos shell to insert test data

```javascript
sh.enableSharding('test')
sh.shardCollection("test.sales", { "_id": "hashed" })

// inserting data for testing
var largeArray = []
for (let index = 0; index < 777; index++) {
    largeArray.push(
        { 'item': Math.random().toString().substring(2, 9), 'price': Math.random() * 100000, 'quantity': Math.random() * 100, 'date': new Date() }
    )
}
db.sales.insertMany(largeArray)

db.adminCommand( { flushRouterConfig: "test.sales" } );
db.getSiblingDB("test").sales.getShardDistribution();
```

# Notes

## MongoDB Kubernetes

Currently, i do not recommend to use [MongoDB Community Kubernetes Operator](https://github.com/mongodb/mongodb-kubernetes-operator) since it need to use helm to configure settings e.g. TLS. Follow [How to Run MongoDB on Kubernetes](https://phoenixnap.com/kb/kubernetes-mongodb) to build cluster should be a better choice. Just forget bitnami, it is worse than MongoDB operator. - 20230610

## HAproxy and Mongos TLS

You dont have to change haproxy configuration, what you need is switch mongodb compose TLS mode to `ON` and specify *CA* and *mongos certificate*.

`directConnection=true` is necessary since mongo server return kubernetes dns that can not be resolve by public DNS if you set false

```
mongodb://tom-mongos:jerry@localhost:27000/?authMechanism=DEFAULT&directConnection=true&tlsCAFile=F%3A%5CKChan+Notes%5CTech%5CRepositories%5CTech-examples%5CKubernetes%5Crepository%5Cunknown-clusters%5Cdatabase%5Cmongo%5Ckey%5CmongoCA.crt&tls=true&tlsInsecure=true&tlsCertificateKeyFile=F%3A%5CKChan+Notes%5CTech%5CRepositories%5CTech-examples%5CKubernetes%5Crepository%5Cunknown-clusters%5Cdatabase%5Cmongo%5Ckey%5Cmongos-0.mongo.pem
```