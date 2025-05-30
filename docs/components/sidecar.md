# Sidecar

The `thanos sidecar` command runs a component that gets deployed along with a Prometheus instance. This allows sidecar to optionally upload metrics to object storage and allow [Queriers](query.md) to query Prometheus data with common, efficient StoreAPI.

In details:

* It implements Thanos' Store API on top of Prometheus' remote-read API. This allows [Queriers](query.md) to treat Prometheus servers as yet another source of time series data without directly talking to its APIs.
* Optionally, the sidecar uploads TSDB blocks to an object storage bucket as Prometheus produces them every 2 hours. This allows Prometheus servers to be run with relatively low retention while their historic data is made durable and queryable via object storage.

  NOTE: This still does NOT mean that Prometheus can be fully stateless, because if it crashes and restarts you will lose ~2 hours of metrics, so persistent disk for Prometheus is highly recommended. The closest to stateless you can get is using remote write (which Thanos supports, see [Receiver](receive.md). Remote write has other risks and consequences, and still if crashed you lose in positive case seconds of metrics data, so persistent disk is recommended in all cases.

* Optionally Thanos sidecar is able to watch Prometheus rules and configuration, decompress and substitute environment variables if needed and ping Prometheus to reload them. Read more about this in [here](#reloader-configuration)

Prometheus servers connected to the Thanos cluster via the sidecar are subject to a few limitations and recommendations for safe operations:

* The recommended Prometheus version is 2.2.1 or greater (including newest releases). This is due to Prometheus instability in previous versions as well as lack of `flags` endpoint.
* (!) The Prometheus `external_labels` section of the Prometheus configuration file has unique labels in the overall Thanos system. Those external labels will be used by the sidecar and then Thanos in many places. See [external labels](../storage.md#external-labels) docs.
* The `--web.enable-admin-api` flag is enabled to support sidecar to get metadata from Prometheus like external labels.
* The `--web.enable-lifecycle` flag is enabled if you want to use sidecar reloading features (`--reload.*` flags).

If you choose to use the sidecar to also upload data to object storage:

* Must specify object storage (`--objstore.*` flags)
* It only uploads uncompacted Prometheus blocks. For compacted blocks, see [Upload compacted blocks](#upload-compacted-blocks).
* The `--storage.tsdb.min-block-duration` and `--storage.tsdb.max-block-duration` must be set to equal values to disable local compaction in order to use Thanos sidecar upload, otherwise leave local compaction on if sidecar just exposes StoreAPI and your retention is normal. The default of `2h` is recommended. Mentioned parameters set to equal values disable the internal Prometheus compaction, which is needed to avoid the corruption of uploaded data when Thanos compactor does its job, this is critical for data consistency and should not be ignored if you plan to use Thanos compactor. Even though you set mentioned parameters equal, you might observe Prometheus internal metric `prometheus_tsdb_compactions_total` being incremented, don't be confused by that: Prometheus writes initial head block to filesystem via its internal compaction mechanism, but if you have followed recommendations - data won't be modified by Prometheus before the sidecar uploads it. Thanos sidecar will also check sanity of the flags set to Prometheus on the startup and log errors or warning if they have been configured improperly (#838).
* The retention of Prometheus is recommended to not be lower than three times of the min block duration, so 6 hours. This achieves resilience in the face of connectivity issues to the object storage since all local data will remain available within the Thanos cluster. If connectivity gets restored the backlog of blocks gets uploaded to the object storage.

## Reloader Configuration

Thanos can watch changes in Prometheus configuration and refresh Prometheus configuration if `--web.enable-lifecycle` enabled.

You can configure watching for changes in directory via `--reloader.rule-dir=DIR_NAME` flag.

Thanos sidecar can watch `--reloader.config-file=CONFIG_FILE` configuration file, replace environment variables found in there in `$(VARIABLE)` format, and produce generated config in `--reloader.config-envsubst-file=OUT_CONFIG_FILE` file.

## Example basic deployment

```bash
prometheus \
  --storage.tsdb.max-block-duration=2h \
  --storage.tsdb.min-block-duration=2h \
  --web.enable-lifecycle
```

```bash
thanos sidecar \
    --tsdb.path        "/path/to/prometheus/data/dir" \
    --prometheus.url   "http://localhost:9090" \
    --objstore.config-file  "bucket.yml"
```

The example content of `bucket.yml`:

```yaml mdox-exec="go run scripts/cfggen/main.go --name=gcs.Config"
type: GCS
config:
  bucket: ""
  service_account: ""
  use_grpc: false
  grpc_conn_pool_size: 0
  http_config:
    idle_conn_timeout: 0s
    response_header_timeout: 0s
    insecure_skip_verify: false
    tls_handshake_timeout: 0s
    expect_continue_timeout: 0s
    max_idle_conns: 0
    max_idle_conns_per_host: 0
    max_conns_per_host: 0
    tls_config:
      ca_file: ""
      cert_file: ""
      key_file: ""
      server_name: ""
      insecure_skip_verify: false
    disable_compression: false
  chunk_size_bytes: 0
  max_retries: 0
prefix: ""
```

## Upload compacted blocks

If you want to migrate from a pure Prometheus setup to Thanos and have to keep the historical data, you can use the flag `--shipper.upload-compacted`. This will also upload blocks that were compacted by Prometheus. Values greater than 1 in the `compaction.level` field of a Prometheus block’s `meta.json` file indicate level of compaction.

To use this, the Prometheus compaction needs to be disabled. This can be done by setting the following flags for Prometheus:

- `--storage.tsdb.min-block-duration=2h`
- `--storage.tsdb.max-block-duration=2h`

## Flags

```$ mdox-exec="thanos sidecar --help"
usage: thanos sidecar [<flags>]

Sidecar for Prometheus server.


Flags:
  -h, --[no-]help                Show context-sensitive help (also try
                                 --help-long and --help-man).
      --[no-]version             Show application version.
      --log.level=info           Log filtering level.
      --log.format=logfmt        Log format to use. Possible options: logfmt or
                                 json.
      --tracing.config-file=<file-path>
                                 Path to YAML file with tracing
                                 configuration. See format details:
                                 https://thanos.io/tip/thanos/tracing.md/#configuration
      --tracing.config=<content>
                                 Alternative to 'tracing.config-file' flag
                                 (mutually exclusive). Content of YAML file
                                 with tracing configuration. See format details:
                                 https://thanos.io/tip/thanos/tracing.md/#configuration
      --[no-]enable-auto-gomemlimit
                                 Enable go runtime to automatically limit memory
                                 consumption.
      --auto-gomemlimit.ratio=0.9
                                 The ratio of reserved GOMEMLIMIT memory to the
                                 detected maximum container or system memory.
      --http-address="0.0.0.0:10902"
                                 Listen host:port for HTTP endpoints.
      --http-grace-period=2m     Time to wait after an interrupt received for
                                 HTTP Server.
      --http.config=""           [EXPERIMENTAL] Path to the configuration file
                                 that can enable TLS or authentication for all
                                 HTTP endpoints.
      --grpc-address="0.0.0.0:10901"
                                 Listen ip:port address for gRPC endpoints
                                 (StoreAPI). Make sure this address is routable
                                 from other components.
      --grpc-server-tls-cert=""  TLS Certificate for gRPC server, leave blank to
                                 disable TLS
      --grpc-server-tls-key=""   TLS Key for the gRPC server, leave blank to
                                 disable TLS
      --grpc-server-tls-client-ca=""
                                 TLS CA to verify clients against. If no
                                 client CA is specified, there is no client
                                 verification on server side. (tls.NoClientCert)
      --grpc-server-tls-min-version="1.3"
                                 TLS supported minimum version for gRPC server.
                                 If no version is specified, it'll default to
                                 1.3. Allowed values: ["1.0", "1.1", "1.2",
                                 "1.3"]
      --grpc-server-max-connection-age=60m
                                 The grpc server max connection age. This
                                 controls how often to re-establish connections
                                 and redo TLS handshakes.
      --grpc-grace-period=2m     Time to wait after an interrupt received for
                                 GRPC Server.
      --prometheus.url=http://localhost:9090
                                 URL at which to reach Prometheus's API.
                                 For better performance use local network.
      --prometheus.ready_timeout=10m
                                 Maximum time to wait for the Prometheus
                                 instance to start up
      --prometheus.get_config_interval=30s
                                 How often to get Prometheus config
      --prometheus.get_config_timeout=30s
                                 Timeout for getting Prometheus config
      --prometheus.http-client-file=<file-path>
                                 Path to YAML file or string with http
                                 client configs. See Format details:
                                 https://thanos.io/tip/components/sidecar.md/#configuration.
      --prometheus.http-client=<content>
                                 Alternative to 'prometheus.http-client-file'
                                 flag (mutually exclusive). Content
                                 of YAML file or string with http
                                 client configs. See Format details:
                                 https://thanos.io/tip/components/sidecar.md/#configuration.
      --tsdb.path="./data"       Data directory of TSDB.
      --reloader.config-file=""  Config file watched by the reloader.
      --reloader.config-envsubst-file=""
                                 Output file for environment variable
                                 substituted config file.
      --reloader.rule-dir=RELOADER.RULE-DIR ...
                                 Rule directories for the reloader to refresh
                                 (repeated field).
      --reloader.watch-interval=3m
                                 Controls how often reloader re-reads config and
                                 rules.
      --reloader.retry-interval=5s
                                 Controls how often reloader retries config
                                 reload in case of error.
      --reloader.method=http     Method used to reload the configuration.
      --reloader.process-name="prometheus"
                                 Executable name used to match the process being
                                 reloaded when using the signal method.
      --request.logging-config-file=<file-path>
                                 Path to YAML file with request logging
                                 configuration. See format details:
                                 https://thanos.io/tip/thanos/logging.md/#configuration
      --request.logging-config=<content>
                                 Alternative to 'request.logging-config-file'
                                 flag (mutually exclusive). Content
                                 of YAML file with request logging
                                 configuration. See format details:
                                 https://thanos.io/tip/thanos/logging.md/#configuration
      --objstore.config-file=<file-path>
                                 Path to YAML file that contains object
                                 store configuration. See format details:
                                 https://thanos.io/tip/thanos/storage.md/#configuration
      --objstore.config=<content>
                                 Alternative to 'objstore.config-file'
                                 flag (mutually exclusive). Content of
                                 YAML file that contains object store
                                 configuration. See format details:
                                 https://thanos.io/tip/thanos/storage.md/#configuration
      --[no-]shipper.upload-compacted
                                 If true shipper will try to upload compacted
                                 blocks as well. Useful for migration purposes.
                                 Works only if compaction is disabled on
                                 Prometheus. Do it once and then disable the
                                 flag when done.
      --hash-func=               Specify which hash function to use when
                                 calculating the hashes of produced files.
                                 If no function has been specified, it does not
                                 happen. This permits avoiding downloading some
                                 files twice albeit at some performance cost.
                                 Possible values are: "", "SHA256".
      --shipper.meta-file-name="thanos.shipper.json"
                                 the file to store shipper metadata in
      --store.limits.request-series=0
                                 The maximum series allowed for a single Series
                                 request. The Series call fails if this limit is
                                 exceeded. 0 means no limit.
      --store.limits.request-samples=0
                                 The maximum samples allowed for a single
                                 Series request, The Series call fails if
                                 this limit is exceeded. 0 means no limit.
                                 NOTE: For efficiency the limit is internally
                                 implemented as 'chunks limit' considering each
                                 chunk contains a maximum of 120 samples.
      --min-time=0000-01-01T00:00:00Z
                                 Start of time range limit to serve. Thanos
                                 sidecar will serve only metrics, which happened
                                 later than this value. Option can be a constant
                                 time in RFC3339 format or time duration
                                 relative to current time, such as -1d or 2h45m.
                                 Valid duration units are ms, s, m, h, d, w, y.

```

## Configuration

### Prometheus HTTP client

You can configure the Prometheus HTTP client for Thanos sidecar with YAML, either by passing the YAML content directly to the `--prometheus.http-client` flag, or by passing the YAML file path to the `--prometheus.http-client-file` flag.

The configuration format is the following:

```yaml
basic_auth:
  username: ""
  password: ""
  password_file: ""
bearer_token: ""
bearer_token_file: ""
proxy_url: ""
tls_config:
  ca_file: ""
  cert_file: ""
  key_file: ""
  server_name: ""
  insecure_skip_verify: false
transport_config:
  max_idle_conns: 0
  max_idle_conns_per_host: 0
  idle_conn_timeout: 0
  response_header_timeout: 0
  expect_continue_timeout: 0
  max_conns_per_host: 0
  disable_compression: false
  tls_handshake_timeout: 0
```
