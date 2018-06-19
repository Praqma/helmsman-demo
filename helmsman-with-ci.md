# Using Helmsman through CircleCI

## Setup structure for CircleCI:
Circle CI expects a `.circleci` directory in the root directory of your project's git repo. Lets create it and setup necessary files under it:

```
[kamran@kworkhorse helmsman-demo]$ mkdir .circleci

[kamran@kworkhorse helmsman-demo]$ cd .circleci/

[kamran@kworkhorse .circleci]$ vi config.yaml
version: 2
jobs:
  deploy-helm-charts:
    docker:
      - image: praqma/helmsman:v1.3.0-rc
    steps:
      - checkout
      - run:
        name: Deploy Helm charts/packages using Helmsman
        command: helmsman -debug -apply -f helmsman-deployments.toml

workflows:
  version: 2
  build:
    jobs:
      - deploy-helm-charts
[kamran@kworkhorse .circleci]$
```

## Create additional namespaces [optional]:
We create few namespaces just to show how helmsman can setup applications in different namespaces.
```
[kamran@kworkhorse helmsman-demo]$ kubectl create namespace production
namespace "production" created
[kamran@kworkhorse helmsman-demo]$ kubectl create namespace development
namespace "development" created
[kamran@kworkhorse helmsman-demo]$ 
```


## Setup applications values file(s):
Next, we setup a directory structure for the applications we need to deploy, and place necessary **values files** in these directories. We also need a `helmsman-deployments.toml` file file in the project root.

Ideally, under the applications directory, you should have a directory for each corresponding namespace you have in your kubernetes cluster. 
```
[kamran@kworkhorse .circleci]$ cd ..
[kamran@kworkhorse helmsman-demo]$ mkdir applications/development -p
[kamran@kworkhorse helmsman-demo]$ mkdir applications/production -p

[kamran@kworkhorse helmsman-demo]$ tree -d applications/
applications/
├── development
└── production

2 directories
[kamran@kworkhorse helmsman-demo]$ 
```


In this example I will show the deployment of same mysql chart through helmsman. This means, I will need a **values file** for mysql deployment. Also I have a namespace named **development** in my example  cluster, which I will use to deploy mysql in.

Normally the provider of the helm chart provides you with a sample/example values file too. In mysql's case, I am using the values file from: [https://raw.githubusercontent.com/kubernetes/charts/master/stable/mysql/values.yaml](https://raw.githubusercontent.com/kubernetes/charts/master/stable/mysql/values.yaml) . Create `mysql-development.values` file in `applications/development` directory. Enable the configuration items you are interested in. In the exmaple below, only the items used in the demo are shown.

```
[kamran@kworkhorse helmsman-demo]$ vi applications/development/mysql-development.values

image: "mysql"
imageTag: "5.7.14"

imagePullPolicy: IfNotPresent

nodeSelector: {}

livenessProbe:
  initialDelaySeconds: 30
  periodSeconds: 10
  timeoutSeconds: 5
  successThreshold: 1
  failureThreshold: 3

readinessProbe:
  initialDelaySeconds: 5
  periodSeconds: 10
  timeoutSeconds: 1
  successThreshold: 1
  failureThreshold: 3

 ## Persist data to a persistent volume
persistence:
  enabled: true
  ## database data Persistent Volume Storage Class
  ## If defined, storageClassName: <storageClass>
  ## If set to "-", storageClassName: "", which disables dynamic provisioning
  ## If undefined (the default) or set to null, no storageClassName spec is
  ##   set, choosing the default provisioner.  (gp2 on AWS, standard on
  ##   GKE, AWS & OpenStack)
  ##
  # storageClass: "-"
  accessMode: ReadWriteOnce
  size: 2Gi

 ## Configure resource requests and limits
resources:
  requests:
    memory: 256Mi
    cpu: 100m

## Configure the service
service:
  type: ClusterIP
  port: 3306
```

## Setup helmsman-deployments.toml file:
Now we setup `helmsman-deployments.toml` file in the root directory of the project's git repo.

```
[kamran@kworkhorse helmsman-demo]$ vi helmsman-deployments.toml
[metadata]
org = "example.com"
maintainer = "k8s-admin (me@example.com)"
description = "example Desired State File for demo purposes."
[certificates]

[settings]
kubeContext = "gke_praqma-education_europe-north1-a_helm-demo-telenor" # will try connect to this context first, if it does not exist, it will be created using the details below
storageBackend = "secret" # default is configMap

[namespaces]
  [namespaces.production]
  protected = true
  [namespaces.development]
   protected = false
[helmRepos]
stable = "https://kubernetes-charts.storage.googleapis.com"
incubator = "http://storage.googleapis.com/kubernetes-charts-incubator"

[apps]
    [apps.]
    name = "mysql" # should be unique across all apps
    description = "mysql"
    namespace = "development" # maps to the namespace as defined in namespaces above
    enabled = true # change to false if you want to delete this app release [empty = false]
    chart = "stable/mysql" # changing the chart name means delete and recreate this chart
    version = "0.3.7" # chart version
    valuesFile = "applications/development/mysql-development.values" # leaving it empty uses the default chart values
    purge = false # will only be considered when there is a delete operation
    test = false # run the tests when this release is installed for the first time only
    protected = false
    priority= -3
    wait = true
    [apps.artifactory]
    name = "artifactory" # should be unique across all apps
    description = "artifactory"
    namespace = "production" # maps to the namespace as defined in namespaces above
    enabled = false # change to false if you want to delete this app release [empty = flase]
    chart = "stable/artifactory" # changing the chart name means delete and recreate this chart
    version = "7.0.6" # chart version
    valuesFile = "" # leaving it empty uses the default chart values
    purge = false # will only be considered when there is a delete operation
    test = false # run the tests when this release is installed for the first time only
    priority= -2
[kamran@kworkhorse helmsman-demo]$ 
```




