# Setup

*NOTE! Kubernetes v 1.17 used. It is the only prerequisite.*

There are 3 YAML manifests. Copy them to any new directory. Use **kubectl** to deploy to any available k8s cluster.

>kubectl apply -f .

This will "one-click" deploy the whole Kafka cluster.

>kubectl delete -f .

Removes all created resources.

# Manifest description

**zk_set.yaml** deploys ZooKeper cluster *(**ZK** from now on)*, which is so far inevitable for Kafka.

**kafka_service.yaml** and **kafka_deploy.yaml** set up Kafka cluster.

# zk_set.yaml

A manifest for building 3-node ZK cluster. 
It has four parts overall:
- zk-headless, headless service for controlling ZK internal comms, like leader election.
- zk-cluster, load-balanced service for Kafka -> ZK traffic
- Pod disruption budget, since ZK requires majority of voices to elect leader.
- zk Deployment, actual 3 replicas of ZK server

Deployment has optional anti-affinity used, just to spread ZKs around available k8s nodes. This will increase its HA.
I decided to unite all zk components into a single manifest to avoid cluttering the repo.

# kafka_service.yaml

Deploys kafka-service as a Service to handle kafka broker traffic. Since kafka brokers by default use only port 9092, this file is quite simple.

# kafka_deployment.yaml

Builds a 3-node Kafka cluster. For extra availability it uses *readinessProbe* to ensure port 9092 is opened. *livenessProbe* can be also introduced in real setup to ensure traffic flow. Since it is a test, I used a simpler setup.

Kafka cluster has a convoluted dependency on ZK, thus it was a bit tedious to configure networking. The image is used accepts env.variables to manage Kafka configuration.
The most important thing is to configure LISTENERS and connection to ZK.

For some reason Kafka should be able to reach its own listener, and also remote listeners.
        - name: KAFKA_LISTENERS
          value: "PLAINTEXT://0.0.0.0:9092"
is an internal listener on localhost.
        - name: KAFKA_ADVERTISED_LISTENERS
          value: "PLAINTEXT://$(MY_POD_IP):9092"
is an "outer" listener which is used by ZK service and to accept/generate messages.
I had to inject MY_POD_IP var. here taken from pod auto-generated IP.

In order to connect to ZK service, kafka brokers will use internally resolved 
*zk-cluster.default.svc.cluster.local:2181*
It then does not matter which ZK service dies, traffic flow will be balanced.

# My tests

I deployed it all over AWS EKS. To ensure that cluster was actually working, built-inconsole Kafka tools were used. (Actually it could be encapsulated into livenessProbes)

Something like

>kubectl get pods
>kubectl exec -it kafka-whatever -- bash
>cd /opt/kafka/bin
>kafka-topics.sh --create --zookeeper zk-cluster:2181 --replication-factor 1 --partitions 1 --topic MyTest
>kafka-console-producer.sh --broker-list localhost:9092 --topic MyTest

Generated some messages, and then from any other pod (same path for sh scripts)

>kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic MyTest --from-beginning

 received same messages, generated under first pod.
 
 # My Issue and Security considerations
 
 I will start with **issues**. There we quite a lot.
 
Kafka does not have official docker image, so realistically we probably need to generate our own dockerfile and image. I just searched for the most popular and adequately documented Kafka and ZK images.

Kafka has poor documentation. E.g. ADVERTISED_HOST_NAME config variable does not work and had to be replaced with ADVERTISED_LISTENERS, because otherwise leader cannot be elected. It mention it here, because this actually contradicts official docs. That's just one of many config related issues. Seems like Kafka default docs are not suited for Kubernetes usage. A lot of things must be tried out manually.

ZK service is very "monolithic" and scales badly. Since ZK requires to explicitly specify IDs of ZK servers you cannot just use **kubectl scale** to increase N of zk replicas.
In my example brokers will dynamically locate live ZKs, but to increase from 3 to 4 zk replicas, one needs to modify config files. It can be done through COMMAND: block in k8s manifest, and a sh script, but not out of the box.

Kafka uses external and internal listeners for Docker network and outer world. It can be a problem if using external "producers" or "consumers".

Now, about **Security**.

1)My test setup uses ephemeral stateless Deployments, they lose data after reboot.  For production a *StatefulSet*   with mounted volumes is better , since Kafka is very stateful itself.

2)My setup uses PLAINTEXT listeners. Production should use TLS listeners. This would imply additional problem of introducing certificates inside pods.

3)wurstmeister/kafka images, which I used expose too much permissions to kafka brokers. E.g. from inside the pod, by default: **whoami** from pod will show **root** access.

4)Multiple partition and broker setups do not guarantee FIFO message delivery. Can be a problem for certain setups.

5)Data is written into 1 replica by default coz of **min.insync.replicas** parameter. it should be increased in production to allow parallel write of data to all kafka replicas.

# Performance considerations

Kafka tends to consume all available ram resources because it uses a paging cache, thus must be strictly limited in a manifest and probably use anti-affinity to evenly spead across a cluster.

I played with available AWS instances for k8s nodes. Seems like HDD disks are not enough for adequate latency, so SSDs or object-storage as S3 should be used.

Generic test like
>kafka-producer-perf-test.sh --topic MyTest --record-size 100--num-records 10000000  --throughput -1 --producer.config ../config/producer.properties

on T3.medium, generated around 46000 records per second; ~100 bytes per record. **43MB/s throughput and 98ms** latency.



# Alerting and Monitoring

It all depends on available tools at our disposal. If allowed I would choose easy to use tool like Prometheus. 
If impossible, we can use default Kafka ability to dump metrics through **JMX**.
It could be utilized, by specifying **JMX_PORT** env.variable in *kafka_deployment* manifest.

The data can be then handled by another pod, containing something like** jmxtrans** docker image. This would require a standalone data handler in our cluster. jmxtrans can flux the data into AWS CloudFormation for example or any other graph building tool.

We can monitor the following on broker nodes:

**Page cache read ratio in %**. Kafka actively uses caching, so if brokers can't keep up with reading from cache, this ratio will go below 100. Abnormally low values like 40% will  sign that more brokers must be deployed.

**Disk usage** obvously as kafka should store data on volumes to avoid data loss. This metric must have an alert on it.

**Garbage collection** (hourly or per min); Kafka uses JVM, so inactivity periods in gc will cause problems. This must be alerted.

**Requests per second** as far as I remember can monitor consumer/producer requests number, to get an average idea of data influx.

**Total time** to server a request in ms.

As for producers, it makes sense to monitor **Broker response rate**, just to make sure brokers keep up with ack-ing messages generated.

Consumers may expose **Messages consumed** metric, since bytes consumed will hardly be adequate, as we don't know the expected size of a single message.

There can be some other simpler alerts like "no leader" situation, or ZK outage.








