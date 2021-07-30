# Install Multicluster CockroachDB

Based on documentation [here](https://www.cockroachlabs.com/docs/stable/orchestrate-cockroachdb-with-kubernetes-multi-cluster.html)

## Create three K8S clusters

* Standup a cluster named cockroachdb1 in `us-central1-a`

```
gcloud container clusters create cockroachdb1 --zone=us-central1-a
```
* Rename kubecontext
```
k ctx cockroachdb1=$(k ctx -c)
```

* Standup a cluster named cockroachdb2 in `us-east1-b`
```
gcloud container clusters create cockroachdb2 --zone=us-east1-b
```
* Rename kubecontext
```
k ctx cockroachdb2=$(k ctx -c)
```

* Standup a cluster named cockroachdb3 in `us-west1-a`
```
gcloud container clusters create cockroachdb3 --zone=us-west1-a
```

* Rename kubecontext
```
k ctx cockroachdb3=$(k ctx -c)
```

## Access to the clusters'

* Provide `cluster-admin` kubernetes role-binding to your user-id

```
export USER= <<your email>>

kubectl create clusterrolebinding $USER-cluster-admin-binding --clusterrole=cluster-admin --user=$USER --context cockroachdb1

kubectl create clusterrolebinding $USER-cluster-admin-binding --clusterrole=cluster-admin --user=$USER --context cockroachdb2

kubectl create clusterrolebinding $USER-cluster-admin-binding --clusterrole=cluster-admin --user=$USER --context cockroachdb3

```

## Start CockroachDB

* Download scripts and config files 

```
curl -OOOOOOOOO https://raw.githubusercontent.com/cockroachdb/cockroach/master/cloud/kubernetes/multiregion/{README.md,client-secure.yaml,cluster-init-secure.yaml,cockroachdb-statefulset-secure.yaml,dns-lb.yaml,example-app-secure.yaml,external-name-svc.yaml,setup.py,teardown.py}

```

* Files downloaded
```
$ ls
client-secure.yaml                   dns-lb.yaml              README.md
cluster-init-secure.yaml             example-app-secure.yaml  setup.py
cockroachdb-statefulset-secure.yaml  external-name-svc.yaml   teardown.py
```

* Edit `setup.py` file for 

```
contexts = {
    'us-central1-a': 'cockroachdb1',
    'us-east1-b': 'cockroachdb2',
    'us-west1-a': 'cockroachdb3',
}
```

```

```

## Install CockroachDB locally

* Download binary

```
curl https://binaries.cockroachdb.com/cockroach-v21.1.6.linux-amd64.tgz | tar -xz && sudo cp -i cockroach-v21.1.6.linux-amd64/cockroach /usr/local/bin/
```

* Copy GEOS libraries
```
sudo mkdir -p /usr/local/lib/cockroach

sudo cp -i cockroach-v21.1.6.linux-amd64/lib/libgeos.so /usr/local/lib/cockroach/

sudo cp -i cockroach-v21.1.6.linux-amd64/lib/libgeos_c.so /usr/local/lib/cockroach/

```

* Provide execute permissions

```
sudo chmod +x /usr/local/bin/cockroach
sudo chmod +x /usr/local/lib/cockroach
sudo chmod +rx /usr/local/lib/cockroach/*
```

* Test

```
which cockroach
cockroach demo
```

* in the cockroach terminal

```
> SELECT ST_IsValid(ST_MakePoint(1,2));
```

should see

```
  st_isvalid
--------------
     true
(1 row)

Time: 1ms total (execution 1ms / network 0ms)
```

## Install database on all three clusters

* Run database setup
```
python setup.py
```
It takes a few minutes to complete

* Verify

```
kubectl get pods --selector app=cockroachdb --all-namespaces --context cockroachdb1
```
* Output
```
NAMESPACE       NAME            READY   STATUS    RESTARTS   AGE
us-central1-a   cockroachdb-0   1/1     Running   0          5m9s
us-central1-a   cockroachdb-1   1/1     Running   0          5m8s
us-central1-a   cockroachdb-2   1/1     Running   0          5m8s
```

```
kubectl get pods --selector app=cockroachdb --all-namespaces --context cockroachdb2
```
* Output
```
NAMESPACE    NAME            READY   STATUS    RESTARTS   AGE
us-east1-b   cockroachdb-0   1/1     Running   0          5m44s
us-east1-b   cockroachdb-1   1/1     Running   0          5m44s
us-east1-b   cockroachdb-2   1/1     Running   0          5m43s
```

```
kubectl get pods --selector app=cockroachdb --all-namespaces --context cockroachdb3
```

* Output

```
NAMESPACE    NAME            READY   STATUS    RESTARTS   AGE
us-west1-a   cockroachdb-0   1/1     Running   0          6m24s
us-west1-a   cockroachdb-1   1/1     Running   0          6m24s
us-west1-a   cockroachdb-2   1/1     Running   0          6m24s
```

## Connect the cluster pods

* Open firewall port `26257` for inter-node traffic

```
gcloud compute firewall-rules create allow-cockroach-internal --allow=tcp:26257 --source-ranges=10.0.0.0/8,172.16.0.0/12,192.168.0.0/16
```

# Test

* Launch a client-pod 
```
kubectl create -f client-secure.yaml --context cockroachdb1
```

* Connect via the client
```
kubectl exec -it cockroachdb-client-secure --context cockroachdb1 --namespace default -- ./cockroach sql --certs-dir=/cockroach-certs --host=cockroachdb-public
```
* Test

```
#
# Welcome to the CockroachDB SQL shell.
# All statements must be terminated by a semicolon.
# To exit, type: \q.
#
# Server version: CockroachDB CCL v21.1.6 (x86_64-unknown-linux-gnu, built 2021/07/20 15:30:39, go1.15.11) (same version as client)
# Cluster ID: 0e945bb8-665c-49fd-afc7-b0baf5690fb5
#
# Enter \? for a brief introduction.
#
root@cockroachdb-public:26257/defaultdb> CREATE DATABASE bank;
CREATE DATABASE

Time: 476ms total (execution 475ms / network 1ms)

root@cockroachdb-public:26257/defaultdb> CREATE TABLE bank.accounts (id INT PRIMARY KEY, balance DECIMAL);
CREATE TABLE

Time: 715ms total (execution 715ms / network 1ms)

root@cockroachdb-public:26257/defaultdb> INSERT INTO bank.accounts VALUES (1, 1000.50);
INSERT 1

Time: 995ms total (execution 994ms / network 1ms)

root@cockroachdb-public:26257/defaultdb> SELECT * FROM bank.accounts;
  id | balance
-----+----------
   1 | 1000.50
(1 row)

Time: 36ms total (execution 36ms / network 1ms)

root@cockroachdb-public:26257/defaultdb> CREATE USER roach WITH PASSWORD 'Q7gc8rEdS'; 
CREATE ROLE

Time: 737ms total (execution 737ms / network 1ms)

root@cockroachdb-public:26257/defaultdb> GRANT admin TO roach;
GRANT

Time: 3.164s total (execution 3.164s / network 0.001s)

root@cockroachdb-public:26257/defaultdb> \q

```

## Access via Console

* Port-forward from your local machine to a cockroachdb-pod

```
kubectl port-forward cockroachdb-0 8080 --context cockroachdb2 --namespace us-east1-b
```

* Connect to http://localhost:8080 from the browser to access CockroachDB Console

    * Verify 9 Nodes 3 per region
    * `bank` database in the Databases 
    * Table `public`
    * Performance in the Network Latency page


## Simulate datacenter failure 

* Scale down statefulset in `cockroachdb3`

```
kubectl scale statefulset cockroachdb --replicas=0 --context cockroachdb3 --namespace us-west1-a
```

* Verify

```
kubectl get po --context cockroachdb3 --namespace us-west1-a
```

Output

```
No resources found in us-west1-a namespace.
```

* Notice 3 dead nodes on the overview page on the console


## Clean up

* Delete client pod

```
kubectl delete pod cockroachdb-client-secure --context cockroachdb1
```

* Delete all clusters

```
gcloud container clusters delete cockroachdb1 --zone=us-central1-a
gcloud container clusters delete cockroachdb2 --zone=us-east1-b
gcloud container clusters delete cockroachdb3 --zone=us-west1-a
```



