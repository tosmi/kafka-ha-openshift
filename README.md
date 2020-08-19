

# Kafka disaster recovery with MirrorMaker 2


# Table of Contents

1.  [Kafka disaster recovery with MirrorMaker 2](#org227197e)
2.  [Introduction](#org2730aeb)
3.  [Getting started](#org2a890dd)
4.  [Examples](#org05e6ede)
    1.  [Single instance Kafka](#org6df3d54)
    2.  [DR Kafka configuration](#org095b4e6)
5.  [Route sharding](#org327ab8c)
    1.  [OpenShift 4.n](#orgfbe20d7)
    2.  [OpenShift 3.11 (untested)](#orgd0babff)


# Introduction

This repository includes example file for creating a disaster safe Kafka cluster.
The example configuration is

-   a main-kafka cluster deployed with strimzi (<https://strimzi.io>)
-   a dr-kafka cluster deployed with strimzi (<https://strimzi.io>)
-   a MirrorMaker2 configuration to sync topics from main-kafka to dr-kafka


# Getting started

For installating the strimzi operator follow the [quick-start
installation](https://strimzi.io/docs/operators/master/quickstart.html#proc-install-product-str) documention.  We used the strimzi operator bundled with
Red Hat's AMQ streams. But the examples below should also work with
the upstream operator. AMQ Streams 1.5 comes with Kafka 2.5 and
Strimzi 0.18.0.


# Examples


<a id="org6df3d54"></a>

## Single instance Kafka

In the [examples/single-kafka](examples/single-kafka) folder is a very simple single kafka
configuration for getting started with the strimzi operator. It creates the following objects

-   A main-kafka [namespace](examples/single-kafka/10-main-kafka-namespace.yml) for running the main kafka instance
-   A strimzi kafka [resource](examples/single-kafka/20-main-kafka.yml) for running Kafka
-   A strimzi topic [resource](examples/single-kafka/30-topic.yml) for creating a test topic
-   A test [producer](examples/single-kafka/40-test-producer.yml) that uses the main-kafka Kafka instance
-   A test [consumer](examples/single-kafka/50-test-consumer.yml) that uses the main-kafka Kafka instance


## DR Kafka configuration

For testing the desaster recovery safe Kafka configuration we create a
second namespace dr-kafka and mirror all topics from main-kafka with
MirrorMaker 2 to this instance.

We are using the same resouces as in [4.1](#org6df3d54). Additionally we create


# Route sharding

We would like to expose routes created by strimzi to a separate ingress controller (aka router).
The are a little bit different between OpenShift 4 and 3.11.


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
