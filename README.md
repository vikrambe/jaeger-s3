This is the repository that contains object storage (S3/GCS/AzureBlob) plugin for Jaeger.

## About
S3, Google Cloud Storage(GCS) and Microsoft Azure Blob Storage object storage support for Jaeger. 

Amazon DynamoDB and Google BigTable for indexes **may** work with some changes to configuration file. Reports on testing on these storage backends is appreciated.

With this plugin, you won't need to run and maintain Tempo, at all!
## Preresquities
None. No longer needs my custom jaeger code. Just use the official ones.

You can now use this plugin with Jaeger Operator/Helmchart/K8s since there are no dependency on my custom Jaeger code.

Scroll down to the end to see how to do a demo for Jaeger Operator on Kubernetes.

## Build/Compile
In order to compile the plugin from source code you can use `go build`:

```
cd /path/to/jaeger-s3
go build ./cmd/jaeger-s3/
```

## Configuration
#### Storage
[https://github.com/grafana/loki/blob/37a7189d4ed76655144d982e2eeebf495e0809ea/docs/sources/configuration/_index.md#storage_config](https://github.com/grafana/loki/blob/37a7189d4ed76655144d982e2eeebf495e0809ea/docs/sources/configuration/_index.md#storage_config)
#### Index (schema config)
[https://github.com/grafana/loki/blob/37a7189d4ed76655144d982e2eeebf495e0809ea/docs/sources/configuration/_index.md#schema_config](https://github.com/grafana/loki/blob/37a7189d4ed76655144d982e2eeebf495e0809ea/docs/sources/configuration/_index.md#schema_config)
#### More info
[https://grafana.com/docs/loki/latest/operations/storage/boltdb-shipper/](https://grafana.com/docs/loki/latest/operations/storage/boltdb-shipper/)

Sample basic config (AWS):
```
schema_config:
  configs:
    - from: 2018-10-24
      store: boltdb-shipper
      object_store: s3
      schema: v11
      index:
        prefix: index_
        period: 24h
      row_shards: 10

storage_config:
  aws:
    bucketnames: bucketname
    region: ap-southeast-1
    access_key_id: aws_access_key_id
    secret_access_key: aws_secret_access_key
    endpoint: s3.ap-southeast-1.amazonaws.com
    http_config:
      idle_conn_timeout: 90s
      response_header_timeout: 0s
      tls_handshake_timeout: 3s # change this to something larger if you have `TLS Handshake Timeout` or 0 to disable timeout
  boltdb_shipper:
    active_index_directory: /tmp/loki/boltdb-shipper-active
    cache_location: /tmp/loki/boltdb-shipper-cache
    cache_ttl: 24h         # Can be increased for faster performance over longer query periods, uses more disk space
    shared_store: s3
  filesystem:
    directory: /tmp/loki/chunks

compactor:
  working_directory: /tmp/loki/boltdb-shipper-compactor
  shared_store: s3
```

Sample basic config (GCS):
```
storage_config:
  boltdb_shipper:
    active_index_directory: /loki/boltdb-shipper-active
    cache_location: /loki/boltdb-shipper-cache
    cache_ttl: 24h         # Can be increased for faster performance over longer query periods, uses more disk space
    shared_store: gcs
  gcs:
      bucket_name: <bucket>

schema_config:
  configs:
    - from: 2020-07-01
      store: boltdb-shipper
      object_store: gcs
      schema: v11
      index:
        prefix: index_
        period: 24h

compactor:
  working_directory: /tmp/loki/boltdb-shipper-compactor
  shared_store: gcs
```

Sample basic config (Azure BlobStorage):
```
schema_config:
  configs:
    - from: 2020-07-01
      store: boltdb-shipper
      object_store: azure
      schema: v11  
      index:
        prefix: index_
        period: 24h

storage_config:
  boltdb_shipper:
    active_index_directory: /data/loki/index
    shared_store: azure
    cache_location: /data/loki/boltdb-cache
  azure:
    container_name: .. # add container name here
    account_name: .. # add account name here
    account_key: .. # add access key here

compactor:
  working_directory: /tmp/loki/boltdb-shipper-compactor
  shared_store: azure
```

## Start
In order to start plugin just tell jaeger the path to a config compiled plugin.

```
GRPC_STORAGE_PLUGIN_BINARY="./jaeger-s3" GRPC_STORAGE_PLUGIN_CONFIGURATION_FILE=./config-example.yaml SPAN_STORAGE_TYPE=grpc-plugin  GRPC_STORAGE_PLUGIN_LOG_LEVEL=DEBUG ./all-in-one --sampling.strategies-file=/location/of/your/jaeger/cmd/all-in-one/sampling_strategies.json
```

For Jaeger Operator on Kubernetes for testing/demo **!!NOT PRODUCTION!!**, sample manifest:

```
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: observability

commonLabels:
  app.kubernetes.io/instance: observability

resources:
  - https://raw.githubusercontent.com/jaegertracing/jaeger-operator/master/deploy/crds/jaegertracing.io_jaegers_crd.yaml
  - https://raw.githubusercontent.com/jaegertracing/jaeger-operator/master/deploy/service_account.yaml
  - https://raw.githubusercontent.com/jaegertracing/jaeger-operator/master/deploy/role.yaml
  - https://raw.githubusercontent.com/jaegertracing/jaeger-operator/master/deploy/role_binding.yaml
  - https://raw.githubusercontent.com/jaegertracing/jaeger-operator/master/deploy/operator.yaml
  - https://raw.githubusercontent.com/jaegertracing/jaeger-operator/master/deploy/cluster_role.yaml
  - https://raw.githubusercontent.com/jaegertracing/jaeger-operator/master/deploy/cluster_role_binding.yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jaeger-operator
  app: jaeger
spec:
  template:
    spec:
      containers:
      - name: jaeger-operator
        image: jaegertracing/jaeger-operator:master
        args: ["start"]
        env:
        - name: LOG-LEVEL
          value: debug
---
apiVersion: jaegertracing.io/v1
kind: Jaeger
metadata:
  name: jaeger-s3
spec:
  strategy: allInOne
  allInOne:
    image: jaegertracing/all-in-one:latest
    options:
      log-level: debug
  storage:
    type: grpc-plugin
    grpcPlugin:
      image: ghcr.io/muhammadn/jaeger-s3:latest
    options:
      grpc-storage-plugin:
        binary: /plugin/jaeger-s3
        configuration-file: /plugin-config/config-example.yaml
        log-level: debug
  volumeMounts:
    - name: config-volume
      mountPath: /plugin-config
  volumes:
    - name: config-volume
      configMap:
        name: jaeger-s3-config
---
apiVersion: v1
data:
  config-example.yaml: |-
    schema_config:
      configs:
        - from: 2018-10-24
          store: boltdb-shipper
          object_store: s3
          schema: v11
          index:
            prefix: index_
            period: 24h
          row_shards: 10

    storage_config:
      aws:
        bucketnames: yourbuckethere
        region: ap-southeast-1
        access_key_id: youraccesskey
        secret_access_key: youraccesssecret
        endpoint: s3.ap-southeast-1.amazonaws.com
        http_config:
          idle_conn_timeout: 90s
          response_header_timeout: 0s
          tls_handshake_timeout: 3s # change this to something larger if you have `TLS Handshake Timeout` or 0 to disable timeout
      boltdb_shipper:
        active_index_directory: /tmp/loki/boltdb-shipper-active
        cache_location: /tmp/loki/boltdb-shipper-cache
        cache_ttl: 24h         # Can be increased for faster performance over longer query periods, uses more disk space
        shared_store: s3
      filesystem:
        directory: /tmp/loki/chunks

    compactor:
      working_directory: /tmp/loki/boltdb-shipper-compactor
      shared_store: s3
kind: ConfigMap
metadata:
  name: jaeger-s3-config
```

## License

The S3 Storage gRPC Plugin for Jaeger is an [MIT licensed](LICENSE) open source project.
