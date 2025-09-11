# Setup Kafka environment (based on quick start guide from Kafka: https://kafka.apache.org/quickstart)

### 1. Download and exctract Kafka
### 2. Start the Kafka environment
```PowerShell
### ensure system tools are on PATH and skip memory autodetection (avoids WMIC)
$env:PATH = "C:\Windows\System32;C:\Windows;C:\Windows\System32\Wbem;$env:PATH"
$env:KAFKA_HEAP_OPTS = "-Xms512m -Xmx1g"

### create minimal single-node KRaft config (if not present)
New-Item -ItemType Directory -Force .\config\kraft | Out-Null
$cfg = @(
'process.roles=broker,controller',
'node.id=1',
'controller.quorum.voters=1@localhost:9093',
'controller.listener.names=CONTROLLER',
'listeners=PLAINTEXT://:9092,CONTROLLER://:9093',
'advertised.listeners=PLAINTEXT://localhost:9092',
'listener.security.protocol.map=PLAINTEXT:PLAINTEXT,CONTROLLER:PLAINTEXT',
'inter.broker.listener.name=PLAINTEXT',
'log.dirs=./data/kraft-combined-logs'
)
$cfg | Set-Content -Path .\config\kraft\server.properties -Encoding UTF8

### generate a cluster UUID
$env:KAFKA_CLUSTER_ID = (& .\bin\windows\kafka-storage.bat random-uuid 2>$null | Select-Object -Last 1).Trim()

### format log directories (KRaft)
.\bin\windows\kafka-storage.bat format --standalone -t $env:KAFKA_CLUSTER_ID -c .\config\kraft\server.properties

### start the broker (keep this window open)
.\bin\windows\kafka-server-start.bat .\config\kraft\server.properties
```

### 3. Create a topic to store your events
```PowerShell
.\bin\windows\kafka-topics.bat --create --topic quickstart-events --partitions 1 --replication-factor 1 --bootstrap-server localhost:9092
.\bin\windows\kafka-topics.bat --describe --topic quickstart-events --bootstrap-server localhost:9092
```

### 4. Write some events into the topic
```PowerShell
.\bin\windows\kafka-console-producer.bat --topic quickstart-events --bootstrap-server localhost:9092
### type lines and press Enter for each event
```

### 5. Read the events
```PowerShell
.\bin\windows\kafka-console-consumer.bat --topic quickstart-events --from-beginning --bootstrap-server localhost:9092
```
### 6. Import/export your data as streams of events with Kafka Connect
```PowerShell

### add plugin.path (required for file connectors)
Add-Content -Path .\config\connect-standalone.properties -Value 'plugin.path=libs/connect-file-4.1.0.jar'

### seed data
Set-Content -Path .\test.txt -Value 'foo'
Add-Content -Path .\test.txt -Value 'bar'

### run Connect in standalone mode (source + sink)
.\bin\windows\connect-standalone.bat .\config\connect-standalone.properties .\config\connect-file-source.properties .\config\connect-file-sink.properties

### verify output
Get-Content .\test.sink.txt

### optionally: inspect the topic
.\bin\windows\kafka-console-consumer.bat --bootstrap-server localhost:9092 --topic connect-test --from-beginning
```

### 7. Process your events with Kafka Streams
```PowerShell
### developer-oriented section (Java/Scala); no terminal commands here
```

### 8. Terminate the Kafka environment
```PowerShell
### stop producer/consumer/broker with Ctrl+C in their windows
### optionally remove local data
Remove-Item -Recurse -Force .\data\kraft-combined-logs -ErrorAction SilentlyContinue
Remove-Item -Recurse -Force .\logs -ErrorAction SilentlyContinue
```


# Clickstream demo
From now on I'm working with ubuntu, not windows as in previous part

### 1. Install Kafka
### 2. Verify that you have all remaining needed tools 
### 3. Increase max_map_count
```
sudo sysctl -w vm.max_map_count=262144
```
### 4. Clone repository
### 5. Download needed connectors
```
docker run --rm -v $PWD/confluent-hub-components:/usr/share/confluent-hub-components confluentinc/cp-kafka-connect:7.9.0 bash -c 'confluent-hub install --no-prompt confluentinc/kafka-connect-elasticsearch:10.0.2'
docker run --rm -v $PWD/confluent-hub-components:/usr/share/confluent-hub-components confluentinc/cp-kafka-connect:7.9.0 bash -c 'confluent-hub install --no-prompt confluentinc/kafka-connect-datagen:0.4.0'
```
### 6. Give full access to confluent-hub-components
```
chmod -R 777 confluent-hub-components/
```
### 7. Compose up
```
docker-compose up -d
```
### 8. Verify containers
```
docker-compose ps -a
```
### 9. Launch ksqlDB CLI
```
docker-compose exec ksqldb-cli ksql http://ksqldb-server:8088
```
### 10. Log topics
```
show topics;
```
### 11. Create connectors with scripts below
```
RUN SCRIPT '/scripts/create-connectors.sql';
```
### 12. Query sample data from every connector, to verify<br>
```
print clickstream_users limit 3;
print clickstream_codes limit 3;
print clickstream limit 3;
```
### 13. Check connectors on Confluent Control Center
<img width="1519" height="810" alt="confluent_flow_screen" src="https://github.com/user-attachments/assets/b2ab32dc-81bd-4ce9-9d5e-ce6187289dea" /><br>
### 14. Load data
```
RUN SCRIPT '/scripts/statements.sql';
```
### 15. Verify data
<img width="1278" height="887" alt="confluent_ksql_screen" src="https://github.com/user-attachments/assets/632596e3-9216-46b4-b60a-58ac7f1588b4" /><br>
<img width="1519" height="810" alt="confluent_flow_screen" src="https://github.com/user-attachments/assets/6dfb58a5-5313-406e-bdad-b241766dfd74" /><br>
### 16. Set up the required Elasticsearch document mapping template<br>
```
docker-compose exec elasticsearch bash -c '/scripts/elastic-dynamic-template.sh'
```
### 17. Send the ksqlDB tables to Elasticsearch and Grafana<br>
```
docker-compose exec ksqldb-server bash -c '/scripts/ksql-tables-to-grafana.sh'
```
### 18. Load dashboard into Grafana<br>
```
docker-compose exec grafana bash -c '/scripts/clickstream-analysis-dashboard.sh'
```
<img width="1843" height="971" alt="grafana_screen" src="https://github.com/user-attachments/assets/cc3a1bac-0ab1-4bf4-8a47-fd3836c11fe3" /><br>
<img width="1736" height="950" alt="confluent_screen" src="https://github.com/user-attachments/assets/95108ed8-fe63-4622-a554-8abe4f2b2e46" /><br>

Might be that you reach capacity of disk space, especially when working with local vm, as I did in this scenario. Elasticsearch, in that case, will automatically set indexes to read-only. You will have to 
increase disk space, and manualy set "read_only_allow_delete" var to "false":<br>
```
curl -XPUT "http://localhost:9200/_settings" -H 'Content-Type: application/json' -d'
{
  "index": {
    "blocks": {
      "read_only_allow_delete": "false"
    }
  }
}'
```
### 19. Mimic user activity sessions with script below<br>
```
./sessionize-data.sh
```
### 20. Result in Grafana<br>
<img width="1837" height="867" alt="grafana_user_sessions_screen" src="https://github.com/user-attachments/assets/2db6f2f8-f552-415d-8ee6-7dc06268f00f" /><br>


