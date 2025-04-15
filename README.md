Kubernetes Downscaler
=====================

[![Travis CI Build Status](https://travis-ci.org/hjacobs/kube-downscaler.svg?branch=master)](https://travis-ci.org/hjacobs/kube-downscaler)
[![Code Coverage](https://coveralls.io/repos/github/hjacobs/kube-downscaler/badge.svg?branch=master;_=1)](https://coveralls.io/github/hjacobs/kube-downscaler?branch=master)
[![Docker pulls](https://img.shields.io/docker/pulls/hjacobs/kube-downscaler.svg)](https://hub.docker.com/r/hjacobs/kube-downscaler)
[![CalVer](https://img.shields.io/badge/calver-YY.MM.MICRO-22bfda.svg)](http://calver.org/)

Scale down / "pause" Kubernetes workload (`Deployments`, `StatefulSets`, and/or
`HorizontalPodAutoscalers` and `CronJobs` too !) during non-work hours.

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**  *generated with [DocToc](https://github.com/thlorenz/doctoc)*

- [Concepts](#concepts)
  - [Algorithm](#algorithm)
  - [Minimum replicas](#minimum-replicas)
  - [Specific workload](#specific-workload)
  - [Example use cases](#example-use-cases)
- [Usage](#usage)
  - [Deployment](#deployment)
  - [Example configuration](#example-configuration)
  - [Notes](#notes)
- [Configuration](#configuration)
  - [Uptime / downtime spec](#uptime--downtime-spec)
  - [Alternative logic, based on periods](#alternative-logic-based-on-periods)
  - [Command Line Options](#command-line-options)
  - [Namespace Defaults](#namespace-defaults)
- [Contributing](#contributing)
- [License](#license)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->


## Concepts

> :memo: `Deployments` are interchangeable by any kind of _supported workload_ for this whole guide unless explicitly stated otherwise.
>
> The complete list of supported workload is defined [here](./kube_downscaler/scaler.py#9-14).

### Algorithm

`Kube-downscaler` will scale down the deployment\'s replicas if all of the following
conditions are met:

- **current time** is not part of the \"uptime\" schedule or is part of the \"downtime\" schedule.

  If true, the schedules are evaluated in the following order:

    -   `downscaler/downscale-period` or `downscaler/downtime`
            annotation on the workload definition
    -   `downscaler/upscale-period` or `downscaler/uptime`
            annotation on the workload definition
    -   `downscaler/downscale-period` or `downscaler/downtime`
            annotation on the workload\'s namespace
    -   `downscaler/upscale-period` or `downscaler/uptime`
            annotation on the workload\'s namespace
    -   `--upscale-period` or `--default-uptime` CLI argument
    -   `--downscale-period` or `--default-downtime` CLI argument
    -   `UPSCALE_PERIOD` or `DEFAULT_UPTIME` environment variable
    -   `DOWNSCALE_PERIOD` or `DEFAULT_DOWNTIME` environment
            variable

- The workload\'s **namespace** is not part of the exclusion list:

    -   If you provide an exclusion list, it will be used in place
        of the default (which includes only `kube-system`).

- The workload\'s label does not match the labels list.

- The **workload\'s name** is not part of the exclusion list

- The workload is not marked for exclusion (annotation
    `downscaler/exclude: "true"` or
    `downscaler/exclude-until: "2024-04-05"`)

- There are no active pods that force the whole cluster into uptime
  (annotation `downscaler/force-uptime: "true"`)


### Minimum replicas

The deployment, by default, **will be scaled down to zero replicas**. This can
be configured with a deployment or its namespace\'s annotation of `downscaler/downtime-replicas`
or via CLI with `--downtime-replicas`.

Ex: `downscaler/downtime-replicas: "1"`


### Specific workload

In case of `HorizontalPodAutoscalers`, the `minReplicas` field cannot be set to zero and thus
`downscaler/downtime-replicas` should be at least `1`.

-> See later in [#Usage notes](#notes)


Regarding `CronJobs`, their state will be defined to `suspend: true` as you might expect.


### Example use cases

-   Deploy the downscaler to a test (non-prod) cluster with a default
    uptime or downtime time range to scale down all deployments during
    the night and weekend.
-   Deploy the downscaler to a production cluster without any default
    uptime/downtime setting and scale down specific deployments by
    setting the `downscaler/uptime` (or `downscaler/downtime`)
    annotation. This might be useful for internal tooling frontends
    which are only needed during work time.

You need to combine the downscaler with an elastic cluster autoscaler to
actually **save cloud costs**. The [official cluster
autoscaler](https://github.com/kubernetes/autoscaler/tree/master/cluster-autoscaler)
and the
[kube-aws-autoscaler](https://github.com/hjacobs/kube-aws-autoscaler)
were tested to work fine with the downscaler.


## Usage

### Deployment

Deploy the downscaler into your cluster via (also works with
[kind](https://kind.sigs.k8s.io/) or
[Minikube](https://github.com/kubernetes/minikube)):

``` {.sourceCode .bash}
$ kubectl apply -f deploy/
```

In case you are deploying `kube-downscaler` to another namespace than
`default`, for example if your context is pointing to `my-namespace`.
Make sure you change the `deploy/rbac.yaml` Service Account
configuration `namespace: default` to the destination namespace
`my-namespace`, instead of `default`.


### Example configuration

The example configuration uses the `--dry-run` as a safety flag to
prevent downscaling \-\-- remove it to enable the downscaler, e.g. by
editing the deployment:

``` {.sourceCode .bash}
$ kubectl edit deploy kube-downscaler
```

The example deployment manifests come with a configured uptime
(`deploy/config.yaml` sets it to \"Mon-Fri 07:30-20:30 CET\"), you can
overwrite this per namespace or deployment, e.g.:

``` {.sourceCode .bash}
$ kubectl run nginx --image=nginx
$ kubectl annotate deploy nginx 'downscaler/uptime=Mon-Fri 09:00-17:00 America/Buenos_Aires'
```


### Notes

Note that the _default grace period_ of 15 minutes applies to the new
nginx deployment, i.e.
* if the current time is not within `Mon-Fri 9-17 (Buenos Aires timezone)`,
  it will downscale not immediately, but after 15 minutes. The downscaler
  will eventually log something like:

```
    INFO: Scaling down Deployment default/nginx from 1 to 0 replicas (uptime: Mon-Fri 09:00-17:00 America/Buenos_Aires, downtime: never)
```

Note that in cases where a `HorizontalPodAutoscaler` (HPA) is used along
with Deployments, consider the following:

-   If downscale to 0 replicas is desired, the annotation should be
    applied on the `Deployment`. This is a special case, since
    `minReplicas` of 0 on HPA is not allowed. Setting Deployment
    replicas to 0 essentially disables the HPA. In such a case, the HPA
    will emit events like `failed to get memory utilization: unable to
    get metrics for resource memory: no metrics returned from resource
    metrics API` as there is no Pod to retrieve metrics from.
-   If downscale greater than 0 is desired, the annotation should be
    applied on the HPA. This allows for dynamic scaling of the Pods even
    during downtime based upon the external traffic as well as maintain
    a lower `minReplicas` during downtime if there is no/low traffic. **If
    the Deployment is annotated instead of the HPA, it leads to a race
    condition** where `kube-downscaler` scales down the Deployment and HPA
    upscales it as its `minReplicas` is higher.

To enable Downscaler on HPA with `--downtime-replicas=1`,
ensure to add the following annotations to Deployment and HPA.

``` {.sourceCode .bash}
$ kubectl annotate deploy nginx 'downscaler/exclude=true'
$ kubectl annotate hpa nginx 'downscaler/downtime-replicas=1'
$ kubectl annotate hpa nginx 'downscaler/uptime=Mon-Fri 09:00-17:00 America/Buenos_Aires'
```

## Configuration

### Uptime / downtime spec

The downscaler is configured via command line args, environment
variables and/or Kubernetes annotations.

Time definitions (e.g. `DEFAULT_UPTIME`) accept a comma separated list
of specifications, e.g. the following configuration would downscale all
deployments for non-work hours:

``` {.sourceCode .bash}
DEFAULT_UPTIME="Mon-Fri 07:30-20:30 Europe/Berlin"
```

To only downscale during the weekend and Friday after 20:00:

``` {.sourceCode .bash}
DEFAULT_DOWNTIME="Sat-Sun 00:00-24:00 CET,Fri-Fri 20:00-24:00 CET'
```

Each time specification can be in one of two formats:

-   Recurring specifications have the format
    `<WEEKDAY-FROM>-<WEEKDAY-TO-INCLUSIVE> <HH>:<MM>-<HH>:<MM> <TIMEZONE>`.
    The timezone value can be any [Olson
    timezone](https://en.wikipedia.org/wiki/Tz_database), e.g.
    \"US/Eastern\", \"PST\" or \"UTC\".
-   Absolute specifications have the format `<TIME_FROM>-<TIME_TO>`
    where each `<TIME>` is an ISO 8601 date and time of the format
    `<YYYY>-<MM>-<DD>T<HH>:<MM>:<SS>[+-]<TZHH>:<TZMM>`.


### Alternative logic, based on periods

Instead of strict uptimes or downtimes, you can chose time periods for
upscaling or downscaling. The time definitions are the same. In this
case, the upscale or downscale happens only on time periods, rest of
times will be ignored.

If upscale or downscale periods are configured, uptime and downtime will
be ignored. This means that some options are mutually exclusive, e.g.
you can either use `--downscale-period` or `--default-downtime`, but not
both.

This definition will downscale your cluster between 19:00 and 20:00. If
you upscale your cluster manually, it won\'t be scaled down until next
day 19:00-20:00.

``` {.sourceCode .bash}
DOWNSCALE_PERIOD="Mon-Sun 19:00-20:00 Europe/Berlin"
```


### Command Line Options

Available command line options:

`--dry-run`

:   Dry run mode: do not change anything, just print what would be done

`--debug`

:   Debug mode: print more information

`--once`

:   Run loop only once and exit

`--interval`

:   Loop interval (default: 30s)

`--namespace`

:   Restrict the downscaler to work only in a single namespace (default:
    all namespaces). This is mainly useful for deployment scenarios
    where the deployer of kube-downscaler only has access to a given
    namespace (instead of cluster access). If used simultaneously with
    `--exclude-namespaces`, none is applied.

`--include-resources`

:   Downscale resources of this kind as comma separated list.
    \[deployments, statefulsets, stacks, horizontalpodautoscalers, rollouts, scaledobjects\]
    (default: deployments)

`--grace-period`

:   Grace period in seconds for new deployments before scaling them down
    (default: 15min). The grace period counts from time of creation of
    the deployment, i.e. updated deployments will immediately be scaled
    down regardless of the grace period.

`--upscale-period`

:   Alternative logic to scale up only in given period of time (default:
    never), can also be configured via environment variable
    `UPSCALE_PERIOD` or via the annotation `downscaler/upscale-period`
    on each deployment

`--downscale-period`

:   Alternative logic to scale down only in given period of time
    (default: never), can also be configured via environment variable
    `DOWNSCALE_PERIOD` or via the annotation
    `downscaler/downscale-period` on each deployment

`--default-uptime`

:   Default time range to scale up for (default: always), can also be
    configured via environment variable `DEFAULT_UPTIME` or via the
    annotation `downscaler/uptime` on each deployment

`--default-downtime`

:   Default time range to scale down for (default: never), can also be
    configured via environment variable `DEFAULT_DOWNTIME` or via the
    annotation `downscaler/downtime` on each deployment

`--exclude-namespaces`

:   Exclude namespaces from downscaling (list of regex patterns,
    default: kube-system), can also be configured via environment
    variable `EXCLUDE_NAMESPACES`. If used simultaneously with
    `--namespace`, none is applied.

`--exclude-deployments`

:   Exclude specific deployments/statefulsets/cronjobs from downscaling
    (default: kube-downscaler, downscaler), can also be configured via
    environment variable `EXCLUDE_DEPLOYMENTS`. Despite its name, this
    option will match the name of any included resource type
    (Deployment, StatefulSet, CronJob, ..).

`--downtime-replicas`

:   Default value of replicas to downscale to, the annotation
    `downscaler/downtime-replicas` takes precedence over this value.

`--deployment-time-annotation`

:   Optional: name of the annotation that would be used instead of the
    creation timestamp of the resource. This option should be used if
    you want the resources to be kept scaled up during a grace period
    (`--grace-period`) after a deployment. The format of the
    annotation\'s timestamp value must be exactly the same as for
    Kubernetes\' `creationTimestamp`: `%Y-%m-%dT%H:%M:%SZ`. Recommended:
    set this annotation by your deployment tooling automatically.

`--matching-labels`

:   Optional: list of workload\'s labels which are covered by the kube-downscaler
    scope. All workloads whose labels don't match any in the list are ignored.
    For backwards compatibility, if this argument is not specified,
    kube-downscaler will apply to all resources.

### Namespace Defaults

`DEFAULT_UPTIME`, `DEFAULT_DOWNTIME`, `FORCE_UPTIME` and exclusion can
also be configured using Namespace annotations. **Where configured these
values supersede the other global default values**.

``` {.sourceCode .yaml}
apiVersion: v1
kind: Namespace
metadata:
    name: foo
    labels:
        name: foo
    annotations:
        downscaler/uptime: Mon-Sun 07:30-18:00 CET
```

The following annotations are supported on the Namespace level:

-   `downscaler/upscale-period`
-   `downscaler/downscale-period`
-   `downscaler/uptime`: set \"uptime\" for all resources in this
    namespace
-   `downscaler/downtime`: set \"downtime\" for all resources in this
    namespace
-   `downscaler/force-downtime`: force scaling down all resources in this
    namespace - can be `true`/`false` or a period
-   `downscaler/force-uptime`: force scaling up all resources in this
    namespace - can be `true`/`false` or a period
-   `downscaler/exclude`: set to `true` to exclude all resources in the
    namespace
-   `downscaler/exclude-until`: temporarily exclude all resources in the
    namespace until the given timestamp
-   `downscaler/downtime-replicas`: overwrite the default target
    replicas to scale down to (default: zero)


## Contributing

Easiest way to contribute is to provide feedback! We would love to hear
what you like and what you think is missing. Create an issue or [ping
try\_except\_ on Twitter](https://twitter.com/try_except_).

PRs are welcome.

## License

This program is free software: you can redistribute it and/or modify it
under the terms of the GNU General Public License as published by the
Free Software Foundation, either version 3 of the License, or (at your
option) any later version.

This program is distributed in the hope that it will be useful, but
WITHOUT ANY WARRANTY; without even the implied warranty of
MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See the GNU General
Public License for more details.

You should have received a copy of the GNU General Public License along
with this program. If not, see <http://www.gnu.org/licenses/>.
