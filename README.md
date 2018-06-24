#!/bin/bash

set -v

dcos package install cassandra --name=/prod/cassandra # --package-version=2.1.0-3.0.16

dcos package install kafka --name=/prod/kafka --package-version=2.0.3-0.11.0

dcos package install spark --name=/prod/spark # --package-version=2.3.0-2.2.1-2

dcos job add prod/cassandra-schema.json
dcos job run initialize-prod-cassandra-schema-job

dcos spark --name=/prod/spark run --submit-args='--driver-cores 0.1 --driver-memory 1024M --total-executor-cores 4 --class de.nierbeck.floating.data.stream.spark.KafkaToCassandraSparkApp https://oss.sonatype.org/content/repositories/snapshots/de/nierbeck/floating/data/spark-digest_2.11/0.2.1-SNAPSHOT/spark-digest_2.11-0.2.1-SNAPSHOT-assembly.jar METRO-Vehicles node-0-server.prodcassandra.autoip.dcos.thisdcos.directory:9042 broker.prodkafka.l4lb.thisdcos.directory:9092'

dcos marathon app add prod/ingest/marathon.json
dcos marathon app add prod/ui/marathon.json


dcos task exec --interactive --tty prod_bus-demo_ui /bin/bash
curl http://169.254.169.254/latest/meta-data/public-ipv4

dcos task exec --interactive --tty prod_bus-demo_ui curl -s ifconfig.co


# Install Kubernetes from UI.
# While Kubernetes is installing, install /dev/cassandra, /dev/kafka, and /dev/spark.
# Check Kubernetes in UI, then go and set up the CLI.

dcos package install kubernetes --cli
dcos kubernetes kubeconfig

kubectl apply -f schema-pod.yaml

kubectl exec -it $(kubectl get pods --selector=app=schema-pod --output=jsonpath={.items..metadata.name}) /bin/bash

cqlsh node-0-server.devcassandra.autoip.dcos.thisdcos.directory
describe keyspace streaming;

/opt/bus-demo/import_data.sh node-0-server.devcassandra.autoip.dcos.thisdcos.directory

cqlsh node-0-server.devcassandra.autoip.dcos.thisdcos.directory
describe keyspace streaming;


# Check everything else is installed and then start the Spark job.
dcos spark --name=/dev/spark run --submit-args='--driver-cores 0.1 --driver-memory 1024M --total-executor-cores 4 --class de.nierbeck.floating.data.stream.spark.KafkaToCassandraSparkApp https://oss.sonatype.org/content/repositories/snapshots/de/nierbeck/floating/data/spark-digest_2.11/0.2.1-SNAPSHOT/spark-digest_2.11-0.2.1-SNAPSHOT-assembly.jar METRO-Vehicles node-0-server.devcassandra.autoip.dcos.thisdcos.directory:9042 broker.devkafka.l4lb.thisdcos.directory:9092'

# Install Jenkins.
# Create Jenkins pipeline.
# Kick of first build.

git merge dev
git push

# Go back to UI, wait for build.
# Go to UI, wait for bus-demo/ui to start.

dcos task exec --interactive --tty dev_bus-demo_ui curl -s ifconfig.co

# Go to browser, look at buses.




dcos kafka --name=/dev/kafka topic list

dcos task exec kafka-toolbox /bin/bash -c "export JAVA_HOME=/opt/jdk1.8.0_144/jre/; /opt/kafka_2.12-0.11.0.0/bin/kafka-console-consumer.sh --zookeeper master.mesos:2181/dcos-service-prod__kafka --topic METRO-Vehicles --property print.timestamp=true --property print.value=false"



dcos kafka --name=/prod/kafka update package-versions


dcos kafka --name=/prod/kafka update start --package-version=2.0.4-1.0.0



---------------------------------------------------------------------------------------------------------------------



dcos task exec --interactive --tty dev_bus-demo_ui curl -s ifconfig.co
dcos task exec --interactive --tty prod_bus-demo_ui curl -s ifconfig.co





dcos package repo add --index=0 edgelb-aws https://downloads.mesosphere.com/edgelb/v1.0.2/assets/stub-universe-edgelb.json
dcos package repo add --index=0 edgelb-pool-aws https://downloads.mesosphere.com/edgelb-pool/v1.0.2/assets/stub-universe-edgelb-pool.json


dcos package install edgelb --cli



echo "{ dcos-token: '$(dcos config show core.dcos_acs_token)' }" | mustache - dcos-token-credentials.json.mustache >dcos-token-credentials.json


curl -i -X POST --insecure $(dcos config show core.dcos_url)/service/jenkins/credentials/store/system/domain/_/createCredentials --header Authorization:token=$(dcos config show core.dcos_acs_token) --data-urlencode "json=$(cat dcos-token-credentials.json)"
