
# Disaster recovery using Longhorn

Longhorn is a distributed block storage system on K8S. You can use longhorn as a persistent storage provider. The data gets replicated across multiple nodes on the kubernetes cluster hence providing HA. More about Longhorn [here](https://longhorn.io/docs/1.1.2/what-is-longhorn/)


This technology also allows backups to be taken on NFS or S3. This documentation discusses an example of disaster recovery using this backup on a separate kubernetes cluster.

## Prerequisites

* Two kubernetes clusters  
    * Source GKE/Anthos cluster to deploy the application 
    * Target GKE/Anthos cluster to restore the application to

* A client workstation to access the kubernetes cluster with kubectl

* If using baremetal cluster, install open-iscsi on all baremetal nodes as this is required

```
apt-get install -y open-iscsi
```

* If using GKE cluster use Ubuntu images for nodes and you will need cluster admin access to the user as explained [here](https://longhorn.io/docs/1.1.2/advanced-resources/os-distro-specific/csi-on-gke/)

## Install Longhorn CSI

We will install longhorn on both the source and target clusters.

Installation based on this [documentation](https://longhorn.io/docs/1.1.2/deploy/install/install-with-kubectl/)


* Deploy longhorn. This creates `longhorn-system` namespace

```
kubectl apply -f https://raw.githubusercontent.com/longhorn/longhorn/v1.1.2/deploy/longhorn.yaml
```

* Watch the pods are all running 

```
kubectl get pods \
--namespace longhorn-system \
--watch
```

## Longhorn Ingress

There are two options. You can either setup kubernetes ingress or simply change the service type to load balancer if your cluster supports it.

### Create Kubernetes ingress

* For Longhorn UI, install ingress as in [here](https://longhorn.io/docs/1.1.2/deploy/accessing-the-ui/longhorn-ingress/). 

* Substitute username and password and create an auth file
```
USER=<USERNAME_HERE>; PASSWORD=<PASSWORD_HERE>; echo "${USER}:$(openssl passwd -stdin -apr1 <<< ${PASSWORD})" >> auth
```
* Create a secret using this auth file

```
kubectl -n longhorn-system create secret generic basic-auth --from-file=auth
```

* Create an ingress file named `longhorn-ingress.yaml` with the following content. 

```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: longhorn-ingress
  namespace: longhorn-system
  annotations:
    # type of authentication
    nginx.ingress.kubernetes.io/auth-type: basic
    # prevent the controller from redirecting (308) to HTTPS
    nginx.ingress.kubernetes.io/ssl-redirect: 'false'
    # name of the secret that contains the user/password definitions
    nginx.ingress.kubernetes.io/auth-secret: basic-auth
    # message to display with an appropriate context why the authentication is required
    nginx.ingress.kubernetes.io/auth-realm: 'Authentication Required '
spec:
  rules:
  - http:
      paths:
      - pathType: Prefix
        path: "/"
        backend:
          service:
            name: longhorn-frontend
            port:
              number: 80
```

* Instantiate Kubernetes ingress

```
kubectl -n longhorn-system apply -f longhorn-ingress.yml
```

* Verify

```
kubectl -n longhorn-system get ingress
```

to see output like below

```
NAME               CLASS    HOSTS   ADDRESS           PORTS   AGE
longhorn-ingress   <none>   *       192.168.122.111   80      2d19h
```

You can use this ip address to access Longhorn UI.

## Configure Google Cloud Storage to store backups on the clusters

Longhorn supports S3 and NFS as storage for backup. We will configure GCS storage bucket as the S3 storage. In order to use it, we need to enable interoperability.

### Configure GCS Storage bucket as S3 storage

* Navigate to `Cloud Storage` from Google Cloud Console and create a new storage bucket (let's say named `longhornbackup`) as explained [here](https://cloud.google.com/storage/docs/creating-buckets)

* Navigate to `Settings` and `Interoperability`. If you have not set up interoperability before, click Enable interoperability access. 

* At the bottom of the page create a new access key. Copy the access key and secret 

* Also note the Request Endpoint for use with other storage systems. This storage URI is `https://storage.googleapis.com`

* Back to CLI. Set these noted values as environment variables

```
export ACCESS_KEY=<<your access key>
export SECRET= <<your secret>>
export ENDPOINT=https://storage.googleapis.com
```

* Create a secret (`gcp-storage-secret`) in with the above variables in the `longhorn-system` namespace

```
kubectl create secret generic gcp-storage-secret \
    --from-literal=AWS_ACCESS_KEY_ID=${ACCESS_KEY} \
    --from-literal=AWS_SECRET_ACCESS_KEY=${SECRET} \
    --from-literal=AWS_ENDPOINTS=${ENDPOINT} \
    -n longhorn-system
```

### Configure backup on LongHorn UI

* Access Longhorn UI in the browser

* Navigate to `Setting` -> `General` on the top menu

* Scrolldown on the page until you find `Backup Target` and set the following values:

    * Backup Target: `s3://longhornbackup@us/`

    * Backup Target Credential Secret: `gcp-storage-secret`

* Scroll down all the way to the bottom and save

* Now switch the `Backup` item in the top menu and you should not see any errors.


## Deploy a sample application that uses Persistent Storage

* Select the cluster on Kubernetes menu in Google Cloud Console

* Navigate to `Market Place` menu on the left bottom

* Search for `wordpress` and press on `configure` button

* Choose the following in the left menu
    * Create new namespace with name `wordpress`
    * Choose storage class `longhorn`



* Create a new namespace with name `wordpress`

```
kubectl create ns wordpress
```

* Create secret

```
kubectl create secret generic mysql-pass \
    --from-literal=password=mysql \
    -n wordpress
```



* Deploy mysql database backend with PVC that uses `longhorn` storage class
    
```
kubectl apply -f https://raw.githubusercontent.com/VeerMuchandi/wordpress/master/mysql-deployment.yaml.withpvc -n wordpress
```

Output

```
service/wordpress-mysql created
persistentvolumeclaim/mysql-pv-claim created
deployment.apps/wordpress-mysql created
```

* Verify mysql pod is running `kubectl get po -n wordpress` and corresponding PVC is good `kubectl get pvc -n wordpress`

```
NAME                              READY   STATUS    RESTARTS   AGE
wordpress-mysql-6c479567b-cbtj7   1/1     Running   0          48s
```

```
NAME             STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
mysql-pv-claim   Bound    pvc-36688974-8803-4fc1-991d-1dfa8c57a561   20Gi       RWO            longhorn       119s
```

* Deploy wordpress

```
kubectl apply -f https://raw.githubusercontent.com/VeerMuchandi/wordpress/master/wordpress-deployment.yaml.withpvc -n wordpress
```
Output

```
service/wordpress created
persistentvolumeclaim/wp-pv-claim created
deployment.apps/wordpress created
```

* Watch the wordpress pod coming up `kubectl get po -n wordpress`

```
wordpress-mysql-6c479567b-cbtj7   1/1     Running             0          3m34s
wordpress-5994d99f46-9j5gq        1/1     Running             0          26s
```

Also note an additional PVC added for the wordpress pod

```
$ kubectl get pvc -n wordpress
NAME             STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
mysql-pv-claim   Bound    pvc-36688974-8803-4fc1-991d-1dfa8c57a561   20Gi       RWO            longhorn       4m55s
wp-pv-claim      Bound    pvc-5eb13322-a154-46cd-a74a-9cd975a1e7b6   20Gi       RWO            longhorn       97s
```

* Note the `wordpress` service is a LoadBalancer type and is assigned an ExternalIP

```
$ kubectl get svc -n wordpress
NAME              TYPE           CLUSTER-IP    EXTERNAL-IP       PORT(S)        AGE
wordpress         LoadBalancer   10.96.2.196   192.168.122.112   80:32202/TCP   2m2s
wordpress-mysql   ClusterIP      None          <none>            3306/TCP       5m20s
```

* Use this External IP to access the application from the browser

* Fill in the details on the welcome page and create a userid and press on the `Install Wordpress` button.

* Sign-in, create a post and publish. View the post after that to make sure the post is visible.

## Back up the persistent volumes with Longhorn

* Switch over to Longhorn UI

* Switch to `Volume` tab and select the two volumes that correspond to wordpress application that were bound to the two PVCs ie., run `kubectl get pvc -n wordpress ` to identify them

* Press on `Create Backup` and then `OK`. It will take a few minutes for the backups to be created. You can click on each each volume URI hyperlink and scroll down to `Snapshots and Backups` section to view the progress. When the backup reaches 100%, it will be available on google cloud storage. Wait until both backups reach 100%


## Restore wordpress application on the target cluster

* Switch the kubernetes context to the the target cluster

* Configure the `Setting` -> `General`, Backup Target and Backup Credential as discussed [here](#configure-backup-on-longhorn-ui)

* In the `Backup` menu, you should see the two backups that you can select and click on the `Create Disaster Recovery Volume` button

* Navigate to the `Volume` menu you will see both the volumes getting downloaded. It may take some time for the copy to complete. Once done, go to the menu under `Operation` table heading for each volume and `Activate Disaster Recovery Volume`

* Create a namespace `wordpress` in the target cluster

```
kubectl create namespace wordpress
```

* Now click the menu under `Operation` again for each volume and choose to `Create PV/PVC`. Leave `Create PVC` and `Use Previous PVC` checked and  set the `Namespace` to  `wordpress`.

* Now check the PVCs on the target cluster running `kubectl get pvc -n wordpress` and you should find both the PVCs bound to the PVs. 


## Deploy the application and verify the data is recovered

Run the following commands to deploy wordpress application using existing PVCs

* Create secret

```
kubectl create secret generic mysql-pass \
    --from-literal=password=mysql \
    -n wordpress
```

* Create mysql deployment
```
kubectl apply -f https://raw.githubusercontent.com/VeerMuchandi/wordpress/master/mysql-deployment.yaml -n wordpress
```
and verify that the mysql pod get deployed

```
$ kubectl get po -n wordpress
NAME                              READY   STATUS    RESTARTS   AGE
wordpress-mysql-6c479567b-lcwzt   1/1     Running   0          2m29s
```

* Wait until the pod is `Running` and run the following command to deploy the wordpress pod

```
kubectl apply -f https://raw.githubusercontent.com/VeerMuchandi/wordpress/master/wordpress-deployment.yaml -n wordpress
```

* Wait until the wordpress starts running

```
$ kubectl get po -n wordpress
NAME                              READY   STATUS    RESTARTS   AGE
wordpress-5994d99f46-6hczr        1/1     Running   0          30s
wordpress-mysql-6c479567b-lcwzt   1/1     Running   0          3m35s
```

* Get the K8S services deployed in the `wordpress` namespace

```
$ kubectl get svc -n wordpress
NAME              TYPE           CLUSTER-IP    EXTERNAL-IP     PORT(S)        AGE
wordpress         LoadBalancer   10.48.7.202   35.193.95.237   80:32602/TCP   67s
wordpress-mysql   ClusterIP      None          <none>          3306/TCP       57m
```

Note that the wordpress service is of type `LoadBalancer` and has an external IP.

* Use that external IP to access the application and notice that the data you added in the previous site is here. 

Celebrate your **disaster recovery**!!







































