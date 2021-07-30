# Kafka on GKE with open source Strimzi

Based on Strimzi quickstarts [here](https://strimzi.io/quickstarts/)

* Switch to cluster of your choice

```
kubectx cluster-1
```

## Install Strimzi Operator

* Create a namespace named `kafka`

```
kubectl create namespace kafka
```

* Install Strimzi ClusterRoles, ClusterRoleBindings and CustomResourceDefinitions

```
kubectl create -f 'https://strimzi.io/install/latest?namespace=kafka' -n kafka
```

Output similar to what is shown below (lists the objects created on the cluster)

```
customresourcedefinition.apiextensions.k8s.io/kafkas.kafka.strimzi.io created
rolebinding.rbac.authorization.k8s.io/strimzi-cluster-operator-entity-operator-delegation created
clusterrolebinding.rbac.authorization.k8s.io/strimzi-cluster-operator created
customresourcedefinition.apiextensions.k8s.io/kafkausers.kafka.strimzi.io created
customresourcedefinition.apiextensions.k8s.io/kafkarebalances.kafka.strimzi.io created
deployment.apps/strimzi-cluster-operator created
customresourcedefinition.apiextensions.k8s.io/kafkamirrormaker2s.kafka.strimzi.io created
clusterrole.rbac.authorization.k8s.io/strimzi-entity-operator created
clusterrole.rbac.authorization.k8s.io/strimzi-cluster-operator-global created
clusterrolebinding.rbac.authorization.k8s.io/strimzi-cluster-operator-kafka-broker-delegation created
rolebinding.rbac.authorization.k8s.io/strimzi-cluster-operator created
clusterrole.rbac.authorization.k8s.io/strimzi-cluster-operator-namespaced created
clusterrolebinding.rbac.authorization.k8s.io/strimzi-cluster-operator-kafka-client-delegation created
clusterrole.rbac.authorization.k8s.io/strimzi-kafka-client created
serviceaccount/strimzi-cluster-operator created
clusterrole.rbac.authorization.k8s.io/strimzi-kafka-broker created
customresourcedefinition.apiextensions.k8s.io/kafkatopics.kafka.strimzi.io created
customresourcedefinition.apiextensions.k8s.io/kafkabridges.kafka.strimzi.io created
customresourcedefinition.apiextensions.k8s.io/kafkaconnectors.kafka.strimzi.io created
customresourcedefinition.apiextensions.k8s.io/kafkaconnects2is.kafka.strimzi.io created
customresourcedefinition.apiextensions.k8s.io/kafkaconnects.kafka.strimzi.io created
customresourcedefinition.apiextensions.k8s.io/kafkamirrormakers.kafka.strimzi.io created
configmap/strimzi-cluster-operator created
```



* Watch the pods in the `kafka` namespace

`strimzi-cluster-operator` pod is running on the cluster that watches out for the creation of CRDs to stand up a kafka cluster. 

```
$ kubectl get po -n kafka
NAME                                      READY   STATUS    RESTARTS   AGE
strimzi-cluster-operator-c8ddd59d-znlw7   1/1     Running   0          3m15s
```

To watch this operator logs `kubectl logs deployment/strimzi-cluster-operator -n kafka -f` to know what goes on when trying to install kafka.


## Provision a Kafka Cluster

* Custom resource to create a kafka cluster

```
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: my-cluster
spec:
  kafka:
    version: 2.8.0
    replicas: 1
    listeners:
      - name: plain
        port: 9092
        type: internal
        tls: false
      - name: tls
        port: 9093
        type: internal
        tls: true
    config:
      offsets.topic.replication.factor: 1
      transaction.state.log.replication.factor: 1
      transaction.state.log.min.isr: 1
      log.message.format.version: "2.8"
      inter.broker.protocol.version: "2.8"
    storage:
      type: jbod
      volumes:
      - id: 0
        type: persistent-claim
        size: 100Gi
        deleteClaim: false
  zookeeper:
    replicas: 1
    storage:
      type: persistent-claim
      size: 100Gi
      deleteClaim: false
  entityOperator:
    topicOperator: {}
    userOperator: {}
```
This deploys single node cluster. For three node cluster use [this file](https://strimzi.io/examples/latest/kafka/kafka-persistent.yaml)

* Create a CR that provisions a kafka cluster `my-cluster` in the namespace `kafka`

```
kubectl apply -f https://strimzi.io/examples/latest/kafka/kafka-persistent-single.yaml -n kafka
```


```
kubectl wait kafka/my-cluster --for=condition=Ready --timeout=300s -n kafka 
```

* Note pods running now for `my-cluster`

```
$ k get po -n kafka
NAME                                          READY   STATUS    RESTARTS   AGE
my-cluster-entity-operator-7594496dff-wfq65   3/3     Running   0          21m
my-cluster-kafka-0                            1/1     Running   0          21m
my-cluster-zookeeper-0                        1/1     Running   0          22m
strimzi-cluster-operator-c8ddd59d-znlw7       1/1     Running   0          70m
```

* Note the PVCs created and bound

```
$ kubectl get pvc -n kafka
NAME                          STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
data-0-my-cluster-kafka-0     Bound    pvc-fa00087b-8c8d-44fb-8628-3c6f51bec2db   100Gi      RWO            standard       20m
data-my-cluster-zookeeper-0   Bound    pvc-76fceb48-523b-432e-8ae2-de0f575c43d2   100Gi      RWO            standard       21m
```

## Test 

* Post messages from terminal 1

```
kubectl -n kafka run kafka-producer -ti --image=quay.io/strimzi/kafka:0.24.0-kafka-2.8.0 --rm=true --restart=Never -- bin/kafka-console-producer.sh --broker-list my-cluster-kafka-bootstrap:9092 --topic my-topic
```

Type messages at prompt `>`

* Receive messages from terminal 2

```
kubectl -n kafka run kafka-consumer -ti --image=quay.io/strimzi/kafka:0.24.0-kafka-2.8.0 --rm=true --restart=Never -- bin/kafka-console-consumer.sh --bootstrap-server my-cluster-kafka-bootstrap:9092 --topic my-topic --from-beginning
```

`CTRL+c` to terminate on both terminals.

## Clean up

* Delete kafka cluster
```
kubectl delete kafka/my-cluster -n kafka
```

* Delete operator

```
kubectl delete -f 'https://strimzi.io/install/latest?namespace=kafka' -n kafka
```

* Delete namespace

```
kubectl delete namespace kafka
```
