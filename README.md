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
[kamran@kworkhorse applications]$ helm repo list
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

( TODO to do todo)


 

# Using helm:
## Check the (list of) repositories:
First, make sure you have some repositories on your local computer. (You get them after `helm init` step above).

```
[kamran@kworkhorse applications]$ helm repo list
NAME  	URL                                             
stable	https://kubernetes-charts.storage.googleapis.com
local 	http://127.0.0.1:8879/charts                    
[kamran@kworkhorse applications]$ 
```


# Installing a chart (package):

```
[kamran@kworkhorse applications]$ helm install stable/mysql
NAME:   snug-beetle
LAST DEPLOYED: Mon Dec 11 11:10:48 2017
NAMESPACE: default
STATUS: DEPLOYED

RESOURCES:
==> v1/Secret
NAME               TYPE    DATA  AGE
snug-beetle-mysql  Opaque  2     1s

==> v1/PersistentVolumeClaim
NAME               STATUS  VOLUME                                    CAPACITY  ACCESS MODES  STORAGECLASS  AGE
snug-beetle-mysql  Bound   pvc-8f58a324-de5b-11e7-8a7e-02412acf5adc  8Gi       RWO           gp2           1s

==> v1/Service
NAME               TYPE       CLUSTER-IP      EXTERNAL-IP  PORT(S)   AGE
snug-beetle-mysql  ClusterIP  100.64.124.209  <none>       3306/TCP  1s

==> v1beta1/Deployment
NAME               DESIRED  CURRENT  UP-TO-DATE  AVAILABLE  AGE
snug-beetle-mysql  1        1        1           0          1s

==> v1/Pod(related)
NAME                                READY  STATUS   RESTARTS  AGE
snug-beetle-mysql-1228755901-nkthh  0/1    Pending  0         1s


NOTES:
MySQL can be accessed via port 3306 on the following DNS name from within your cluster:
snug-beetle-mysql.default.svc.cluster.local

To get your root password run:

    kubectl get secret --namespace default snug-beetle-mysql -o jsonpath="{.data.mysql-root-password}" | base64 --decode; echo

To connect to your database:

1. Run an Ubuntu pod that you can use as a client:

    kubectl run -i --tty ubuntu --image=ubuntu:16.04 --restart=Never -- bash -il

2. Install the mysql client:

    $ apt-get update && apt-get install mysql-client -y

3. Connect using the mysql cli, then provide your password:
    $ mysql -h snug-beetle-mysql -p

[kamran@kworkhorse applications]$ 
```



# List installed charts / packages:
```
[kamran@kworkhorse applications]$ helm ls
NAME      	REVISION	UPDATED                 	STATUS  	CHART           	NAMESPACE  
bitbucket 	7       	Wed Dec  6 15:47:19 2017	DEPLOYED	bitbucket-0.1.3 	default    
confluence	7       	Wed Dec  6 15:47:16 2017	DEPLOYED	confluence-0.1.3	default    
jenkins   	9       	Wed Dec  6 15:47:25 2017	DEPLOYED	jenkins-0.9.0   	staging    
jira      	8       	Fri Dec  8 13:49:47 2017	DEPLOYED	jira-0.1.8      	default    
traefik   	20      	Wed Dec  6 15:50:25 2017	DEPLOYED	traefik-1.14.2  	kube-system
snug-beetle	1       	Mon Dec 11 11:10:48 2017	DEPLOYED	mysql-0.3.0     	default    
[kamran@kworkhorse applications]$ 
```
Notice mysql is deployed with the name `snug-beetle` ! (I don't know why! :( )

# Delete a deployment:
```
[kamran@kworkhorse applications]$ helm delete confluence
release "confluence" deleted
[kamran@kworkhorse applications]$
```

Deleting a chart/package also releases it's volumes (PVC). But if you `rollback` , you will get it back and the pvc volumes will be mounted again.

# Rollback a deletion operation:
```
[kamran@kworkhorse code-as-code]$ helm rollback confluence 7
Rollback was a success! Happy Helming!
[kamran@kworkhorse code-as-code]$
```


## Verify that confluence PVC  is obtained and bound:
```
[kamran@kworkhorse code-as-code]$ kubectl get pvc
NAME                                         STATUS    VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
bitbucket-persistent-storage-bitbucket-0     Bound     pvc-c824c7e1-da6f-11e7-8a7e-02412acf5adc   50Gi       RWO            gp2            4d
confluence-persistent-storage-confluence-0   Bound     pvc-09248c9f-de56-11e7-8a7e-02412acf5adc   50Gi       RWO            gp2            42m
jira-persistent-storage-jira-0               Bound     pvc-46359504-dc16-11e7-8a7e-02412acf5adc   50Gi       RWO            gp2            2d
snug-beetle-mysql                            Bound     pvc-8f58a324-de5b-11e7-8a7e-02412acf5adc   8Gi        RWO            gp2            3m
[kamran@kworkhorse code-as-code]$ 
```


