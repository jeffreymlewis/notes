# Overview
Alert when a kuberntes [CronJob](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/) fails by monitoring failure of the underlying [Jobs](https://kubernetes.io/docs/concepts/workloads/controllers/job/).

# Requirements
This solution requires that [kube-state-metrics](https://github.com/kubernetes/kube-state-metrics) be installed in your Kubernetes cluster. It also requires a label named `cronjob_name` is added to `.spec.jobTemplate.metadata.labels` in each CronJob. (I recommend adding the `cronjob_name` label to `.spec.jobTemplate.spec.template.metadata.labels` as well, but this is not required.)

# Prometheus Monitor for Kubernetes CronJobs
I'll start by showing the entire query, then breakdown the different parts.

## Number of failures for a CronJob's most recent Job
This PromQL query will output the number of times a CronJob's most recent Job has failed. A CronJob is defined as a Kubernetes Job containing the label `cronjob_name`. If no jobs are failing, the output will be empty.
```
clamp_max(
  MAX BY (label_cronjob_name, namespace) (
    kube_job_status_start_time
    * ON(job_name, namespace) GROUP_LEFT(label_cronjob_name)
    kube_job_labels{label_cronjob_name!=""}
  )
  == ON(label_cronjob_name, namespace) GROUP_RIGHT()
  MAX BY (job_name, label_cronjob_name, namespace) (
    kube_job_status_start_time
    * ON(job_name, namespace) GROUP_LEFT(label_cronjob_name)
    kube_job_labels{label_cronjob_name!=""}
  ),
  1
)
* ON(job_name, namespace) GROUP_LEFT()
kube_job_status_failed > 0
```

## recent Job start time for each CronJob
This query returns the latest job start time for each cronjob. Unfortunately the [CronJob metrics](https://github.com/kubernetes/kube-state-metrics/blob/master/docs/cronjob-metrics.md) don't include status information, so we need to go deeper and look at the [Job metrics](https://github.com/kubernetes/kube-state-metrics/blob/master/docs/job-metrics.md) instead.
```
  MAX BY (label_cronjob_name, namespace) (
    kube_job_status_start_time
    * ON(job_name, namespace) GROUP_LEFT(label_cronjob_name)
    kube_job_labels{label_cronjob_name!=""}
  )
```

## Jobs start time (if they have the 'cronjob_name' label)
This query returns the start time of all Jobs associated with a kubernetes CronJob (aka, all Jobs containing the `cronjob_name` label). We use the `MAX BY` aggregation so we can return only the labels of interest (job_name, label_cronjob_name, namespace).
```
  MAX BY (job_name, label_cronjob_name, namespace) (
    kube_job_status_start_time
    * ON(job_name, namespace) GROUP_LEFT(label_cronjob_name)
    kube_job_labels{label_cronjob_name!=""}
  )
```

## Most recent Job for each CronJob
We then use the equality operator to find the most recent Job for each CronJob.
```
  MAX BY (label_cronjob_name, namespace) (
    kube_job_status_start_time
    * ON(job_name, namespace) GROUP_LEFT(label_cronjob_name)
    kube_job_labels{label_cronjob_name!=""}
  )
  == ON(label_cronjob_name, namespace) GROUP_RIGHT()
  MAX BY (job_name, label_cronjob_name, namespace) (
    kube_job_status_start_time
    * ON(job_name, namespace) GROUP_LEFT(label_cronjob_name)
    kube_job_labels{label_cronjob_name!=""}
  )
```

## clamp_max()
The `kube_job_status_start_time` metric gives us Job start time in seconds since epoch. We do not actually care about the start time, and only need the labels from the most recent Job so we can compare against other metrics. We use [clamp_max()](https://prometheus.io/docs/prometheus/latest/querying/functions/#clamp_max) to change this metric to '1', allowing us to join with other metrics (like `kube_job_status_failed`) using multiplication.

## Number of failed jobs
Now that we have 'job_name', 'label_cronjob_name', and 'namespace' for the most recent job, we can join against `kube_job_status_failed` to find the number of times the given Job has failed, and alert if this metric is > 0. (This is the full query shown above.)
