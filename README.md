

# Kafka disaster recovery with MirrorMaker 2


# Table of Contents

1.  [Kafka disaster recovery with MirrorMaker 2](#org45da08a)
2.  [Introduction](#orga6712a0)
3.  [Getting started](#orge1b3667)
4.  [Examples](#org8fa45b9)
    1.  [Single instance Kafka](#org94aa479)
        1.  [Test Case](#org94cbcf3)
    2.  [DR Kafka configuration](#org74bcd2d)
5.  [Route sharding](#org7555179)
    1.  [Labeling kafka objects with custom labels](#orgb1d49cd)
    2.  [OpenShift 4.n](#org49863bf)
    3.  [OpenShift 3.11 (untested)](#orga066192)
6.  [Helpful kafka commands](#org523790e)


# Introduction

This repository includes example file for creating a disaster safe Kafka cluster.
The example configuration is

-   a main-kafka cluster deployed with strimzi (<https://strimzi.io>)
-   a dr-kafka cluster deployed with strimzi (<https://strimzi.io>)
-   a MirrorMaker2 configuration to sync topics from main-kafka to dr-kafka


# Getting started

For installing the strimzi operator follow the [quick-start
installation](https://strimzi.io/docs/operators/master/quickstart.html#proc-install-product-str) documentation.  We used the strimzi operator bundled with
Red Hat's AMQ streams. But the examples below should also work with
the upstream operator. AMQ Streams 1.5 comes with Kafka 2.5 and
Strimzi 0.18.0.


# Examples


<a id="org94aa479"></a>

## Single instance Kafka

In the [examples/single-kafka](examples/single-kafka) folder is a very simple single kafka
configuration for getting started with the strimzi operator. It creates the following objects

-   A main-kafka [namespace](examples/single-kafka/10-main-kafka-namespace.yml) for running the main kafka instance
-   A strimzi kafka [resource](examples/single-kafka/20-main-kafka.yml) for running Kafka
-   A strimzi topic [resource](examples/single-kafka/30-topic.yml) for creating a test topic
-   A test [producer](examples/single-kafka/40-test-producer.yml) that uses the main-kafka Kafka instance
-   A test [consumer](examples/single-kafka/50-test-consumer.yml) that uses the main-kafka Kafka instance


### Test Case

We start a simple hello-world consumer and producer:

    $ oc create -f examples/single-kafka/40-test-producer-main.yml
    $ oc create -f examples/single-kafka/50-test-consumer-main.yml

    $ oc exec main-kafka-kafka-0 -- /opt/kafka/bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 --describe --group hello-world-consumer
    OpenJDK 64-Bit Server VM warning: If the number of processors is expected to increase from one, then you should configure the number of parallel GC threads appropriately using -XX:ParallelGCThreads=N

    GROUP                TOPIC           PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG             CONSUMER-ID                                                          HOST            CLIENT-ID
    hello-world-consumer test-topic      0          130             130             0               consumer-hello-world-consumer-1-2ee0a659-5ca9-44dc-9ef4-ad8c1ff28672 /10.116.0.76    consumer-hello-world-consumer-1
    hello-world-consumer test-topic      1          124             124             0               consumer-hello-world-consumer-1-2ee0a659-5ca9-44dc-9ef4-ad8c1ff28672 /10.116.0.76    consumer-hello-world-consumer-1
    hello-world-consumer test-topic      2          128             128             0               consumer-hello-world-consumer-1-2ee0a659-5ca9-44dc-9ef4-ad8c1ff28672 /10.116.0.76    consumer-hello-world-consumer-1

after stopping the consumer

    oc exec main-kafka-kafka-0 -- /opt/kafka/bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 --describe --group hello-world-consumer
    Defaulting container name to kafka.
    Use 'oc describe pod/main-kafka-kafka-0 -n main-kafka' to see all of the containers in this pod.
    OpenJDK 64-Bit Server VM warning: If the number of processors is expected to increase from one, then you should configure the number of parallel GC threads appropriately using -XX:ParallelGCThreads=N

    GROUP                TOPIC           PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG             CONSUMER-ID                                                          HOST            CLIENT-ID
    hello-world-consumer test-topic      0          180             182             2               consumer-hello-world-consumer-1-50f8337e-439c-46d4-aeba-5bf3523261d0 /10.116.0.77    consumer-hello-world-consumer-1
    hello-world-consumer test-topic      1          177             178             1               consumer-hello-world-consumer-1-50f8337e-439c-46d4-aeba-5bf3523261d0 /10.116.0.77    consumer-hello-world-consumer-1
    hello-world-consumer test-topic      2          180             183             3               consumer-hello-world-consumer-1-50f8337e-439c-46d4-aeba-5bf3523261d0 /10.116.0.77    consumer-hello-world-consumer-1

Because there is no consumer running but we are still producing
messages, current offset is static, log end offset and lag are
increasing.

After starting the test consumer again, lag becomes 0 again and current-offset matches log-end-offset

    $ oc exec main-kafka-kafka-0 -- /opt/kafka/bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 --describe --group hello-world-consumer
    Defaulting container name to kafka.
    Use 'oc describe pod/main-kafka-kafka-0 -n main-kafka' to see all of the containers in this pod.
    OpenJDK 64-Bit Server VM warning: If the number of processors is expected to increase from one, then you should configure the number of parallel GC threads appropriately using -XX:ParallelGCThreads=N

    GROUP                TOPIC           PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG             CONSUMER-ID                                                          HOST            CLIENT-ID
    hello-world-consumer test-topic      0          275             275             0               consumer-hello-world-consumer-1-c6606c35-58f6-48ad-b64c-a3391a1309d1 /10.116.0.78    consumer-hello-world-consumer-1
    hello-world-consumer test-topic      1          265             265             0               consumer-hello-world-consumer-1-c6606c35-58f6-48ad-b64c-a3391a1309d1 /10.116.0.78    consumer-hello-world-consumer-1
    hello-world-consumer test-topic      2          286             286             0               consumer-hello-world-consumer-1-c6606c35-58f6-48ad-b64c-a3391a1309d1 /10.116.0.78    consumer-hello-world-consumer-1


## DR Kafka configuration

For testing the desaster recovery safe Kafka configuration we create a
second namespace dr-kafka and mirror all topics from main-kafka with
MirrorMaker 2 to this instance.

We are using the same resources as in [4.1](#org94aa479). Additionally we create


# Route sharding

We would like to expose routes created by strimzi to a separate ingress controller (aka router).
The are a little bit different between OpenShift 4 and 3.11.


## Labeling kafka objects with custom labels

For selecting routes on different ingress controllers/routes it might be feasible to add custom label to the route objects
strimzi creates for kafka. Basically you have to add a `perPodRoute` and `externalBootstrapRoute` template:

    spec:
      kafka:
        template:
          perPodRoute:
    	metadata:
    	  labels:
    	    type: kafka
          externalBootstrapRoute:
    	metadata:
    	  labels:
    	    type: kafka

This will add a `type: kafka` label to all Kafka and Kafka bootstrap route objects in k8s/ocp.

Find a full example below:

    apiVersion: kafka.strimzi.io/v1beta1
    kind: Kafka
    metadata:
      name: main-kafka
      namespace: main-kafka
    spec:
      kafka:
        template:
          perPodRoute:
    	metadata:
    	  labels:
    	    type: kafka
          externalBootstrapRoute:
    	metadata:
    	  labels:
    	    type: kafka
        replicas: 3
        listeners:
          plain: {}
          tls: {}
          external:
    	type: route
        storage:
          type: ephemeral
      zookeeper:
        replicas: 3
        storage:
          type: ephemeral
      entityOperator:
        topicOperator: {}


## OpenShift 4.n

Create an ingress controller with the following options:

    routeSelector:
      matchLabels:
        strimzi.io/kind: Kafka

A example configuration is [here](ingress/kafka-ingress.yml)

This ingress controller will select routes with a label
`strimzi.io/kind: Kafka` . If you want to publish routes only from a
specific Kafka cluster on this ingress controller you could also use
the label `strimzi.io/cluster: main-kafka`. But remember that you have
to change the route selector for the default ingress controller as
well (see below).

We also do not want to publish kafka routes on the default ingress controller so we change the default configuration
with

    oc edit ingresscontrollers.operator.openshift.io default  -n openshift-ingress-operator

and add the following stanza

    spec:
      routeSelector:
        matchExpressions:
        - key: strimzi.io/kind
          operator: NotIn
          values:
          - Kafka

so the default ingress controller will <span class="underline">not</span> pick up routes with the label `strimzi.io/kind: Kafka`.


## OpenShift 3.11 (untested)

**WARNING**: This is untested because no 3.11 cluster was available.

According to the route sharding docs at
<https://docs.openshift.com/container-platform/3.11/install_config/router/default_haproxy_router.html#using-router-shards>
you have to use environment variables to modify the router
configuration.

One problem is how to exclude routes created for Kafka from the
default router. A possible solution is to expose kafka in a separate
subdomain and use the environment variable `ROUTER_DENIED_DOMAINS` in
the default router so it does **not** pick up routes for kafka.

We would propose the following steps:

1.  Create a new router template with `oc adm router --dry-run -o yaml --service-account=router > kafka-router.yml`
2.  Modify the generated yaml file
    -   change the router name
    -   add the following environment variable `ROUTE_LABELS='strimzi.io/kind=Kafka'` **OR**
    -   use `ROUTER_ALLOWED_DOMAINS`, so that the kafka router only picks up routes for a certain domain
        e.g. `oc set env dc/router ROUTER_ALLOWED_DOMAINS=kafka.ocp3.local`, if more than one domain is used they should be separated by a comma.
3.  Create the router with ~oc create -f kafka-router.yml and test if it picks up the kafka routes
4.  Modify the default router so it does not expose routes for the Kafka domain `oc set env dc/router ROUTER_DENIED_DOMAINS=kafka.ocp3.local`


# Helpful kafka commands

    $ oc exec main-kafka-kafka-0 -- /opt/kafka/bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 --describe --group offset-consumer

    $ oc exec main-kafka-kafka-0 -- /opt/kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 --list

    $ oc exec main-kafka-kafka-0 -- /opt/kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 --topic test-topic --describe
