CloudSQL to Honeycomb
=====================

This is a docker image, to be run in a k8s cluster, to ingest postgres logs from
[CloudSQL](https://cloud.google.com/sql/docs/postgres/) and send them off to
[honeycomb](https://www.honeycomb.io/).

## CloudSQL configuration
You'll want to turn on query logging. Assuming no custom database flags are set
on your cloudsql instance, you can run:
`gcloud sql instances patch <instance_name> --database-flags log_min_duration_statement=0`.
Caveat 1: this requires restarting your cloudsql instance, which gcloud will do for
you.

To see what flags are set: `gcloud sql instances describe <instance_name>` and
look for the top-level key `databaseFlags`. (You'll want to include existing
flags in your `gcloud sql instances patch` command, if any are set.)

## Build
`docker build -t postgres-honeytail .`

Push to your preferred docker registry.

## Ingestion
Ingestion is done via [Cloud Pub/Sub](https://cloud.google.com/pubsub/).

To use this, set up a Stackdriver sink of your postgres logs to a Pub/Sub topic:
[](https://cloud.google.com/logging/docs/export/configure_export_v2). You will
also need to create a
[subscription](https://cloud.google.com/pubsub/docs/subscriber) to that topic.

`logs.py` will subscribe, and receive messages. It will also pull a timestamp
from the pubsub message object, and add it to the postgres logline for
`honeytail`'s use. (Honeycomb's documentation recommends setting
`log_line_prefix`, but CloudSQL does not currently support that flag.)

`run.sh` takes the output of `logs.py` and sends it along to honeycomb via
`honeytail`.

## Config and deploy
We run this image as a single replica deployment in k8s.

### Required environment variables
- `GOOGLE_APPLICATION_CREDENTIALS_JSON` is used to auth to gcloud; the service
  account must have read access to the pubsub subscription.
- `PROJECT_ID` the gcloud project the pubsub subscription is in
- `SUBSCRIPTION_NAME` the pubsub subscription name
- `HONEYCOMB_WRITEKEY` (not required if `DEBUG` is set, see below)

### Optional environment variables
- `DEBUG` runs `honeytail` with the flags `--debug` (setting the log level to
  DEBUG) and `--debug_stdout` (writing events to stdout instead of sending to
  honeycomb).  The latter flag also means that `HONEYCOMB_WRITEKEY` is not
  verified, so it can be left unset.
- `DATASET` the honeycomb dataset to write to; defaults to `postgres`

## Testing
`docker run -t postgres-honeytail python test_logs.py`
