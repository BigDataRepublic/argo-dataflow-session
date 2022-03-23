# Hands on as part of the Argo Dataflow Knowledge Session

This repository contains the example code to use for the hands on.

### Install a working Kafka using Helm

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
kubectl create ns argo-kafka
kubectl apply -f argo_yamls/kafka_secret.yaml
helm install kafka bitnami/kafka --namespace=argo-kafka
```

You can access the cluster using the following (as mentioned when installing)
```bash
kubectl run kafka-client --restart='Never' --image docker.io/bitnami/kafka:3.1.0-debian-10-r49 --namespace argo-kafka --command -- sleep infinity
kubectl exec --tty -i kafka-client --namespace argo-kafka -- bash

kafka-console-consumer.sh --bootstrap-server kafka.argo-kafka.svc.cluster.local:9092 --topic <topic> --from-beginning
kafka-console-producer.sh --bootstrap-server kafka.argo-kafka.svc.cluster.local:9092 --topic <topic>
```

### Setting up environment

As described on [Quick start](https://github.com/argoproj-labs/argo-dataflow/blob/main/docs/QUICK_START.md)
We first assume that you have minikube running (otherwise, see [instructions](https://minikube.sigs.k8s.io/docs/start/)).

Deploy into the `argo-dataflow-system` namespace:

```bash
kubectl apply -f https://raw.githubusercontent.com/argoproj-labs/argo-dataflow/main/config/quick-start.yaml
```

Change to the installation namespace:

```bash
kubectl config set-context --current --namespace=argo-dataflow-system
```

If you want the user interface:

```bash
kubectl apply -f https://raw.githubusercontent.com/argoproj-labs/argo-dataflow/main/config/apps/argo-server.yaml
kubectl get deploy -w ;# (ctrl+c when available)
kubectl port-forward svc/argo-server 2746:2746
```

Open [http://localhost:2746/pipelines/argo-dataflow-system](http://localhost:2746/pipelines/argo-dataflow-system).

### Starting a demo job

Lets first try to run the example flatten expand pipeline to check whether the system is working and that kafka is available.

In the example we reference the kafka secret ([see configuration](https://github.com/argoproj-labs/argo-dataflow/blob/main/docs/CONFIGURATION.md)) that should have been created together with the cluster.

```bash
kubectl apply -f argo_yamls/102-flatten-expand-pipeline.yaml
```

It should now show up in the GUI that you can access through the tunnel.

Check out the sidecar logs to see the logging output being written out.

### Starting the input data flow for the hand on exercise

We created a very amazing inline yaml python job to generate data that resembles table football games:
```bash
kubectl apply -f argo_yamls/kickerscore_kafka_input.yaml
```

It will generate match outputs every 5 seconds. And every 5 minutes it will trigger the end of a "Tournament".

Using this information, take a look at the input data using the kafka tools available mentioned above, and implement the missing blocks and apply:
```bash
kubectl apply -f argo_yamls/kickerscore_pipeline.yaml
```

We already implemented some extra stuff. Feel free to change and do random thing ;). The log sinks can be added to have a better grasp of what the processors are doing.

Take a look at the [docs](https://github.com/argoproj-labs/argo-dataflow/tree/main/docs), and [examples](https://github.com/argoproj-labs/argo-dataflow/tree/main/examples).
