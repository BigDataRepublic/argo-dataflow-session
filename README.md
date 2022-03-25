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

Using this information, take a look at the input data using the kafka tools available mentioned above.

Now try to implement the steps below, either using the [python dsl](https://pypi.org/project/argo-dataflow/), 
or simply add to the `kickerscore_pipeline.yaml` and apply it with:
```bash
kubectl apply -f argo_yamls/kickerscore_pipeline.yaml
```

#### Step 1:
Implement something that makes sure that only valid teams pass, that means to exclude invalid teams and cheaters.

Events to expect on `kicker_results` topic are:
```json lines
{ "winner": { "team": "Invalid Teamname", "score": 10 }, "loser": { "team": "Diamond Cutters", "score": 1 } }
{ "winner": { "team": "Diamond Cutters", "score": 0 }, "loser": { "team": "Nunchuk Racers", "score": 2 } }
{ "winner": { "team": "Shark Panthers", "score": 10 }, "loser": { "team": "Invalid Teamname", "score": 8 } }
{ "winner": { "team": "Nunchuk Racers", "score": 10 }, "loser": { "team": "Invalid Teamname", "score": 1 } }
{ "winner": { "team": "Deadly Stingers", "score": 10 }, "loser": { "team": "Nunchuk Racers", "score": 2 } }
{ "winner": { "team": "Shark Panthers", "score": 10 }, "loser": { "team": "Alpha Flyers", "score": 2 } }
{ "winner": { "team": "Risky Business", "score": 10 }, "loser": { "team": "Nunchuk Racers", "score": 2 } }
{ "winner": { "team": "Cheaters", "score": 4 }, "loser": { "team": "Invalid Teamname", "score": 1 } }
{ "winner": { "team": "Nunchuk Racers", "score": 0 }, "loser": { "team": "Shark Panthers", "score": 8 } }
{ "winner": { "team": "Deadly Stingers", "score": 10 }, "loser": { "team": "Alpha Flyers", "score": 1 } }
```
Filter out events where the following teams are involved: Invalid Teamname & Cheaters and write them to topic `kicker_filter_teams`.
The events on this topic should look like this:
```json lines
{ "winner": { "team": "Diamond Cutters", "score": 0 }, "loser": { "team": "Nunchuk Racers", "score": 2 } }
{ "winner": { "team": "Deadly Stingers", "score": 10 }, "loser": { "team": "Nunchuk Racers", "score": 2 } }
{ "winner": { "team": "Shark Panthers", "score": 10 }, "loser": { "team": "Alpha Flyers", "score": 2 } }
{ "winner": { "team": "Risky Business", "score": 10 }, "loser": { "team": "Nunchuk Racers", "score": 2 } }
{ "winner": { "team": "Nunchuk Racers", "score": 0 }, "loser": { "team": "Shark Panthers", "score": 8 } }
{ "winner": { "team": "Deadly Stingers", "score": 10 }, "loser": { "team": "Alpha Flyers", "score": 1 } }
```
Optionally you can also try to filter out invalid winners that do not have a score of 10.

#### Step 2:
Implement a mapping to only extract the winner data from the json messages

So the output event written on topic `kicker_winners`:
```json lines
{ "team": "Diamond Cutters", "score": 0 } // <- these are some weird winners
{ "team": "Deadly Stingers", "score": 10 }
{ "team": "Shark Panthers", "score": 10 }
{ "team": "Risky Business", "score": 10 }
{ "team": "Nunchuk Racers", "score": 0 } // <- these are some weird winners
{ "team": "Deadly Stingers", "score": 10 }
```

#### Step 3:
Using that information, you can create additional steps to process the events to something like the following:
```json lines
{ "champion": "Shark Panthers", "at": "2022-03-23 21:28:13.828344", "wins": 8 }
{ "champion": "Diamond Cutters", "at": "2022-03-23 21:33:14.562645", "wins": 10 }
{ "champion": "Deadly Stingers", "at": "2022-03-23 22:04:53.251637", "wins": 0 }
{ "champion": "Diamond Cutters", "at": "2022-03-23 22:08:42.209762", "wins": 3 }
{ "champion": "Shark Panthers", "at": "2022-03-23 22:13:42.144747", "wins": 6 }
{ "champion": "Alpha Flyers", "at": "2022-03-23 22:18:42.633291", "wins": 4 }
{ "champion": "Diamond Cutters", "at": "2022-03-23 22:23:43.467081", "wins": 14 }
```
To get the above, you can use the championship triggers on the `kicker_timer` topic.

Just have some fun changing random things. See the `feat/solutions` for the complete pipeline.

### For more information
Take a look at the [docs](https://github.com/argoproj-labs/argo-dataflow/tree/main/docs), and [examples](https://github.com/argoproj-labs/argo-dataflow/tree/main/examples).
