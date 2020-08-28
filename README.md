# Red Hat OpenShift AMQ and Serverless
Making OpenShift AMQ and Serverless work together; the magic of Kafka and Knative combined!~


## Prerequisites
1.  OCP 4.5
2.  Red Hat AMQ Operator Installed
3.  Red Hat Serverless Operator Installed


## Setting Up AMQ in OCP
These notes are taken from the great [upstream docs](https://strimzi.io/quickstarts/).

1.  oc apply -f https://strimzi.io/examples/latest/kafka/kafka-persistent-single.yaml

2.  wait for creation
```
oc wait kafka/my-cluster --for=condition=Ready --timeout=300s
```

3.  Producer
```
oc run kafka-producer -ti --image=strimzi/kafka:0.19.0-kafka-2.5.0 --rm=true --restart=Never -- bin/kafka-console-producer.sh --broker-list my-cluster-kafka-bootstrap:9092 --topic my-topic
```

4.  Consumer
```
oc run kafka-consumer -ti --image=strimzi/kafka:0.19.0-kafka-2.5.0 --rm=true --restart=Never -- bin/kafka-console-consumer.sh --bootstrap-server my-cluster-kafka-bootstrap:9092 --topic my-topic --from-beginning
```


## Hooking Knative Up With Kafka
Instructions heavily influenced by [this tutorial](https://redhat-developer-demos.github.io/knative-tutorial/knative-tutorial-adv/eventing-with-kafka.html).

1.  Create projects
```
oc new-project knativetutorial
oc new-project kafka
```

2.  Install Operator
Should be able to skip since we are using RH AMQ.  Run the verify steps first and see if all those things exist.

```
curl -L \
https://github.com/strimzi/strimzi-kafka-operator\
/releases/download/0.17.0/strimzi-cluster-operator-0.17.0.yaml \
  | sed 's/namespace:.*/namespace: kafka/' \
  | kubectl apply -n kafka -f -

# verify, look for 'strimzi-cluster-operator'
watch "kubectl get pods -n kafka"

# verify all api's exist
# kafkabridges, kafkaconnectors, kafkaconnects, kafkaconnects2is,
#  kafkamirrormaker2s, kafkamirrormakers, kafkas, kafkatopics, kafkausers
kubectl api-resources --api-group='kafka.strimzi.io'
```

3.  Create Cluster
```
oc apply -f eventing/kafka-broker-my-cluster.yaml

# verify, warning this takes forever
# run the following and wait for entity-operator, cluster, and zookeeper
watch "kubectl get pods -n kafka"
```

4.  Create Topic
```
kubectl -n kafka create -f eventing/kafka-topic-my-topic.yaml

# verify, see the topic
kubectl -n kafka get kafkatopics

# test sending and receiving
/bin/kafka-producer.sh
/bin/kafka-consumer.sh
```

5.  Create Knative Eventing Source
```
kubectl apply -f https://github.com/knative/eventing-contrib/releases/download/v0.14.1/kafka-source.yaml

# verify
watch "kubectl get pods -n knative-sources"
```

6.  Create Knative Kafka Channel (Connects Knative Eventing Channel with Kafka)
```
# run the following and important thing to note is "my-cluster-kafka-bootstrap.kafka:9092"
kubectl get services -n kafka

curl -L "https://github.com/knative/eventing-contrib/releases/download/v0.14.1/kafka-channel.yaml" \
 | sed 's/REPLACE_WITH_CLUSTER_URL/my-cluster-kafka-bootstrap.kafka:9092/' \
 | kubectl apply --filename -

# verify takes a while
watch "kubectl get pods -n knative-eventing"

# verify that the api is there
kubectl api-resources --api-group='sources.knative.dev'
```

7.  Setup Kafka Channel as Default Knative Channel
```
oc apply -f eventing/default-kafka-channel.yaml
```

8.  Create Kafka Source to Knative Sink
```
oc apply -n knativetutorial -f eventing/eventing-hello-sink.yaml
kubectl get ksvc -n knativetutorial

# you can follow logs
stern eventinghello -c user-container

# create topic
kubectl apply -n kafka -f eventing/kafka-topic-my-topic.yaml

# verify
kubectl get -n kafka kafkatopics

# create kafka source, but first epic hack (https://github.com/knative/docs/issues/2124)
# wait a bit to let the delete actually happen then try again if the oc apply fails
oc delete mutatingwebhookconfigurations.admissionregistration.k8s.io sinkbindings.webhook.sources.knative.dev
oc delete mutatingwebhookconfigurations.admissionregistration.k8s.io legacysinkbindings.webhook.sources.knative.dev
oc -n knativetutorial apply -f eventing/mykafka-source.yaml

# verify, look for anything that looks like 'mykafka-source'
watch kubectl get pods
```

9.  Test everything
```
# In another see the logs
stern eventinghello -c user-container

# In one terminal run the producer to produce events
./bin/kafka-producer.sh
```
