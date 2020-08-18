

# Kafka disaster recovery with MirrorMaker 2


# Table of Contents

1.  [Kafka disaster recovery with MirrorMaker 2](#org57521c8)
2.  [Introduction](#org1d189e7)
3.  [Route sharding](#org24261b2)
    1.  [OpenShift 4.n](#orgd6d890e)
    2.  [OpenShift 3.11 (untested)](#org5d78aba)


# Introduction

This repository includes example file for creating a disaster safe Kafka cluster.
The example configuration is

-   a main-kafka cluster deployed with strimzi (<https://strimzi.io>)
-   a dr-kafka cluster deployed with strimzi (<https://strimzi.io>)
-   a MirrorMaker2 configuration to sync topics from main-kafka to dr-kafka


# Route sharding

We would like to expose routes created by strimzi to a separate ingress controller (aka router).
The are a little bit different between OpenShift 4 and 3.11.


## OpenShift 4.n

Create an ingress controller with the following options:

    routeSelector:
      matchLabels:
        strimzi.io/kind: Kafka

This ingress controller will select routes with a label `strimzi.io/kind: Kafka` .

We also do not want to publish kafka routes on the default ingress controller so we change the default configuration
with

    oc edit ingresscontrollers.operator.openshift.io default  -n openshift-ingress-operator


## OpenShift 3.11 (untested)
