# Brave Sync Server v2

A sync server implemented in go to communicate with Brave sync clients using
[components/sync/protocol/sync.proto](https://cs.chromium.org/chromium/src/components/sync/protocol/sync.proto).
Current Chromium version for sync protocol buffer files used in this repo is Chromium 116.0.5845.183.

This server supports endpoints as bellow.
- The `POST /v2/command/` endpoint handles Commit and GetUpdates requests from sync clients and return corresponding responses both in protobuf format. Detailed of requests and their corresponding responses are defined in `schema/protobuf/sync_pb/sync.proto`. Sync clients are responsible for generating valid access tokens and present them to the server in the Authorization header of requests.

Currently we use dynamoDB as the datastore, the schema could be found in `schema/dynamodb/table.json`.

# Hack

1. Run `make docker` to build two images, `brave-sync` and `brave-dynamo`
2. Run `make PUSH=docker.io/... push` to push the images to some docker registries

The example `docker-compose.yml` could be:

```yaml
services:
  brave-sync:
    image: brave-sync:latest
    user: 1000:1000
    ports:
      - 127.0.0.1:8295:8295
    depends_on:
      - brave-dynamo
      - redis
    environment:
      # After devices setup, you can remove the `ALLOW_NEW_REG` to keep some badass out:
      - ALLOW_NEW_REG=1
      - ENV=local
      - AWS_ACCESS_KEY_ID=GOSYNC
      - AWS_SECRET_ACCESS_KEY=GOSYNC
      - AWS_REGION=us-west-2
      - AWS_ENDPOINT=http://brave-dynamo:8000
      - TABLE_NAME=client-entity-dev
      - REDIS_URL=redis:6379
    restart: unless-stopped

  brave-dynamo:
    image: brave-dynamo:latest
    user: 1000:1000
    volumes:
      - /where/to/store/the/data:/db
    restart: unless-stopped

  redis:
    image: redis:6.2
    user: 1000:1000
    environment:
      - ALLOW_EMPTY_PASSWORD=yes
    volumes:
      - /where/to/store/the/redis:/data
    restart: unless-stopped
```

For most up-to-date compose file, please visit the [`docker-compose.yml`](https://github.com/z1gc/brave-sync/blob/master/docker-compose.yml) :)

* A environment `ALLOW_NEW_REG` is added, a new client (which have no devices connected) is only allowed if it's set to `1`
* You can push the images, due to some network restrictions, I'm not going to publish them via docker.io

## Developer Setup
1. [Install Go 1.22](https://golang.org/doc/install)
2. [Install GolangCI-Lint](https://github.com/golangci/golangci-lint#install)
3. [Install gowrap](https://github.com/hexdigest/gowrap#installation)
4. Clone this repo
5. [Install protobuf protocol compiler](https://github.com/protocolbuffers/protobuf#protocol-compiler-installation) if you need to compile protobuf files, which could be built using `make protobuf`.
6. Build via `make`

## Local development using Docker and DynamoDB Local
1. Clone this repo
2. Run `make docker`
3. Run `make docker-up`
4. To connect to your local server from a Brave browser client use `--sync-url="http://localhost:8295/v2"`
5. For running unit tests, run `make docker-test`

# Updating protocol definitions
1. Copy the `.proto` files from `components/sync/protocol` in `chromium` to `schema/protobuf/sync_pb` in `go-sync`.
2. Copy the `.proto` files from `components/sync/protocol` in `brave-core` to `schema/protobuf/sync_pb` in `go-sync`.
3. Run `make repath-proto` to set correct import paths in `.proto` files.
4. Run `make proto-go-module` to add the `go_module` option to `.proto` files.
5. Run `make protobuf` to generate the Go code from `.proto` definitions.

## Prometheus Instrumentation
The instrumented datastore and redis interfaces are generated, providing integration with Prometheus.  The following will re-generate the instrumented code, required when updating protocl definitions:

```
make instrumented
```

Changes to `datastore/datastore.go` or `cache/cache.go` should be followed with the above command.
