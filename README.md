
# Table of Contents

1.  [Kafka disaster recovery with MirrorMaker 2](#orgee9a91c)


<a id="orgee9a91c"></a>

# Kafka disaster recovery with MirrorMaker 2

This repository includes example file for creating a disaster safe Kafka cluster.
The example configuration is

-   a main-kafka cluster deployed with strimzi (<https://strimzi.io>)
-   a dr-kafka cluster deployed with strimzi (<https://strimzi.io>)
-   a MirrorMaker2 configuration to sync topics from main-kafka to dr-kafka
