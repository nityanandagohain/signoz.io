---
title: Collecting Docker container logs
id: collect_docker_logs
---

# Collecting Docker container logs

With SigNoz you can collect all your docker container logs and perform different queries on top of it.
Below are the steps to collect docker container logs.

##  Steps for collecting logs if SigNoz is running on the same host.

* Modify the `docker-compose.yaml` file present inside `deploy/docker/clickhouse-setup` to run the OTEL collector as root user and mount the docker container directory as highlighted below.
    ```yaml {5,8}
    ...
    otel-collector:
        image: signoz/signoz-otel-collector:0.55.0-rc.3
        command: ["--config=/etc/otel-collector-config.yaml"]
        user: "root" # required for reading docker container logs
        volumes:
        - ./otel-collector-config.yaml:/etc/otel-collector-config.yaml
        - /var/lib/docker/containers:/var/lib/docker/containers:ro
    ...
    ```

* Add the filelog reciever to `otel-collector-config.yaml` which is present inside `deploy/docker/clickhouse-setup`
    ```yaml {2-31}
    receivers:
      filelog/containers:
        include: [  "/var/lib/docker/containers/*/*.log" ]
        start_at: end
        include_file_path: true
        include_file_name: false
        operators:
        - type: json_parser
          id: parser-docker
          output: extract_metadata_from_filepath
          timestamp:
            parse_from: attributes.time
            layout: '%Y-%m-%dT%H:%M:%S.%LZ'
        # Extract metadata from file path
        - type: regex_parser
          id: extract_metadata_from_filepath
          regex: '^.*containers/(?P<container_id>[^_]+)/.*log$'
          parse_from: attributes["log.file.path"]
          output: parse_body
        - type: move
          id: parse_body
          from: attributes.log
          to: body
          output: add_source
        - type: add
          id: add_source
          field: resource["source"]
          value: "docker"
        - type: remove
          id: time
          field: attributes.time
    ...
    ```
    Here we are parsing the logs and extracting different values like timestamp, body and removing duplicate fields using different operators that are available.
    You can read more about operators [here](./logs.md#operators-for-parsing-and-manipulating-logs)

* Next we will modify our pipeline inside `otel-collector-config.yaml` to include the receiver we have created above.
    ```yaml {4}
    service:
        ....
        logs:
            receivers: [otlp, filelog/containers]
            processors: [batch]
            exporters: [clickhouselogsexporter]
    ```

* Now we can restart the otel collector container so that new changes are applied and the docker container logs will be visible in SigNoz.

## Steps for collecting logs if SigNoz is running on a different host.

If you have a signoz running on a different host then you will have to run a otel-collector to export logs from your host to the host where SigNoz is running.

* We will create a `otel-collector-config.yaml`
  ```yaml
  receivers:
    filelog/containers:
      include: [  "/var/lib/docker/containers/*/*.log" ]
      start_at: end
      include_file_path: true
      include_file_name: false
      operators:
      - type: json_parser
        id: parser-docker
        output: extract_metadata_from_filepath
        timestamp:
          parse_from: attributes.time
          layout: '%Y-%m-%dT%H:%M:%S.%LZ'
      # Extract metadata from file path
      - type: regex_parser
        id: extract_metadata_from_filepath
        regex: '^.*containers/(?P<container_id>[^_]+)/.*log$'
        parse_from: attributes["log.file.path"]
        output: parse_body
      - type: move
        id: parse_body
        from: attributes.log
        to: body
        output: add_source
      - type: add
        id: add_source
        field: resource["source"]
        value: "docker"
      - type: remove
        id: time
        field: attributes.time
  processors:
    batch:
      send_batch_size: 10000
      send_batch_max_size: 11000
      timeout: 10s
  exporters:
    otlp/log:
      endpoint: http://<host>:<port>
      tls:
        insecure: true
  service:
    pipelines:
      logs:
        receivers: [filelog/containers]
        processors: [batch]
        exporters: [ otlp/log ]
  ```
  Here we are parsing the logs and extracting different values like timestamp, body and removing duplicate fields using different operators that are available. You can read more about operators [here](./logs.md#operators-for-parsing-and-manipulating-logs)

  The parsed logs are batched up using the batch processor and then exported to the host where signoz is deployed. The `oltp/log` exporter here uses a http endpoint but if you want to use https you will have to provide the certificate and the key. You can read more about it [here](https://github.com/open-telemetry/opentelemetry-collector/blob/main/exporter/otlpexporter/README.md)

  For finding the right host and port for your SigNoz cluster please follow the guide [here](../install/troubleshooting.md#signoz-otel-collector-address-grid).  

* We will start our otel-collector container and mount the docker container path so that the logs can be read for all containers.
  ```
  docker run -d --name signoz-host-otel-collector --user root -v /var/lib/docker/containers:/var/lib/docker/containers:ro -v $(pwd)/otel-collector-config.yaml:/etc/otel/config.yaml signoz/signoz-otel-collector:0.55.0-rc.3
  ```

* If there are no errors your logs will be exported and visible on the SigNoz UI. 