# lagom-java-chirper-example

This project is an example for [Lightbend Reactive Platform Tooling](https://developer.lightbend.com/docs/reactive-platform-tooling/latest/).

It's a Lagom Java example showcasing a Twitter-like application

## Local JVM

* Start all services using `sbt runAll`.

## Minikube

#### Prerequisites

* [Docker](https://www.docker.com/)
* [Helm](https://github.com/kubernetes/helm)
* [Minikube](https://github.com/kubernetes/minikube)
* [SBT](http://www.scala-sbt.org/)

### Clone the Chirper sample repo

This GitHub repository is enabled with Lightbend Orchestration for Kubernetes:

`git clone https://github.com/longshorej/lagom-java-chirper-tooling-example.git`

### Build & Deploy

#### 1. Environment Preparation

##### Install reactive-cli

See [Reactive Platform Tooling Documentation](https://developer.lightbend.com/docs/reactive-platform-tooling/latest/#install-the-cli)

Ensure you're using `reactive-cli` 0.9.0 or newer. You can check the version with `rp version`.

##### Start minikube

> If you have an existing Minikube, you can delete your old one start fresh via `minikube delete`

```bash
minikube start --memory 6000
```

##### Setup Docker engine context to point to Minikube

```bash
eval $(minikube docker-env)
```

##### Enable Ingress Controller

```bash
minikube addons enable ingress
```

#### 2a. Development Workflow

> Note that this is an alternative to the Operations workflow documented below.

`sbt-reactive-app` defines a task, `deploy minikube`, that can be used to deploy all aggregated subprojects to
your running Minikube. It also installs the [Reactive Sandbox](https://github.com/lightbend/reactive-sandbox/) if
your project needs it, e.g. for Lagom applications that use Cassandra or Kafka.

##### Build and Deploy Project

Run sbt:

`sbt`

Then at the prompt:

`> deploy minikube`

Once completed, Chirper and its dependencies should be installed in your cluster. Continue with step 3, Verify Deployment.

#### 2b. Operations Workflow

> Note that this is an alternative to the Development workflow documented above.

##### Install Reactive Sandbox

The `reactive-sandbox` includes development-grade (i.e. it will lose your data) installations of Cassandra, Elasticsearch, Kafka, and ZooKeeper. It's packaged as a Helm chart for easy installation into your Kubernetes cluster.

> Note that if you have an external Cassanda cluster, you can skip this step. You'll need to change the `cassandra_svc` variable (defined below) if this is the case.

```bash
helm init
helm repo add lightbend-helm-charts https://lightbend.github.io/helm-charts
helm update
```

Verify that Helm is available (this takes a minute or two):

```bash
kubectl --namespace kube-system get deploy/tiller-deploy
```

```
NAME            DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
tiller-deploy   1         1         1            1           3m
```

Install the sandbox. Since Chirper only uses Cassandra, we're disabling the other services but you can leave them enabled by omitting the `set` flag if you wish.

```bash
helm install lightbend-helm-charts/reactive-sandbox --name reactive-sandbox --set elasticsearch.enabled=false,kafka.enabled=false,zookeeper.enabled=false
```

Verify that it is available (this takes a minute or two):

```bash
kubectl get deploy/reactive-sandbox
```

```
NAME               DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
reactive-sandbox   1         1         1            1           1m
```

##### Build Project

```bash
sbt clean docker:publishLocal
```

##### View Images

```bash
docker images
```

##### Deploy Projects

Finally, you're ready to deploy the services. Be sure to adjust the secret variables and cassanda service address as necessary.

```bash
# Be sure to change these secret values

chirp_secret="youmustchangeme"
friend_secret="youmustchangeme"
activity_stream_secret="youmustchangeme"
front_end_secret="youmustchangeme"

# Default address for reactive-sandbox, change if using external Cassandra

cassandra_svc="_cql._tcp.reactive-sandbox-cassandra.default.svc.cluster.local"

# Configure the services to allow requests to the minikube IP (Play's Allowed Hosts Filter)

allowed_host="$(minikube ip)"

# deploy chirp-impl

rp generate-kubernetes-resources "chirp-impl:1.0.0-SNAPSHOT" \
  --generate-pod-controllers --generate-services \
  --env JAVA_OPTS="-Dplay.http.secret.key=$chirp_secret -Dplay.filters.hosts.allowed.0=$allowed_host" \
  --external-service "cas_native=$cassandra_svc" \
  --service-type NodePort \
  --pod-controller-replicas 2 | kubectl apply -f -

# deploy friend-impl

rp generate-kubernetes-resources "friend-impl:1.0.0-SNAPSHOT" \
  --generate-pod-controllers --generate-services \
  --env JAVA_OPTS="-Dplay.http.secret.key=$friend_secret -Dplay.filters.hosts.allowed.0=$allowed_host" \
  --external-service "cas_native=$cassandra_svc" \
  --pod-controller-replicas 2 | kubectl apply -f -
  
# deploy activity-stream-impl

rp generate-kubernetes-resources "activity-stream-impl:1.0.0-SNAPSHOT" \
  --generate-pod-controllers --generate-services \
  --env JAVA_OPTS="-Dplay.http.secret.key=$activity_stream_secret -Dplay.filters.hosts.allowed.0=$allowed_host" | kubectl apply -f -
  
# deploy front-end

rp generate-kubernetes-resources "front-end:1.0.0-SNAPSHOT" \
  --generate-pod-controllers --generate-services \
  --env JAVA_OPTS="-Dplay.http.secret.key=$front_end_secret -Dplay.filters.hosts.allowed.0=$allowed_host" | kubectl apply -f -
  
# deploy ingress
rp generate-kubernetes-resources \
  --ingress-path-suffix '*' \
  --generate-ingress --ingress-name chirper \
  "$registry/chirp-impl:1.0.0-SNAPSHOT" \
  "$registry/friend-impl:1.0.0-SNAPSHOT" \
  "$registry/activity-stream-impl:1.0.0-SNAPSHOT" \
  "$registry/front-end:1.0.0-SNAPSHOT" | kubectl apply -f -
```

#### 3. Verify Deployment

Now that you've deployed your services (using either the Developer or Operations workflows), you can use `kubectl` to 
inspect the resources, and your favorite web browser to use the application.

> See the resources created for you

```bash
kubectl get all
```

```
$ kubectl get all
NAME                                     DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
deploy/activityservice-v1-0-0-snapshot   1         1         1            1           29m
deploy/chirpservice-v1-0-0-snapshot      1         1         1            1           29m
deploy/friendservice-v1-0-0-snapshot     1         1         1            1           29m
deploy/front-end-v1-0-0-snapshot         1         1         1            1           29m
deploy/loadtestservice-v1-0-0-snapshot   1         1         1            0           29m
deploy/reactive-sandbox                  1         1         1            1           31m

NAME                                            DESIRED   CURRENT   READY     AGE
rs/activityservice-v1-0-0-snapshot-8b6d9b59c    1         1         1         29m
rs/chirpservice-v1-0-0-snapshot-7ddb9c86b5      1         1         1         29m
rs/friendservice-v1-0-0-snapshot-5bc56f9cc9     1         1         1         29m
rs/front-end-v1-0-0-snapshot-5979dfd7f          1         1         1         29m
rs/loadtestservice-v1-0-0-snapshot-5f598666c5   1         1         0         29m
rs/reactive-sandbox-75fc9fbfd4                  1         1         1         31m

NAME                                     DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
deploy/activityservice-v1-0-0-snapshot   1         1         1            1           29m
deploy/chirpservice-v1-0-0-snapshot      1         1         1            1           29m
deploy/friendservice-v1-0-0-snapshot     1         1         1            1           29m
deploy/front-end-v1-0-0-snapshot         1         1         1            1           29m
deploy/loadtestservice-v1-0-0-snapshot   1         1         1            0           29m
deploy/reactive-sandbox                  1         1         1            1           31m

NAME                                            DESIRED   CURRENT   READY     AGE
rs/activityservice-v1-0-0-snapshot-8b6d9b59c    1         1         1         29m
rs/chirpservice-v1-0-0-snapshot-7ddb9c86b5      1         1         1         29m
rs/friendservice-v1-0-0-snapshot-5bc56f9cc9     1         1         1         29m
rs/front-end-v1-0-0-snapshot-5979dfd7f          1         1         1         29m
rs/loadtestservice-v1-0-0-snapshot-5f598666c5   1         1         0         29m
rs/reactive-sandbox-75fc9fbfd4                  1         1         1         31m

NAME                                                  READY     STATUS             RESTARTS   AGE
po/activityservice-v1-0-0-snapshot-8b6d9b59c-9gb69    1/1       Running            0          29m
po/chirpservice-v1-0-0-snapshot-7ddb9c86b5-f79sf      1/1       Running            0          29m
po/friendservice-v1-0-0-snapshot-5bc56f9cc9-2plgk     1/1       Running            0          29m
po/front-end-v1-0-0-snapshot-5979dfd7f-hbxd6          1/1       Running            0          29m
po/loadtestservice-v1-0-0-snapshot-5f598666c5-skqw2   0/1       CrashLoopBackOff   10         29m
po/reactive-sandbox-75fc9fbfd4-zhbn9                  1/1       Running            0          31m

NAME                                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                         AGE
svc/activityservice                  ClusterIP   10.110.68.12    <none>        10000/TCP                       29m
svc/chirpservice                     ClusterIP   10.99.213.170   <none>        10000/TCP,10001/TCP,10002/TCP   29m
svc/friendservice                    ClusterIP   10.105.26.63    <none>        10000/TCP,10001/TCP,10002/TCP   29m
svc/front-end                        ClusterIP   10.99.30.84     <none>        10000/TCP                       29m
svc/kubernetes                       ClusterIP   10.96.0.1       <none>        443/TCP                         43m
svc/loadtestservice                  ClusterIP   10.98.71.47     <none>        10000/TCP                       29m
svc/reactive-sandbox-cassandra       ClusterIP   None            <none>        9042/TCP                        31m
svc/reactive-sandbox-elasticsearch   ClusterIP   None            <none>        9200/TCP                        31m
svc/reactive-sandbox-kafka           ClusterIP   None            <none>        9092/TCP                        31m
svc/reactive-sandbox-zookeeper       ClusterIP   None            <none>        2181/TCP                        31m
```

> Open the URL this command prints in the browser

```bash
echo "http://$(minikube ip)"
```
