# Kafka Setup (Hard + Docker)

This repository documents how I installed and tested Apache Kafka in two ways:

1. **Hard (manual) setup** on an Ubuntu 24.04 VM (Kafka 4.1.1).
2. **Docker setup** using `docker compose` with:
   - Kafka (bitnami legacy image, KRaft mode)
   - Kafka UI (provectuslabs/kafka-ui)

Both setups are tested using **CLI producers/consumers**.

---

## Part 1 – Hard Setup on Ubuntu 24.04

### 1. Environment

- **OS:** Ubuntu 24.04 (VM)
- **Java:** OpenJDK 21
- **Kafka:** `kafka_2.13-4.1.1` (KRaft mode, no ZooKeeper)

### 2. System update and Java installation

```bash
sudo apt update
sudo apt upgrade -y

sudo apt install -y openjdk-21-jdk
java -version
```
### 3. Kafka user and directories
```bash 
# Create dedicated system user for Kafka
sudo useradd -r -m -U -d /opt/kafka -s /bin/false kafka

# Make sure directory exists and belongs to kafka user
sudo mkdir -p /opt/kafka
sudo chown -R kafka:kafka /opt/kafka
```

### 4. Download and extract Kafka (4.1.1)
```bash 
cd /tmp

wget https://downloads.apache.org/kafka/4.1.1/kafka_2.13-4.1.1.tgz

sudo tar -xzf kafka_2.13-4.1.1.tgz -C /opt/kafka --strip-components=1
sudo chown -R kafka:kafka /opt/kafka
```
Now /opt/kafka contains bin/, config/, libs/, etc.

### 5. Kafka configuration (KRaft mode)
```bash
sudo nano /opt/kafka/config/server.properties
```
I added this key configuration value:
```bash
# Voters in the controller quorum (single node)
controller.quorum.voters=1@localhost:9093
```
controller.quorum.voters → single-node controller quorum.

### 6. Initialize KRaft storage (Cluster ID)

Generate a cluster ID:
```bash
sudo /opt/kafka/bin/kafka-storage.sh random-uuid
```
Format the storage with this ID:
```bash 
sudo /opt/kafka/bin/kafka-storage.sh format \
  -t ID_Generated \
  -c /opt/kafka/config/server.properties
```

### 7. systemd service for Kafka
Create service file:
```bash 
sudo nano /etc/systemd/system/kafka.service
```
Content:
```bash
[Unit]
Description=Apache Kafka Server
After=network.target

[Service]
Type=simple
User=kafka
ExecStart=/opt/kafka/bin/kafka-server-start.sh /opt/kafka/config/server.properties
ExecStop=/opt/kafka/bin/kafka-server-stop.sh
Restart=on-abnormal

[Install]
WantedBy=multi-user.target
```
Reload and start:
```bash
sudo systemctl daemon-reload
sudo systemctl enable kafka
sudo systemctl start kafka
sudo systemctl status kafka
```
Kafka now be running on localhost:9092.

---

## Part 2 – Docker Setup (Kafka + Kafka UI)

For the Docker setup I created a small project folder:
kafka-docker/
  docker-compose.yml

### 8. Requirements
Docker installed and running.

Docker Compose v2 available via docker compose command.

### 9. docker-compose.yml
Write File: kafka-docker/docker-compose.yml:

#### Notes:

* bitnamilegacy/kafka:3.7.0 is a Kafka image that supports KRaft mode.

* KAFKA_ENABLE_KRAFT=yes → KRaft mode (no ZooKeeper).

* KAFKA_CFG_ADVERTISED_LISTENERS=PLAINTEXT://kafka:9092 → other containers (like kafka-ui) connect using the service name kafka.

* Kafka UI connects to kafka:9092 and exposes a web interface on port 8080.

### 10. Running the Docker stack

From inside kafka-docker/:
```bash 
# Make sure systemd Kafka is stopped so ports don't conflict
sudo systemctl stop kafka || true

docker compose up -d
docker ps
```
Then open Kafka UI in a browser:

From outside VM: http://<VM-IP>:8080

You should see the local cluster and its topics.

## Part 3 – Testing (Publish / Consume)

The last part of the task is to test both setups using publish/consume via CLI.

### A. Testing the hard setup (systemd Kafka)

Make sure docker Kafka is stopped when you test this:
#### 1. Start Kafka service
```bash
sudo systemctl start kafka
sudo systemctl status kafka
```
#### 2. Create a test topic
```bash 
/opt/kafka/bin/kafka-topics.sh \
  --create \
  --topic hard-test-topic \
  --bootstrap-server localhost:9092 \
  --partitions 1 \
  --replication-factor 1
```
#### 3. Start a consumer (terminal 1)
```bash 
/opt/kafka/bin/kafka-console-consumer.sh \
  --topic hard-test-topic \
  --from-beginning \
  --bootstrap-server localhost:9092
```

#### 4. Start a producer (terminal 2)
```bash 
/opt/kafka/bin/kafka-console-producer.sh \
  --topic hard-test-topic \
  --bootstrap-server localhost:9092
  ```
Look at images.

### B. Testing the Docker setup (Kafka inside container)
```bash
docker exec -it kafka bash
```
and make the same commands

# Summary

## Hard setup: Kafka 4.1.1 installed manually on Ubuntu 24.04, running as a systemd service, tested with CLI.

## Docker setup: Kafka 3.7.0 (bitnami legacy image) + Kafka UI (provectuslabs/kafka-ui) running via docker compose, tested with producer/consumer from inside the Kafka container.

## Both setups successfully created topics and produced/consumed messages and test it.




