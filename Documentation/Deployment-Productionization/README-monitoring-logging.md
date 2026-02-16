# Monitoring & Logging

## Deployed Monitoring & Logging using Percona Monitoring and Management (PMM)

Command to deploy PMM Server:
```
$ curl -fsSL https://www.percona.com/get/pmm | /bin/bash
Gathering/downloading required components, this may take a moment

Checking docker installation - installed.

Created PMM Network: pmm-net
Generated Watchtower Token: random-2026-02-15-233944
Pulling percona/pmm-server:3
Created PMM Data Volume: pmm-data
Starting PMM Server...
Created PMM Server: pmm-server

Use the following command if you ever need to update your container manually:
	docker pull percona/pmm-server:3

	docker run -d -p 443:8443 --volume pmm-data:/srv --name pmm-server --network pmm-net -e PMM_WATCHTOWER_HOST=http://watchtower:8080 -e PMM_WATCHTOWER_TOKEN=random-2026-02-15-233944  --restart always percona/pmm-server:3

Unable to find image 'percona/watchtower:latest' locally
latest: Pulling from percona/watchtower
a75b1cc7793e: Pull complete
94c57848d76e: Pull complete
a20740e0c31d: Pull complete
757870dd4d2e: Download complete
Digest: sha256:5446cde8be55b10c693fa76ab758d992c9b27f598cd2aeb25539f14c6d6bf37c
Status: Downloaded newer image for percona/watchtower:latest
9629a1820215aa567d4a7d496c4980821b486992781e8c2c6e6387f114609041
Created Watchtower container
PMM Server has been successfully setup on this system!

You can access your new server using one of the following web addresses:
	https://127.0.0.1:443/
	https://172.31.28.194:443/
	https://172.17.0.1:443/
	https://172.18.0.1:443/

The default username is 'admin' and the password is 'admin' :)
Note: Some browsers may not trust the default SSL certificate when you first open one of the urls above.
If this is the case, Chrome users may want to type 'thisisunsafe' to bypass the warning.

Enjoy Percona Monitoring and Management!
```
Reference: https://docs.percona.com/percona-monitoring-and-management/3/install-pmm/install-pmm-server/deployment-options/docker/index.html#__tabbed_1_1

## Access PMM Server

We can access via browser at https://52.63.115.158:443, ignore the warning and click "Advanced" then "Proceed to 52.63.115.158 (unsafe)".
Use these credentials:
```
Username: admin
Password: Yogyakembal1
```
![alt text](image-4.png)

## Add PMM Client
SSH to the client server that we want to monitor:
```
ssh ubuntu@52.91.150.166
```

Install PMM Client:
``` bash
sudo apt-get update
sudo apt-get install -y percona-pmm-client
```

Add PMM Client to PMM Server:
``` bash
pmm-admin config --server-insecure-tls \
    --server-url=https://service_token:example_token@52.63.115.158:443 \
    52.91.150.166 generic example-nodejs-api
```
Example result:

![alt text](image-6.png)

![alt text](image-5.png)

Reference: https://docs.percona.com/percona-monitoring-and-management/3/install-pmm/install-pmm-client/package_manager.html

## Deploy Logging using Loki and Alloy

### Deploy Loki
In monitoring server, we deploy Loki using docker-compose. Below is the example docker-compose.yml file:
```
services:

  loki:
    image: grafana/loki:${GRAFANA_LOKI_VERSION:-3.5.2}
    ports:
      - 3100:3100/tcp
    volumes:
      - ./loki-config.yaml:/etc/loki/local-config.yaml
    command: -config.file=/etc/loki/local-config.yaml
    networks:
      monitoring_net:
        ipv4_address: 172.20.0.10

networks:
  monitoring_net:
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.0.0/24

```
and loki-config.yaml:
```
# This is a complete configuration to deploy Loki backed by the filesystem.
# The index will be shipped to the storage via tsdb-shipper.

auth_enabled: false

limits_config:
  allow_structured_metadata: true
  volume_enabled: true
  retention_period: 4320h
  max_query_lookback: 4320h
  max_query_length: 4320h
  reject_old_samples: false
#  reject_old_samples_max_age: 15000h
  ingestion_rate_mb: 50
  ingestion_burst_size_mb: 100
  max_streams_per_user: 100000

server:
  http_listen_port: 3100

common:
  ring:
    instance_addr: 0.0.0.0
    kvstore:
      store: inmemory
  replication_factor: 1
  path_prefix: /tmp/loki

schema_config:
  configs:
  - from: 2020-05-15
    store: tsdb
    object_store: filesystem
    schema: v13
    index:
      prefix: index_
      period: 24h

storage_config:
  tsdb_shipper:
    active_index_directory: /tmp/loki/index
    cache_location: /tmp/loki/index_cache
  filesystem:
    directory: /tmp/loki/chunks

pattern_ingester:
  enabled: false

# Note: We are setting the max chunk age far lower than the default expected value
# This is due to the fact this scenario is used within the LogCLI demo and we need a short flush time.
# To show how logcli stats --since 24h '{service_name="Delivery World", package_size="Large"}' works.
ingester:
  lifecycler:
    ring:
      kvstore:
        store: inmemory
      replication_factor: 1
    final_sleep: 0s

  max_chunk_age: 120m
```
Add Loki as the data sources on Grafana PMM Server:
https://52.63.115.158/pmm-ui/graph/connections/datasources

![alt text](image-7.png)

### Deploy Alloy
In client server, we deploy Alloy using docker systemd. We can reference from this documentation: https://grafana.com/docs/alloy/latest/set-up/install/linux/ for the installation.

```
sudo apt install gpg
sudo mkdir -p /etc/apt/keyrings/
wget -q -O - https://apt.grafana.com/gpg.key | gpg --dearmor | sudo tee /etc/apt/keyrings/grafana.gpg
echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com stable main" | sudo tee /
sudo apt-get update
sudo apt-get install alloy
```

And this is the /etc/alloy/config.alloy file:
```
// Discover Docker containers
discovery.docker "containers" {
  host = "unix:///var/run/docker.sock"
}

// Keep only your container
discovery.relabel "nodejs_container" {
  targets = discovery.docker.containers.targets

  rule {
    source_labels = ["__meta_docker_container_name"]
    regex         = "example-nodejs-api-app-1"
    action        = "keep"
  }
}

// Read logs from Docker
loki.source.docker "nodejs_logs" {
  host       = "unix:///var/run/docker.sock"
  targets    = discovery.relabel.nodejs_container.output
  forward_to = [loki.write.default.receiver]
}

// Send logs to Loki
loki.write "default" {
  endpoint {
    url = "http://52.63.115.158:3100/loki/api/v1/push"
  }
}
```
