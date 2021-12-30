# Scheduled Backups

This example shows how to use Cloud Scheduler and Cloud Function to configure
a schedule for creating Cloud Spanner backups.

## Run SpannerCreateBackup locally

Start a terminal for running a local server:

```bash
go run cmd/local_func_server/main.go
```

Start another terminal for calling the function:

```bash
DATA=$(printf '{"database":"projects/[PROJECT_ID]/instances/[INSTANCE_ID]/databases/[DATABASE_ID]", "expire": "6h"}'|base64|tr -d '\n') && curl --data '{"data":"'$DATA'"}' localhost:8080
```

## Run SpannerCreateBackup in Cloud

Create a pub/sub topic:

```bash
gcloud pubsub topics create cloud-spanner-scheduled-backups
```

Deploy the `SpannerCreateBackup` function that subscribes to the above topic:

```bash
gcloud functions deploy SpannerCreateBackup --trigger-topic cloud-spanner-scheduled-backups --runtime go113
```

Call the `SpannerCreateBackup` function from the command-line:

```bash
DATA=$(printf '{"database":"projects/[PROJECT_ID]/instances/[INSTANCE_ID]/databases/[DATABASE_ID]", "expire": "6h"}'|base64|tr -d '\n') && gcloud functions call SpannerCreateBackup --data '{"data":"'$DATA'"}'
```

## Deploy scheduled jobs to Cloud Scheduler

Note: To use Cloud Scheduler, we must [create an App Engine app](https://cloud.google.com/scheduler/docs#supported_regions). Cloud Scheduler must be deployed in the same region with App Engine. To specify a region, add a `location` flag to the `go run cmd/scheduler/main.go ...` command (default is set to `us-central1`).

Make a copy of `schedule-template.yaml`, name it as `schedule.config.yaml` and
replace `PROJECT_ID`, `INSTANCE_ID`, `DATABASE_ID` with your configurations.

Deploy scheduled jobs for creating backups:

```bash
go run cmd/scheduler/main.go -config schedule.config.yaml
```

## Email notification of backup failures

To get email notification, we need to do the following three steps.

### Set up email notification channel

We can follow this [guide](https://cloud.google.com/monitoring/support/notification-options#email)
to add our email address as a notification channel.

### Add logs-based metrics

We can add [logs-based metrics](https://cloud.google.com/logging/docs/logs-based-metrics/)
via GCP Console, API, gcloud, etc. Here, for convenience, we use
[Deployment Manager](https://cloud.google.com/deployment-manager/docs/quickstart)
to create custom metrics.

```bash
gcloud deployment-manager deployments create schedule-backup-metrics-deployment --config metrics.yaml
```

After this, we should see three user-defined metrics under `Logs-based Metrics`
in Cloud Logging.

### Create alerting policies

We need to create [alerting policies](https://cloud.google.com/monitoring/alerts)
that define when we should send an alert notification.

Cloud Monitoring API is still under alpha, so we would recommend using GCP
console to create the alerting policies.

The easiest way is to go to Logs-based Metrics under Cloud Logging and for each
user-defined metric, there is an option `Create alert from metric`. From there,
we can choose `Aggregrator`, such as `sum` or `mean`, for the target metric, and
define what the condition of triggering an alert is, e.g., any time series
violates that the value is above 0 for 1 minute.

At last, we need to add notification channels, e.g., email, to alerting
policies.
