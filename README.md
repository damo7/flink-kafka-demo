# Evoura demo labs - configurable use cases for understanding data streaming and different systems

'''txt
This is a fork of the original work done by gary stafford: streaming data analytics demonstration, written in Java and
using the [Streaming Synthetic Sales Data Generator](https://github.com/garystafford/streaming-sales-generator). 

What we have done is update the various frameworks versions to the latest, update Java code to compatible with Java 11 (including gradel build scripts), and have the jobs uploaded through Flink's API. For now, this lab is only for local development and deployment, but we will start to configure scripting to allow it to be deployed to seperate cloud providers. 

Moving forward we will expand the use cases to include CDC, removing Kafka and replacing it with Redpanda, introduce Ververica and their hosted platform instead of Apache Flink and include seperate storage using Apache Iceberg (and the newly released catalog from snowflake). 

We will also start to configure different objectives to demonstrate the different end use cases that can be used with these tools. From monitoring telecoms networks, cyber security, high frequency trading and embedding AI models into Flink for RAG/LLM use cases. 

Evoura will continue to maintain this working lab with contributions being opened to whoever would like to committ to new use cases. If have any issues, please file a ticket and we will do our best to respond.
'''txt

# Jobs and set-up

* `org.example.RunningTotals`: Consumes a stream of sales transaction messages and publishes a stream of running totals of product transactions, quantities, and sales to a Kafka topic
* `org.example.JoinStreams`: Consumes a stream of product and purchase messages from two different Kafka topics, joins both into an enriched purchase object, and writes results to a third Kafka topic

* Uber JAR built with Gradle using Amazon Corretto (OpenJDK) version 11 (openjdk version "11.0.16.1" 2022-08-12 LTS)


## Input Message Stream

Sample sales purchase messages:

```txt
{"transaction_time": "2022-09-13 12:58:33.442579", "transaction_id": "1102438156100296597", "product_id": "SC02", "price": 5.99, "quantity": 2, "is_member": false, "member_discount": 0.0, "add_supplements": false, "supplement_price": 0.0, "total_purchase": 11.98}
{"transaction_time": "2022-09-13 12:58:33.442579", "transaction_id": "1102438156100296597", "product_id": "SF05", "price": 5.99, "quantity": 2, "is_member": false, "member_discount": 0.0, "add_supplements": false, "supplement_price": 0.0, "total_purchase": 11.98}
{"transaction_time": "2022-09-13 12:58:36.915834", "transaction_id": "2883033696701592101", "product_id": "SC04", "price": 5.99, "quantity": 1, "is_member": false, "member_discount": 0.0, "add_supplements": false, "supplement_price": 0.0, "total_purchase": 5.99}
{"transaction_time": "2022-09-13 12:58:36.915834", "transaction_id": "2883033696701592101", "product_id": "CS01", "price": 4.99, "quantity": 1, "is_member": false, "member_discount": 0.0, "add_supplements": false, "supplement_price": 0.0, "total_purchase": 4.99}
{"transaction_time": "2022-09-13 12:58:36.915834", "transaction_id": "2883033696701592101", "product_id": "CS09", "price": 4.99, "quantity": 1, "is_member": false, "member_discount": 0.0, "add_supplements": false, "supplement_price": 0.0, "total_purchase": 4.99}
{"transaction_time": "2022-09-13 12:58:40.144183", "transaction_id": "3818951251710853699", "product_id": "IS03", "price": 5.49, "quantity": 1, "is_member": false, "member_discount": 0.0, "add_supplements": false, "supplement_price": 0.0, "total_purchase": 5.49}
{"transaction_time": "2022-09-13 12:58:43.378102", "transaction_id": "8423291927238665701", "product_id": "SC04", "price": 5.99, "quantity": 1, "is_member": false, "member_discount": 0.0, "add_supplements": false, "supplement_price": 0.0, "total_purchase": 5.99}
{"transaction_time": "2022-09-13 12:58:49.438818", "transaction_id": "7801537351341225872", "product_id": "SF06", "price": 5.99, "quantity": 2, "is_member": false, "member_discount": 0.0, "add_supplements": false, "supplement_price": 0.0, "total_purchase": 11.98}
{"transaction_time": "2022-09-13 12:58:50.696241", "transaction_id": "5463447061814429579", "product_id": "SF07", "price": 5.99, "quantity": 1, "is_member": true, "member_discount": 0.1, "add_supplements": true, "supplement_price": 1.99, "total_purchase": 7.18}
{"transaction_time": "2022-09-13 12:58:51.933601", "transaction_id": "1037974254425034387", "product_id": "IS02", "price": 5.49, "quantity": 2, "is_member": true, "member_discount": 0.1, "add_supplements": false, "supplement_price": 0.0, "total_purchase": 9.88}
```

## Output Message Stream

Sample running totals messages:

```txt
{"event_time":"2022-09-10T02:43:58.320625Z","product_id":"SC01","transactions":35,"quantities":47,"sales":316.90}
{"event_time":"2022-09-10T02:43:59.528838Z","product_id":"CS10","transactions":23,"quantities":30,"sales":168.78}
{"event_time":"2022-09-10T02:44:02.755865Z","product_id":"CS08","transactions":64,"quantities":70,"sales":358.79}
{"event_time":"2022-09-10T02:44:03.962799Z","product_id":"SC04","transactions":52,"quantities":65,"sales":432.06}
{"event_time":"2022-09-10T02:44:05.150230Z","product_id":"CS04","transactions":22,"quantities":27,"sales":135.70}
{"event_time":"2022-09-10T02:44:06.458681Z","product_id":"SF05","transactions":36,"quantities":44,"sales":301.12}
{"event_time":"2022-09-10T02:44:08.668936Z","product_id":"CS10","transactions":24,"quantities":31,"sales":173.77}
{"event_time":"2022-09-10T02:44:11.889087Z","product_id":"SF03","transactions":22,"quantities":26,"sales":175.42}
{"event_time":"2022-09-10T02:44:15.003075Z","product_id":"CS08","transactions":65,"quantities":71,"sales":365.07}
{"event_time":"2022-09-10T02:44:16.209181Z","product_id":"SF07","transactions":51,"quantities":62,"sales":426.45}
```

Sample enriched purchases messages:

```txt
{"transaction_time":"2022-09-13 12:50:55.644564","transaction_id":"1142152017802750696","product_id":"CS06","product_category":"Classic Smoothies","product_name":"Blimey Limey","product_size":"24 oz.","product_cogs":1.50,"product_price":4.99,"contains_fruit":true,"contains_veggies":false,"contains_nuts":false,"contains_caffeine":false,"purchase_price":4.99,"purchase_quantity":1,"is_member":false,"member_discount":0.00,"add_supplements":false,"supplement_price":0.00,"total_purchase":4.99}
{"transaction_time":"2022-09-13 12:50:57.889523","transaction_id":"5186286335839079216","product_id":"SF06","product_category":"Superfoods Smoothies","product_name":"Get Up and Goji","product_size":"24 oz.","product_cogs":2.10,"product_price":5.99,"contains_fruit":true,"contains_veggies":true,"contains_nuts":false,"contains_caffeine":false,"purchase_price":5.99,"purchase_quantity":1,"is_member":true,"member_discount":0.10,"add_supplements":false,"supplement_price":0.00,"total_purchase":5.39}
{"transaction_time":"2022-09-13 12:50:59.153677","transaction_id":"3488041191398632204","product_id":"SF04","product_category":"Superfoods Smoothies","product_name":"Pomegranate Plunge","product_size":"24 oz.","product_cogs":2.10,"product_price":5.99,"contains_fruit":true,"contains_veggies":false,"contains_nuts":false,"contains_caffeine":false,"purchase_price":5.99,"purchase_quantity":1,"is_member":false,"member_discount":0.00,"add_supplements":false,"supplement_price":0.00,"total_purchase":5.99}
{"transaction_time":"2022-09-13 12:51:01.387870","transaction_id":"77267280375192969","product_id":"CS03","product_category":"Classic Smoothies","product_name":"Paradise Point","product_size":"24 oz.","product_cogs":1.50,"product_price":4.99,"contains_fruit":true,"contains_veggies":false,"contains_nuts":false,"contains_caffeine":false,"purchase_price":4.99,"purchase_quantity":2,"is_member":false,"member_discount":0.00,"add_supplements":false,"supplement_price":0.00,"total_purchase":9.98}
{"transaction_time":"2022-09-13 12:51:01.387870","transaction_id":"77267280375192969","product_id":"SC03","product_category":"Supercharged Smoothies","product_name":"Health Nut","product_size":"24 oz.","product_cogs":2.70,"product_price":5.99,"contains_fruit":false,"contains_veggies":false,"contains_nuts":true,"contains_caffeine":false,"purchase_price":5.99,"purchase_quantity":1,"is_member":false,"member_discount":0.00,"add_supplements":false,"supplement_price":0.00,"total_purchase":5.99}
{"transaction_time":"2022-09-13 12:51:01.387870","transaction_id":"77267280375192969","product_id":"IS04","product_category":"Indulgent Smoothies","product_name":"Mocha Madness","product_size":"24 oz.","product_cogs":2.20,"product_price":5.49,"contains_fruit":false,"contains_veggies":false,"contains_nuts":true,"contains_caffeine":true,"purchase_price":5.49,"purchase_quantity":1,"is_member":false,"member_discount":0.00,"add_supplements":false,"supplement_price":0.00,"total_purchase":5.49}
{"transaction_time":"2022-09-13 12:51:03.072536","transaction_id":"5904837887652070959","product_id":"IS01","product_category":"Indulgent Smoothies","product_name":"Bahama Mama","product_size":"24 oz.","product_cogs":2.20,"product_price":5.49,"contains_fruit":true,"contains_veggies":false,"contains_nuts":false,"contains_caffeine":false,"purchase_price":5.49,"purchase_quantity":1,"is_member":false,"member_discount":0.00,"add_supplements":false,"supplement_price":0.00,"total_purchase":5.49}
{"transaction_time":"2022-09-13 12:51:06.370331","transaction_id":"5479660984247512791","product_id":"SF06","product_category":"Superfoods Smoothies","product_name":"Get Up and Goji","product_size":"24 oz.","product_cogs":2.10,"product_price":5.99,"contains_fruit":true,"contains_veggies":true,"contains_nuts":false,"contains_caffeine":false,"purchase_price":5.99,"purchase_quantity":1,"is_member":false,"member_discount":0.00,"add_supplements":true,"supplement_price":1.99,"total_purchase":7.98}
{"transaction_time":"2022-09-13 12:51:09.807906","transaction_id":"8552032150877524327","product_id":"SF06","product_category":"Superfoods Smoothies","product_name":"Get Up and Goji","product_size":"24 oz.","product_cogs":2.10,"product_price":5.99,"contains_fruit":true,"contains_veggies":true,"contains_nuts":false,"contains_caffeine":false,"purchase_price":5.99,"purchase_quantity":3,"is_member":true,"member_discount":0.10,"add_supplements":false,"supplement_price":0.00,"total_purchase":16.17}
{"transaction_time":"2022-09-13 12:51:09.807906","transaction_id":"8552032150877524327","product_id":"SF05","product_category":"Superfoods Smoothies","product_name":"Caribbean C-Burst","product_size":"24 oz.","product_cogs":2.10,"product_price":5.99,"contains_fruit":true,"contains_veggies":false,"contains_nuts":false,"contains_caffeine":false,"purchase_price":5.99,"purchase_quantity":1,"is_member":true,"member_discount":0.10,"add_supplements":false,"supplement_price":0.00,"total_purchase":5.39}
```

## Compile and Run Flink Job

```shell
# optional - set java version
JAVA_HOME=~/Library/Java/JavaVirtualMachines/corretto-11.0.16.1/Contents/Home

# build uber jar using Gradle
./gradlew clean shadowJar

# Upload JAR and submit the job using flink API's:

curl -X POST -H "Expect:" -F "jarfile=@/path/to/your/flink-kafka-demo-1.2.0.jar" http://localhost:8081/jars/upload
curl -X POST -H "Content-Type: application/json" -d @job-submission.json http://localhost:8081/jars/<jar_id>/run

```

## Kafka

Helpful Kafka commands.

```shell
docker exec -it $(docker container ls --filter  name=kafka-flink_kafka --format "{{.ID}}") bash

export BOOTSTRAP_SERVERS="localhost:9092"

export INPUT_TOPIC="demo.purchases"
export OUTPUT_TOPIC="demo.running.totals"

export PRODUCT_TOPIC="demo.products";
export PURCHASE_TOPIC="demo.purchases";
export PURCHASES_ENRICHED_TOPIC="demo.purchases.enriched";

# list all topics
kafka-topics.sh --list \
    --bootstrap-server $BOOTSTRAP_SERVERS

# describe topic
kafka-topics.sh --describe \
    --topic $OUTPUT_TOPIC \
    --bootstrap-server $BOOTSTRAP_SERVERS

# delete topic
kafka-topics.sh --delete \
    --topic $INPUT_TOPIC \
    --bootstrap-server $BOOTSTRAP_SERVERS

kafka-topics.sh --delete \
    --topic $OUTPUT_TOPIC \
    --bootstrap-server $BOOTSTRAP_SERVERS

# optional: create new topic (or they will be automatically created
kafka-topics.sh --create \
    --topic $INPUT_TOPIC \
    --partitions 1 --replication-factor 1 \
    --config cleanup.policy=compact \
    --bootstrap-server $BOOTSTRAP_SERVERS

kafka-topics.sh --create \
    --topic $OUTPUT_TOPIC \
    --partitions 1 --replication-factor 1 \
    --config cleanup.policy=compact \
    --bootstrap-server $BOOTSTRAP_SERVERS

# view messages
kafka-console-consumer.sh \
    --topic $INPUT_TOPIC --from-beginning \
    --bootstrap-server $BOOTSTRAP_SERVERS

kafka-console-consumer.sh \
    --topic $OUTPUT_TOPIC --from-beginning \
    --bootstrap-server $BOOTSTRAP_SERVERS
```

## Docker Stack

Demonstration uses
Kafka/Flink [Docker Swarm Stack](https://github.com/garystafford/streaming-sales-generator/blob/main/docker-compose.yml)
from 'Sales Data Generator' project.

See [bitnami/kafka](https://hub.docker.com/r/bitnami/kafka) on Docker Hub for more information about running Kafka
locally using Docker.

```shell
# optional: delete previous stack
docker stack rm kafka-flink

# deploy kafka stack
docker swarm init
docker stack deploy kafka-flink --compose-file docker-compose.yml

# optional: to exec into Kafka container
docker exec -it $(docker container ls --filter  name=kafka-flink_kafka --format "{{.ID}}") bash
```

### Containers

Example containers:

```text
CONTAINER ID   NAME                                                           CPU %     MEM USAGE / LIMIT     MEM %     NET I/O           BLOCK I/O         PIDS
9f54226b47c6   streaming-stack_kafka.1.lg9ucoxzsjh21cqg7ek2had2f              0.57%     998.3MiB / 11.67GiB   8.35%     25.5MB / 164MB    1.91MB / 213MB    90
b15dbca6d518   streaming-stack_kafka-ui.1.z4oe4xl16u69zi5dcf44vpmyf           0.10%     424.2MiB / 11.67GiB   3.55%     149MB / 15.3MB    8.66MB / 40.6MB   58
ead0b54b1819   streaming-stack_zookeeper.1.sdwpo5f8mebm06ztjg6oqxb4u          0.12%     210.4MiB / 11.67GiB   1.76%     31.6MB / 25.4MB   369kB / 72.3MB    71
dfe1c5a0b6cd   streaming-stack_taskmanager.1.kdk4nn38ycowpn58ap0wj2hm8        1.00%     991.9MiB / 11.67GiB   8.30%     95.8MB / 79.7MB   7.75MB / 186MB    82
a4bcc1c1f865   streaming-stack_jobmanager.1.j3m7ic4m659uw776pt063wf4t         1.07%     1.684GiB / 11.67GiB   14.43%    251MB / 119MB     95.8MB / 294MB    132
a00cea8c318d   streaming-stack_jupyter.1.ovnff1xgurkbpm6a0janv3f96            0.00%     48.5MiB / 11.67GiB    0.41%     12.2kB / 300B     4.4MB / 15.2MB    3
c62d96402a36   streaming-stack_pinot-controller.1.p58k4jta4snild2rm3mxl38lw   0.59%     1.021GiB / 11.67GiB   8.74%     10.1MB / 12MB     242MB / 132MB     161
c62d1670365b   streaming-stack_pinot-server.1.rdschtw6ucclen2be8pkq78uw       0.39%     1.843GiB / 11.67GiB   15.79%    3.32MB / 4.15MB   15.9MB / 130MB    82
0bb1902e81a4   streaming-stack_pinot-broker.1.ud3xjbjao7rp7b02nbpwa2rg6       0.39%     2.908GiB / 11.67GiB   24.91%    4.81MB / 6.89MB   29.2MB / 37MB     84
```

## References

