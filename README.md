# Plan
In this article we will setup a simple kafka cluster with 2 nodes in it.

We will validate that cluster is working by producing and consuming messages. We will setup
`kafka-ui` to see the cluster details. We will also export cluster metrics and watch them in grafana.

We will use `docker compose` for this to make things simpler. 


The docker image for the kafka will be [bitnami/kafka/](https://hub.docker.com/r/bitnami/kafka/).

# Kafka cluster

Let's start with this `docker-compose.yml`:

```yaml
version: '3.8'

x-kafka-common: &kafka-common
  image: 'bitnami/kafka:latest'
  ports:
    - "9092"
  networks:
    - kafka
  healthcheck:
    test: "bash -c 'printf \"\" > /dev/tcp/127.0.0.1/9092; exit $$?;'"
    interval: 5s
    timeout: 10s
    retries: 3
    start_period: 30s
  restart: unless-stopped

x-kafka-env-common: &kafka-env-common
  ALLOW_PLAINTEXT_LISTENER: 'yes'
  KAFKA_CFG_AUTO_CREATE_TOPICS_ENABLE: 'true'
  KAFKA_CFG_CONTROLLER_QUORUM_VOTERS: 0@kafka-0:9093,1@kafka-1:9093
  KAFKA_KRAFT_CLUSTER_ID: abcdefghijklmnopqrstuv
  KAFKA_CFG_PROCESS_ROLES: controller,broker
  KAFKA_CFG_CONTROLLER_LISTENER_NAMES: CONTROLLER
  KAFKA_CFG_LISTENERS: PLAINTEXT://:9092,CONTROLLER://:9093

services:

  kafka-0:
    <<: *kafka-common
    environment:
      <<: *kafka-env-common
      KAFKA_CFG_NODE_ID: 0
    volumes:
      - kafka_0_data:/bitnami/kafka

  kafka-1:
    <<: *kafka-common
    environment:
      <<: *kafka-env-common
      KAFKA_CFG_NODE_ID: 1
    volumes:
      - kafka_1_data:/bitnami/kafka

networks:
  kafka:

volumes:
  kafka_0_data:
    driver: local
  kafka_1_data:
    driver: local
```


Here we define 2 services: `kafka-0` and `kafka-1`. 

We don't need any encryption, so we use `ALLOW_PLAINTEXT_LISTENER=yes`.
Also we don't use zookeeper, instead we will use [KRaft protocol](https://developer.confluent.io/learn/kraft/).

We will add a healthcheck command, which will later help us to run dependent services.

On each node we need to set its id with this variable: `KAFKA_CFG_NODE_ID`.

By setting `KAFKA_CFG_CONTROLLER_QUORUM_VOTERS` environment variable on both services, we make them to work in a cluster.
Also we will use random value for the cluster id with the variable `KAFKA_KRAFT_CLUSTER_ID`: just use any string.

We set also `KAFKA_CFG_PROCESS_ROLES`, `KAFKA_CFG_CONTROLLER_LISTENER_NAMES` and `KAFKA_CFG_LISTENERS` because
they do not have default values.


All other variables will use their default values, you can read more about them here: 
https://github.com/bitnami/containers/blob/main/bitnami/kafka/README.md#configuration.



Let's run our cluster:

```shell
docker compose up
```

# Validate cluster

Now we need to create a new topic and put some messages into it.
Kafka comes with a set of scripts for managing it, let's use them:

```shell
docker compose exec kafka-0 /opt/bitnami/kafka/bin/kafka-topics.sh --create --bootstrap-server kafka-0:9092,kafka-1:9092 --replication-factor 1 --partitions 1 --topic test
```
We tell the `kafka-topics.sh` that it should use our kafka instances as a bootstrap servers.
As a result you should see 
```
Created topic test.
```

Let's list all topics:
```
$ docker compose exec kafka-0 /opt/bitnami/kafka/bin/kafka-topics.sh --list --bootstrap-server kafka-0:9092,kafka-1:9092
test
$ docker compose exec kafka-0 /opt/bitnami/kafka/bin/kafka-topics.sh --describe --topic test --bootstrap-server kafka-0:9092,kafka-1:9092
Topic: test	TopicId: 7FEciEQRRjqJ2zntIONOcQ	PartitionCount: 1	ReplicationFactor: 1	Configs: segment.bytes=1073741824
	Topic: test	Partition: 0	Leader: 0	Replicas: 0	Isr: 0
```


Let's write some messages into this topic. Open new terminal (this will be producer terminal) and type:
```
docker compose exec kafka-0 /opt/bitnami/kafka/bin/kafka-console-producer.sh --bootstrap-server kafka-0:9092,kafka-1:9092 --producer.config /opt/bitnami/kafka/config/producer.properties --topic test
>message0
```

Open another terminal (consumer terminal) and read the message with a consumer:
```shell
docker compose exec kafka-0  /opt/bitnami/kafka/bin/kafka-console-consumer.sh --bootstrap-server kafka-0:9092,kafka-1:9092 --consumer.config /opt/bitnami/kafka/config/consumer.properties --topic test --from-beginning
```
You should see `message0`. Switch back to the producer terminal and type another message. You will see it immediately 
in consumer terminal.


Press Control+C to close producer and consumer.

At this point we can start a new cluster, create topic, produce the messages and consume them.


# Kafka UI
Next we will add kafka-ui. This application will give us a nice UI to view our cluster.

Add the following to the `docker-compose.yml`:
```yaml
  kafka-ui:
    container_name: kafka-ui
    image: provectuslabs/kafka-ui:latest
    volumes:
      - ./kafka-ui/config.yml:/etc/kafkaui/dynamic_config.yaml
    environment:
      DYNAMIC_CONFIG_ENABLED: 'true'
    depends_on:
      - kafka-0
      - kafka-1
    networks:
      - kafka
    ports:
      - '8080:8080'
    healthcheck:
      test: wget --no-verbose --tries=1 --spider localhost:8080 || exit 1
      interval: 5s
      timeout: 10s
      retries: 3
      start_period: 30s
 ```

You need to create a configuration file for the kafka ui:
```shell
mkdir kafka-ui
touch kafka-ui/config.yml
```
Kafka ui can be configured with environment variables, but I prefer yaml, because variables will become very messy
once you will have more then 1 cluster.

Put the following into `kafka-ui/config.yml`:
```yaml
auth:
  type: LOGIN_FORM

spring:
  security:
    user:
      name: admin
      password: admin

kafka:
  clusters:
    - bootstrapServers: kafka-0:9092,kafka-1:9092
      name: kafka
```

Stop current docker compose process (control+c) and restart it::
```shell
docker compose up
```


Open kafka ui in the browser http://localhost:8080. Use `admin:admin` as a credentials (see `kafka-ui/config.yml`).
You will see our new cluster.

Visit http://localhost:8080/ui/clusters/kafka/all-topics/test/messages?keySerde=String&valueSerde=String&limit=100 and 
make sure you see all messages, that we produced earlier.



# Metrics

## Kafka exporter

Now we need to expose kafka metrics to the prometheus and be able to see them in grafana.
[Kafka exporter](https://github.com/danielqsj/kafka_exporter) will be used to actually export the metrics. 
Let's add new services to the `docker-compose.yml`:
```yaml

  prometheus:
    image: prom/prometheus
    container_name: prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
    ports:
      - 9090:9090
    volumes:
      - ./prometheus:/etc/prometheus
      - prom_data:/prometheus
    networks:
      - kafka  

  kafka-exporter:
    image: docker.io/bitnami/kafka-exporter:latest
    depends_on:
      kafka-0:
        condition: service_healthy
      kafka-1:
        condition: service_healthy
    networks:
      - kafka
    command: --kafka.server=kafka-0:9092 --kafka.server=kafka-1:9092
      
        
  grafana:
    image: grafana/grafana
    container_name: grafana
    ports:
      - 3000:3000
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=grafana
    volumes:
      - ./grafana/provisioning:/etc/grafana/provisioning
      - ./grafana/dashboards:/var/lib/grafana/dashboards
    networks:
      - kafka
```
and this in `volumes` section:
```yaml
  prom_data:
    driver: local
```

Kafka exporter depends on kafka instances, it will produce an error during start, if there are no running kafka brokers. 
This is why we added the healthcheck for kafka.






## JMX exporter

There is also one more way to collect metrics (they can be used together):
[jmx exporter](https://github.com/prometheus/jmx_exporter). We will use it as a java agent.

Download the jar and the config:
```shell
mkdir -p jmx-exporter
curl https://repo1.maven.org/maven2/io/prometheus/jmx/jmx_prometheus_javaagent/0.19.0/jmx_prometheus_javaagent-0.19.0.jar\
  -o jmx-exporter/jmx_prometheus_javaagent-0.19.0.jar
curl  https://raw.githubusercontent.com/prometheus/jmx_exporter/main/example_configs/kafka-2_0_0.yml \
  -o jmx-exporter/kafka-2_0_0.yml
```

Mount it to kafka images:
```
  kafka-0:
    <<: *kafka-common
    environment:
      <<: *kafka-env-common
      KAFKA_CFG_NODE_ID: 0
    volumes:
      - kafka_0_data:/bitnami/kafka
      - ./jmx-exporter:/opt/jmx-exporter

  kafka-1:
    <<: *kafka-common
    environment:
      <<: *kafka-env-common
      KAFKA_CFG_NODE_ID: 1
    volumes:
      - kafka_1_data:/bitnami/kafka
      - ./jmx-exporter:/opt/jmx-exporter
```


Update the `x-kafka-env-common` block in `docker-compose.yml` like this:
```yaml
x-kafka-env-common: &kafka-env-common
  ALLOW_PLAINTEXT_LISTENER: 'yes'
  KAFKA_CFG_AUTO_CREATE_TOPICS_ENABLE: 'true'
  KAFKA_CFG_CONTROLLER_QUORUM_VOTERS: 0@kafka-0:9093,1@kafka-1:9093
  KAFKA_KRAFT_CLUSTER_ID: abcdefghijklmnopqrstuv
  EXTRA_ARGS: "-Xmx=256m -javaagent:/opt/jmx-exporter/jmx_prometheus_javaagent-0.19.0.jar=9404:/opt/jmx-exporter/kafka-2_0_0.yml"
```

The exporter will be available at port 9404:
```shell
docker compose exec kafka-0 curl localhost:9404
```

We also want Kafka UI to use the metrics. Add this to `kafka-ui/config.yml`:
```yaml
kafka:
  clusters:
    - bootstrapServers: kafka-0:9092,kafka-1:9092
      name: kafka
      metrics:
        type: JMX
        port: 9404
```





## Prometheus
Create configuration file for a prometheus:
```
mkdir prometheus
touch prometheus/prometheus.yml
```
Put the following content:
```yaml
global:
  scrape_interval: 15s
  scrape_timeout: 10s
  evaluation_interval: 15s
scrape_configs:
- job_name: kafka-exporter
  honor_timestamps: true
  scrape_interval: 15s
  scrape_timeout: 10s
  metrics_path: /metrics
  scheme: http
  static_configs:
  - targets:
    - kafka-exporter:9308
- job_name: jmx-exporter
  honor_timestamps: true
  scrape_interval: 15s
  scrape_timeout: 10s
  metrics_path: /metrics
  scheme: http
  static_configs:
  - targets:
    - kafka-0:9404
    - kafka-1:9404
```
Here we ask prometheus to get metrics from kafka exporter from `/metrics` endpoint on port `9093` and from
jmx-exporter on port `9404`.




## Grafana

Finally, create the configuration for grafana:
```shell
mkdir -p grafana/provisioning/datasources
mkdir -p grafana/provisioning/dashboards
touch grafana/provisioning/datasources/datasource.yml
touch grafana/provisioning/dashboards/dashboard.yml
curl https://raw.githubusercontent.com/strimzi/strimzi-kafka-operator/main/examples/metrics/grafana-dashboards/strimzi-kafka-exporter.json -o grafana/dashboards/strimzi-kafka-exporter.json
curl https://raw.githubusercontent.com/strimzi/strimzi-kafka-operator/main/examples/metrics/grafana-dashboards/strimzi-kafka.json -o grafana/dashboards/strimzi-kafka.json
```

We tell the grafana to provision prometheus datasource. Also we download the dashboard for the kafka exporter and
jmx exporter  and tell grafana to use it.

Put the following into `grafana/provisioning/datasources/datasource.yml`:

```yaml
apiVersion: 1

deleteDatasources:
  - name: Prometheus
    orgId: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    orgId: 1
    url: http://prometheus:9090
    password:
    user:
    database:
    basicAuth: false
    basicAuthUser:
    basicAuthPassword:
    withCredentials:
    isDefault: true
    version: 1
    editable: false
```

and the following into `grafana/provisioning/dashboards/dashboard.yml`:
```yaml
apiVersion: 1

providers:
  - name: 'dashboards-from-file'
    orgId: 1
    folder: ''
    folderUid: ''
    type: file
    disableDeletion: false
    updateIntervalSeconds: 10
    allowUiUpdates: false
    options:
      path: /var/lib/grafana/dashboards
      foldersFromFilesStructure: true
```


Now restart docker compose and visit http://localhost:3000/dashboards. Use `admin`:`grafana` as credentials.
Open dashboard with name `Kafka Exporter Overview`, you will see some details about the cluster.



# Conclusion
Now we can run grafana cluster, we can validate it by producing and consuming messages from CLI. 
Also we can view the details of the cluster with UI and watch the cluster metrics in grafana.


# Links
- APACHE KAFKA: https://kafka.apache.org/
- Bitnami kafka image: https://hub.docker.com/r/bitnami/kafka/
- Grafana: https://grafana.com/
- Kafka Exporter: https://github.com/danielqsj/kafka_exporter
- Kafka UI: https://github.com/provectus/kafka-ui
- Prometheus: https://prometheus.io/

