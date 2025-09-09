### Setup Kafka environment

# 1. Download and exctract Kafka
# 2. Start the Kafka environment
```PowerShell
# ensure system tools are on PATH and skip memory autodetection (avoids WMIC)
$env:PATH = "C:\Windows\System32;C:\Windows;C:\Windows\System32\Wbem;$env:PATH"
$env:KAFKA_HEAP_OPTS = "-Xms512m -Xmx1g"

# create minimal single-node KRaft config (if not present)
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

# generate a cluster UUID
$env:KAFKA_CLUSTER_ID = (& .\bin\windows\kafka-storage.bat random-uuid 2>$null | Select-Object -Last 1).Trim()

# format log directories (KRaft)
.\bin\windows\kafka-storage.bat format --standalone -t $env:KAFKA_CLUSTER_ID -c .\config\kraft\server.properties

# start the broker (keep this window open)
.\bin\windows\kafka-server-start.bat .\config\kraft\server.properties
```

# 3. Create a topic to store your events
```PowerShell
.\bin\windows\kafka-topics.bat --create --topic quickstart-events --partitions 1 --replication-factor 1 --bootstrap-server localhost:9092
.\bin\windows\kafka-topics.bat --describe --topic quickstart-events --bootstrap-server localhost:9092
```

# 4. Write some events into the topic
```PowerShell
.\bin\windows\kafka-console-producer.bat --topic quickstart-events --bootstrap-server localhost:9092
# type lines and press Enter for each event
```

# 5. Read the events
```PowerShell
.\bin\windows\kafka-console-consumer.bat --topic quickstart-events --from-beginning --bootstrap-server localhost:9092
```
# 6. Import/export your data as streams of events with Kafka Connect
```PowerShell

# add plugin.path (required for file connectors)
Add-Content -Path .\config\connect-standalone.properties -Value 'plugin.path=libs/connect-file-4.1.0.jar'

# seed data
Set-Content -Path .\test.txt -Value 'foo'
Add-Content -Path .\test.txt -Value 'bar'

# run Connect in standalone mode (source + sink)
.\bin\windows\connect-standalone.bat .\config\connect-standalone.properties .\config\connect-file-source.properties .\config\connect-file-sink.properties

# verify output
Get-Content .\test.sink.txt

# optionally: inspect the topic
.\bin\windows\kafka-console-consumer.bat --bootstrap-server localhost:9092 --topic connect-test --from-beginning
```

# 7. Process your events with Kafka Streams
```PowerShell
# developer-oriented section (Java/Scala); no terminal commands here
```

# 8. Terminate the Kafka environment
```PowerShell
# stop producer/consumer/broker with Ctrl+C in their windows
# optionally remove local data
Remove-Item -Recurse -Force .\data\kraft-combined-logs -ErrorAction SilentlyContinue
Remove-Item -Recurse -Force .\logs -ErrorAction SilentlyContinue
```
