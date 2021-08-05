# Multicluster Mongodb ReplicaSet

## Stand up two GKE clusters


Start `cluster1` in `us-central1-a`

```
gcloud container clusters create cluster1 --zone=us-central1-a
```

Rename kubecontext

```
kubectx cluster1=$(kubectx -c)
```

Start `cluster2` in `us-central1-a`

```
gcloud container clusters create cluster2 --zone=us-east1-b
```

Rename kubecontext

```
kubectx cluster2=$(kubectx -c)
```

## Access to clusters

Provide `cluster-admin` kubernetes role-binding to your user-id

```
export USER= <<your email>>

kubectl create clusterrolebinding $USER-cluster-admin-binding --clusterrole=cluster-admin --user=$USER --context cluster1

kubectl create clusterrolebinding $USER-cluster-admin-binding --clusterrole=cluster-admin --user=$USER --context cluster2
```

## Deploy two instances of Mongo on cluster1

Switch to `cluster1`

```
kubectx cluster1
```

Create a new namespace for mongo

```
kubectl create ns mongo
```

Create PVC, Deployment and Service

```
kubectl apply -f pvc1.yaml -n mongo
kubectl apply -f mongo1-deployment.yaml -n mongo
kubectl apply -f mongo1-service.yaml -n mongo
```

In a few mins mongo1 pod should be running `kubectl get po -n mongo`. Verify the service is exposed on port `30017`

```
$ kubectl get svc -n mongo
NAME     TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)           AGE
mongo1   NodePort   10.3.252.208   <none>        27017:30017/TCP   44s
```

## Stand up 2nd mongo on the same cluster

Create PVC, Deployment and Service exposed on port 30018

```
kubectl apply -f pvc2.yaml -n mongo
kubectl apply -f mongo2-deployment.yaml -n mongo
kubectl apply -f mongo2-service.yaml -n mongo
```

Verify

```
$ k get svc -n mongo
NAME     TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)           AGE
mongo1   NodePort   10.3.252.208   <none>        27017:30017/TCP   6m22s
mongo2   NodePort   10.3.240.108   <none>        27017:30018/TCP   29s
```

## Stand 3rd mongo on cluster2

Switch to `cluster2` and create `mongo` namespace

```
kubectx cluster2
kubectl create ns mongo
```

Deploy mongo application

```
kubectl apply -f pvc3.yaml -n mongo
kubectl apply -f mongo3-deployment.yaml -n mongo
kubectl apply -f mongo3-service.yaml -n mongo
```

Service should be exposed on port `30017`

```
$ kubectl get svc -n mongo
NAME     TYPE       CLUSTER-IP   EXTERNAL-IP   PORT(S)           AGE
mongo3   NodePort   10.8.8.9     <none>        27017:30017/TCP   12s
```

## Connect instances as a cluster as a Mongo Replica Set

### Get node external IP addresses

Get internal and external IP address for one of the nodes from `cluster2`. Export external IP as an environment variable NODEIP_CLUSTER2

```
k get nodes -o wide --context cluster2
```


Get internal and external IPs for one of the nodes on `cluster1`. Export external IP as an environment variable NODEIP_CLUSTER1

```
k get nodes -o wide --context cluster1
```


### Create firewall rules to allow connections via NodePorts

You need to identify a client machine from where you will connect to the mongo cluster for testing. Find the IP address of that machine.

Create firewall-rules to allow node port connections from specific IP address of the client machine from where you will be accessing MongoDB.

```
gcloud compute firewall-rules create mongo-nodeports --allow tcp:30017,tcp:30018,tcp:30019 --source-ranges <<ClientMachineIP>>
```

Create a firewall rule that allows the mongo instances to talk to each other via node external IPs for all the nodes in both `cluster1` and `cluster2`.
```
gcloud compute firewall-rules create mongo-nodeports-pods --allow tcp:30017,tcp:30018,tcp:30019 --source-ranges <<podcidr>>,<<cluster1 internalIp cidr>>,<<cluster2 internalip cidr>>,<<comma separated list of node external ips>>
```

### Create a Replica Set

We have 3 pods running. Let us set a topology of primary and secondary among these three. The topology will change as we test by creating scenarios of unavailability. 
We have 
* 1 pod created by `mongo3` deployment running on `cluster2`. We will make this **primary**
* 2 pods created by `mongo1` and `mongo2` deployments running on `cluster1`. We will make these two **secondary** mongo instances


Remote into the container shell
```
kubectx cluster2
kubectl exec -it $(k get po -o jsonpath='{.items[0].metadata.name}' -n mongo) -n mongo -- mongo 
```

Create a replica set

```
rs.initiate()
```

Output similar to

```
{
	"info2" : "no configuration specified. Using a default configuration for the set",
	"me" : "mongo3-7bc459d8c9-rmkz2:27017",
	"ok" : 1,
	"$clusterTime" : {
		"clusterTime" : Timestamp(1627956506, 1),
		"signature" : {
			"hash" : BinData(0,"AAAAAAAAAAAAAAAAAAAAAAAAAAA="),
			"keyId" : NumberLong(0)
		}
	},
	"operationTime" : Timestamp(1627956506, 1)
}

```

Create a replica set (substitute node IP) and configuring the pod running on `cluster2` exposed on NodePort `30017` as `primary`

```
var cfg = rs.conf();
cfg.members[0].host="<<NODEINTERNALIP_CLUSTER2>>:30017";
rs.reconfig(cfg);
```
Output similar to

```
{
	"ok" : 1,
	"$clusterTime" : {
		"clusterTime" : Timestamp(1627996255, 1),
		"signature" : {
			"hash" : BinData(0,"AAAAAAAAAAAAAAAAAAAAAAAAAAA="),
			"keyId" : NumberLong(0)
		}
	},
	"operationTime" : Timestamp(1627996255, 1)
}
```

Add additional members from `cluster1` to the mongo cluster. (substitute node IP)
```
rs.add("<<NODEINTERNALIP_CLUSTER1>>:30017");
rs.add("<NODEINTERNALIP_CLUSTER1>:30018");
```

Verify the status of the mongo cluster by running

```
rs.status()
```

You should see three mongos in members to form a Replica Set where primary is the instance from `cluster2` and secondary are two instances from `cluster1`


## Test the Replica set

We will need a client from which you can connect to the mongo instances and test by inserting data.

### Install mongosh client

Per documentation [here]()

Example for Linux

```
wget https://downloads.mongodb.com/compass/mongodb-mongosh_1.0.4_amd64.deb
sudo dpkg -i mongodb-mongosh_1.0.4_amd64.deb 
```

Verify
```
which mongosh
```

output
```
/usr/bin/mongosh
```


### Connect to the primary mongo and insert data

```
mongosh --host $NODEIP_CLUSTER2 --port 30017
```
Note the prompt showing up as shown below indicating that you are connected to `primary` mongo instance in the replica set.

```
rs0 [direct: primary] test>
```

Let us create a new database with name `myNewDatabase`
```
use myNewDatabase
```

Output similar to

```
switched to db myNewDatabase
```

Add data to a collection named `myCollection`

```
 db.myCollection.insertOne( { x:1 } ); 
 db.myCollection.insertOne( { y:2 } ); 
```

Get data and verify

```
db.getCollection("myCollection").find()
```

Output similar to 

```
[
  { _id: ObjectId("610c421996b1133aa15b484c"), x: 1 },
  { _id: ObjectId("610c432696b1133aa15b484d"), y: 2 }
]

```

### Connect to secondary mongo instances and verify

```
mongosh --host $NODEIP_CLUSTER1 --port 30017
```

Notice the prompt showing up as secondary
```
rs0 [direct: secondary] test>
```

Connect to your database
```
use myNewDatabase
```

Output similar to
```
switched to db myNewDatabase
```

Set ReadPref to read from `secondary` so that you can read data from here.

```
db.getMongo().setReadPref("nearest")
```

Issue a find to get data from the collection

```
db.getCollection("myCollection").find()
```

Output similar to

```
[
  { _id: ObjectId("610c421996b1133aa15b484c"), x: 1 },
  { _id: ObjectId("610c432696b1133aa15b484d"), y: 2 }
]
```

The above proves that the data added to the primary is replicated to secondary instance.

Repeat all the above verification steps connecting to the other secondary instance exposed on Nodeport `30018`

```
mongosh --host $NODEIP_CLUSTER1 --port 30018
```

### Test bringing down the primary pod on cluster2

Let us set the replica count to zero for mongo deployment on `cluster2` that is currently acting as primary

```
kubectx cluster2
kubectl scale deploy mongo3 --replicas=0 -n mongo
```

Verify that the pod goes down

```
$ kubectl get po -n mongo
NAME                      READY   STATUS        RESTARTS   AGE
mongo3-7bc459d8c9-xhv6r   1/1     Terminating   0          29h
```

Switch over to `cluster1` and verify connecting to the mongo instances

```
kubectx cluster1
```

```
mongosh --host $NODEIP_CLUSTER1 --port 30017
```

```
mongosh --host $NODEIP_CLUSTER1 --port 30017
```

One of them will show up as `primary` indicating that the secondary has been elected as primary now.

```
rs0 [direct: primary] test>
```

The other one will continue to show up as secondary

```
rs0 [direct: secondary] test>
```

Also verify that the data is intact

```
use myNewDatabase
use myNewDatabase
```

**Note** For secondary you will have to `db.getMongo().setReadPref("nearest")`.

to notice the two records exist

```
[
  { _id: ObjectId("610c421996b1133aa15b484c"), x: 1 },
  { _id: ObjectId("610c432696b1133aa15b484d"), y: 2 }
]
```

### Test adding back the pod on cluster2

Before we bring up the pod, let us make some changes to the data.

Add an additional record from the primary

```
use myNewDatabase
db.myCollection.insertOne( { z:3 } ); 
```

Output

```
{
  acknowledged: true,
  insertedId: ObjectId("610c5d840e18ad42469c89ca")
}
```

Verify from the secondary that you can see the data

```
use myNewDatabase
db.getMongo().setReadPref("nearest")
db.getCollection("myCollection").find()
```

Output similar to 

```
[
  { _id: ObjectId("610c421996b1133aa15b484c"), x: 1 },
  { _id: ObjectId("610c432696b1133aa15b484d"), y: 2 },
  { _id: ObjectId("610c5d840e18ad42469c89ca"), z: 3 }
]

```

Now let us bring back the pod

```
kubectx cluster2
kubectl scale deployment mongo3 --replicas=1 -n mongo
kubectl get po -n mongo -w
```

Wait until the pod is `Running` and `^C`

Now let us connect to mongo instance on cluster2

```
mongosh --host $NODEIP_CLUSTER2 --port 30017
```

Notice the prompt showing up as `secondary`. This is because the primary is now on `cluster1`.

```
rs0 [direct: secondary] test>
```
Check the database collection again to make sure you have 3 records.


## Additional Testing

* Now bring down the primary instance on `cluster1` to see the impact


## Cleanup

Cleanup `cluster1`

```
kubectx cluster1
kubens mongo
kubectl delete -f mongo1-service.yaml
kubectl delete -f mongo1-deployment.yaml
kubectl delete -f pvc1.yaml

kubectl delete -f mongo2-service.yaml
kubectl delete -f mongo2-deployment.yaml
kubectl delete -f pvc2.yaml
```

Cleanup `cluster2`

```
kubectx cluster1
kubens mongo
kubectl delete -f mongo3-service.yaml
kubectl delete -f mongo3-deployment.yaml
kubectl delete -f pvc3.yaml

```

Delete Clusters
```
gcloud container clusters delete cluster1 --zone=us-central1-a
```

```
gcloud container clusters delete cluster2 --zone=us-east1-b
```







