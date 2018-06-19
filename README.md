# Helm and Helmsman
Helm is a package manager for kubernetes applications, and helmsman is the automation for deploying these pacakged applications. Helmsman was developed by Sami Alajrami from Praqma.


# Prerequisites:
* A working Kubernetes cluster you have access to
* A technician's computer with kubectl, and helm executables
* A CI server (for helmsman)

# Helm:
First, you need to install Helm on your workstation/technician's computer.
 
## Install Helm on your computer:
Referenes: 
* [https://github.com/kubernetes/helm/blob/master/docs/quickstart.md](https://github.com/kubernetes/helm/blob/master/docs/quickstart.md)
* [https://docs.helm.sh/using_helm/](https://docs.helm.sh/using_helm/)

## Initialize Helm
Before you begin installing helm on your cluster, you should know that `helm init` is used to setup helm's server/cluster side components (**tiller**) on the current cluster you have set up as 'current-context' in your kubectl's config file. This should be kept under consideration when you have multiple clusters configured in your `.kube/config` file. Use `kubectl config current-context` to verify that before proceeding.

```
[kamran@kworkhorse ~]$ kubectl config current-context
gke_praqma-education_europe-north1-a_helm-demo-telenor
[kamran@kworkhorse ~]$ 
```


If helm's tiller is already setup on a cluster, then you will see tiller running as deployment as **tiller-deploy**. However, on each new technician's computer,  you  need to `helm init` , which will setup local repos on the technician's computer. `helm init` will skip tiller installation if it sees that it is already running in the cluster.

```
[kamran@kworkhorse ~]$ helm init
Creating /home/kamran/.helm/repository 
Creating /home/kamran/.helm/repository/cache 
Creating /home/kamran/.helm/repository/local 
Creating /home/kamran/.helm/plugins 
Creating /home/kamran/.helm/starters 
Creating /home/kamran/.helm/repository/repositories.yaml 
Adding stable repo with URL: https://kubernetes-charts.storage.googleapis.com 
Adding local repo with URL: http://127.0.0.1:8879/charts 
$HELM_HOME has been configured at /home/kamran/.helm.
Warning: Tiller is already installed in the cluster.
(Use --client-only to suppress this message, or --upgrade to upgrade Tiller to the current version.)
Happy Helming!
[kamran@kworkhorse ~]$ 
```

After `helm init`, you should be able to run `kubectl get pods --namespace kube-system` and see Tiller running.

```
[kamran@kworkhorse ~]$ kubectl get pods --namespace kube-system
NAME                                                          READY     STATUS    RESTARTS   AGE
event-exporter-v0.1.8-599c8775b7-crjwt                        2/2       Running   0          18h
fluentd-gcp-v2.0.9-j2srp                                      2/2       Running   0          18h
heapster-v1.4.3-7b879f8968-drlcv                              3/3       Running   0          18h
kube-dns-778977457c-xthv9                                     3/3       Running   0          18h
kube-dns-autoscaler-7db47cb9b7-mf44d                          1/1       Running   0          18h
kube-proxy-gke-helm-demo-telenor-default-pool-7c1816fe-bnsc   1/1       Running   0          18h
kubernetes-dashboard-6bb875b5bc-h9ngg                         1/1       Running   0          18h
l7-default-backend-6497bcdb4d-fhmd8                           1/1       Running   0          18h
tiller-deploy-5b9d65c7f-r9h5m                                 1/1       Running   0          5m
[kamran@kworkhorse ~]$ 
```

Once Tiller is installed running `helm version` should show you both the client and server version. If it does not, or shows only the client version, then it means helm cannot yet connect to the server/cluster. Use `kubectl` to check if any tiller pods are running.

```
[kamran@kworkhorse ~]$ helm version
Client: &version.Version{SemVer:"v2.7.2", GitCommit:"8478fb4fc723885b155c924d1c8c410b7a9444e6", GitTreeState:"clean"}
Server: &version.Version{SemVer:"v2.7.2", GitCommit:"8478fb4fc723885b155c924d1c8c410b7a9444e6", GitTreeState:"clean"}
[kamran@kworkhorse ~]$
```


**Note:** If you do not do `helm init`, you will get the following error when you try to do anything useful with helm:
```
[kamran@kworkhorse helmsman-demo]$ helm repo list
Error: open /home/kamran/.helm/repository/repositories.yaml: no such file or directory
```

**Note:** By default, when Tiller is installed,it does not have authentication enabled. To learn more about configuring strong TLS authentication for Tiller, use the [https://docs.helm.sh/using_helm/#using-ssl-between-helm-and-tiller](Tiller TLS guide).


# Special instructions for helm installation on RBAC enabled k8s clusters:

Reference: [https://docs.helm.sh/using_helm/#role-based-access-control](https://docs.helm.sh/using_helm/#role-based-access-control)

If you have RBAC enabled on your cluster, then you need to do few additional steps before installing tiller. First, you need a service account for tiller, and then bind a ClusterRole with this service account. You can do both tasks using one yaml file, with the following contents:
(The file is part of this repository)

```
[kamran@kworkhorse ~]$ vi rbac-config.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: tiller
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: tiller
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: tiller
    namespace: kube-system
``` 

Note: The **cluster-admin** role is created by default in a Kubernetes cluster, so you don’t have to define it explicitly.


Now, use `kubectl create -f <yaml-file>` to create these two objects:

```
[kamran@kworkhorse helmsman-demo]$ kubectl create -f rbac-config.yaml 
serviceaccount "tiller" created
clusterrolebinding "tiller" created
[kamran@kworkhorse helmsman-demo]$ 
```

Now do a helm init specifying the service account used for tiller:
```
[kamran@kworkhorse helmsman-demo]$ helm init --upgrade --service-account tiller
$HELM_HOME has been configured at /home/kamran/.helm.

Tiller (the Helm server-side component) has been upgraded to the current version.
Happy Helming!
[kamran@kworkhorse helmsman-demo]$
```

You should be able to get a list of repositories now:
```
[kamran@kworkhorse helmsman-demo]$ helm repo list
NAME   	URL                                             
stable 	https://kubernetes-charts.storage.googleapis.com
local  	http://127.0.0.1:8879/charts                    
[kamran@kworkhorse helmsman-demo]$ 
```


**Note:** If you are working on kubernetes as a service through some  cloud provider, such as GKE, then you will need correct permissions for the account you are using to access the kubernetes cluster - to be able to setup correct RBAC policies/bindings, otherwise you may see errors like the following:
```
[kamran@kworkhorse helmsman-demo]$ kubectl create -f rbac-config.yaml 
serviceaccount "tiller" created
Error from server (Forbidden): error when creating "rbac-config.yaml": clusterrolebindings.rbac.authorization.k8s.io is forbidden: User "someuser@mycompany.iam.gserviceaccount.com" cannot create clusterrolebindings.rbac.authorization.k8s.io at the cluster scope: Required "container.clusterRoleBindings.create" permission.
[kamran@kworkhorse helmsman-demo]$ 
```

**Note:** If you make any mistake in installing tiller, you can always uninstall it and install it again.
```
[kamran@kworkhorse helmsman-demo]$ helm reset
Tiller (the Helm server-side component) has been uninstalled from your Kubernetes Cluster.
[kamran@kworkhorse helmsman-demo]$ 
```


 

# Using helm:
## Check the (list of) repositories:
First, make sure you have some repositories on your local computer. (You get them after `helm init` step above).

```
[kamran@kworkhorse helmsman-demo]$ helm repo list
NAME  	URL                                             
stable	https://kubernetes-charts.storage.googleapis.com
local 	http://127.0.0.1:8879/charts                    
[kamran@kworkhorse helmsman-demo]$ 
```



## Search for a chart:
```
[kamran@kworkhorse helmsman-demo]$ helm search mysql
NAME                         	VERSION	DESCRIPTION                                       
stable/mysql                 	0.3.7  	Fast, reliable, scalable, and easy to use open-...
stable/percona               	0.3.1  	free, fully compatible, enhanced, open source d...
stable/percona-xtradb-cluster	0.1.4  	free, fully compatible, enhanced, open source d...
stable/gcloud-sqlproxy       	0.3.2  	Google Cloud SQL Proxy                            
stable/mariadb               	3.0.3  	Fast, reliable, scalable, and easy to use open-...
[kamran@kworkhorse helmsman-demo]$	
```



## Installing a chart (package):

`helm install` will deploy/install a helm chart as a **release** . 

```
[kamran@kworkhorse helmsman-demo]$ helm install --name mysql --namespace=default stable/mysql
NAME:   mysql
LAST DEPLOYED: Tue Jun 19 21:58:55 2018
NAMESPACE: default
STATUS: DEPLOYED

RESOURCES:
==> v1/Secret
NAME         TYPE    DATA  AGE
mysql-mysql  Opaque  2     0s

==> v1/PersistentVolumeClaim
NAME         STATUS   VOLUME    CAPACITY  ACCESS MODES  STORAGECLASS  AGE
mysql-mysql  Pending  standard  0s

==> v1/Service
NAME         TYPE       CLUSTER-IP   EXTERNAL-IP  PORT(S)   AGE
mysql-mysql  ClusterIP  10.15.243.9  <none>       3306/TCP  0s

==> v1beta1/Deployment
NAME         DESIRED  CURRENT  UP-TO-DATE  AVAILABLE  AGE
mysql-mysql  1        1        1           0          0s

==> v1/Pod(related)
NAME                          READY  STATUS   RESTARTS  AGE
mysql-mysql-7768bff7c8-t9nzf  0/1    Pending  0         0s


NOTES:
MySQL can be accessed via port 3306 on the following DNS name from within your cluster:
mysql-mysql.default.svc.cluster.local

To get your root password run:

    MYSQL_ROOT_PASSWORD=$(kubectl get secret --namespace default mysql-mysql -o jsonpath="{.data.mysql-root-password}" | base64 --decode; echo)

To connect to your database:

1. Run an Ubuntu pod that you can use as a client:

    kubectl run -i --tty ubuntu --image=ubuntu:16.04 --restart=Never -- bash -il

2. Install the mysql client:

    $ apt-get update && apt-get install mysql-client -y

3. Connect using the mysql cli, then provide your password:
    $ mysql -h mysql-mysql -p

To connect to your database directly from outside the K8s cluster:
    MYSQL_HOST=127.0.0.1
    MYSQL_PORT=3306

    # Execute the following commands to route the connection:
    export POD_NAME=$(kubectl get pods --namespace default -l "app=mysql-mysql" -o jsonpath="{.items[0].metadata.name}")
    kubectl port-forward $POD_NAME 3306:3306

    mysql -h ${MYSQL_HOST} -P${MYSQL_PORT} -u root -p${MYSQL_ROOT_PASSWORD}
    

[kamran@kworkhorse helmsman-demo]$ 
```

Note: If you do not specify a **name**, the names of the installed helm chart (the release) and the resulting deployment will be auto-generated.


## List installed charts (releases) and related kubernetes objects:
```
[kamran@kworkhorse helmsman-demo]$ helm list
NAME 	REVISION	UPDATED                 	STATUS  	CHART      	NAMESPACE
mysql	1       	Tue Jun 19 21:58:55 2018	DEPLOYED	mysql-0.3.7	default  
[kamran@kworkhorse helmsman-demo]$ 
```

```
[kamran@kworkhorse helmsman-demo]$ kubectl get deployments
NAME          DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
multitool     1         1         1            1           1d
mysql-mysql   1         1         1            1           10m


[kamran@kworkhorse helmsman-demo]$ kubectl get pods
NAME                           READY     STATUS    RESTARTS   AGE
multitool-74f649f9cb-68hkf     1/1       Running   0          1d
mysql-mysql-7768bff7c8-t9nzf   1/1       Running   0          10m


[kamran@kworkhorse helmsman-demo]$ kubectl get services
NAME          TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)    AGE
kubernetes    ClusterIP   10.15.240.1   <none>        443/TCP    1d
mysql-mysql   ClusterIP   10.15.243.9   <none>        3306/TCP   10m
[kamran@kworkhorse helmsman-demo]$ 
```

```
[kamran@kworkhorse helmsman-demo]$ kubectl get pvc
NAME          STATUS    VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
mysql-mysql   Bound     pvc-32dfe3af-73fb-11e8-a57a-42010aa6000b   8Gi        RWO            standard       13m


[kamran@kworkhorse helmsman-demo]$ kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS    CLAIM                 STORAGECLASS   REASON    AGE
pvc-32dfe3af-73fb-11e8-a57a-42010aa6000b   8Gi        RWO            Delete           Bound     default/mysql-mysql   standard                 12m
[kamran@kworkhorse helmsman-demo]$
```

```
[kamran@kworkhorse helmsman-demo]$ kubectl get secrets
NAME                  TYPE                                  DATA      AGE
default-token-kqnjc   kubernetes.io/service-account-token   3         1d
mysql-mysql           Opaque                                2         11m


[kamran@kworkhorse helmsman-demo]$ kubectl get configmaps
No resources found.
[kamran@kworkhorse helmsman-demo]$ 
```

## Verify that you can use the deployed chart:

We need to connect to the mysql service from within the cluster. For that we already have a multitool running inside the cluster, which has mysql client installed in it. We also need the MYSQL_ROOT_PASSWORD for the current deployment.

```
[kamran@kworkhorse helmsman-demo]$ MYSQL_ROOT_PASSWORD=$(kubectl get secret --namespace default mysql-mysql -o jsonpath="{.data.mysql-root-password}" | base64 --decode; echo)
[kamran@kworkhorse helmsman-demo]$ echo $MYSQL_ROOT_PASSWORD 
UUoATlBI3Q
[kamran@kworkhorse helmsman-demo]$
```

Connect to the multitool and from there use mysql client to connect to the freshly-setup mysql service:
```
[kamran@kworkhorse helmsman-demo]$ kubectl exec -it multitool-74f649f9cb-68hkf bash

[root@multitool-74f649f9cb-68hkf /]# mysql -u root -h mysql-mysql.default.svc.cluster.local -p
Enter password: 
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MySQL connection id is 226
Server version: 5.7.14 MySQL Community Server (GPL)

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MySQL [(none)]> 
```

Hurray!


## Delete a helm release:
```
[kamran@kworkhorse helmsman-demo]$ helm delete mysql
release "mysql" deleted
[kamran@kworkhorse helmsman-demo]$
```

Deleting a release also releases it's volumes (PVC). But if you `rollback` , you will get it back and the pvc volumes will be mounted again.


## Rollback a deletion operation:
```
[kamran@kworkhorse helmsman-demo]$ helm rollback mysql 1
Rollback was a success! Happy Helming!
[kamran@kworkhorse helmsman-demo]$
```



