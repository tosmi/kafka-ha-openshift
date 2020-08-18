

# Kafka disaster recovery with MirrorMaker 2


# Table of Contents

1.  [Kafka disaster recovery with MirrorMaker 2](#org79b5c8a)
2.  [Introduction](#org19364cd)
3.  [Route sharding](#org3a5272f)
    1.  [OpenShift 4.n](#org0ae2a2d)
    2.  [OpenShift 3.11 (untested)](#org11834fe)


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

A example configuration is here <./ingress/kafka-ingress.yml>

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
    -   add the following environment variable `ROUTE_LABELS='strimzi.io/kind=Kafka'`
3.  Create the router with ~oc create -f kafka-router.yml and test if it picks up the kafka routes
4.  Modify the default router so it does not expose routes for the Kafka domain `oc set env dc/router ROUTER_DENIED_DOMAINS=kafka.ocp3.local`
