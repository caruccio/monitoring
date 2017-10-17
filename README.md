Prepare some env vars t use on templates.

```
export ROUTER_BASIC_USERNAME=`oc env --list dc/router -n default |grep STATS_USERNAME  | cut -f 2 -d=`
export ROUTER_BASIC_PASSWORD=`oc env --list dc/router -n default |grep STATS_PASSWORD  | cut -f 2 -d=`
export PROXY_ADMIN_HTPASSWD_HASH=`htpasswd -n -i admin <<<admin | grep .`
export CLUSTER_NAME=$YOUR_CLUSTER_NAME
export APPS_DOMAIN=$YOU_APPS_DOMAIN

export PAGERDUTY_GENERIC_API_KEY=$YOUR_NEWRELIC_KEY
export SLACK_HOOK_URL=$YOUR_SLACK_HOOK_URL
export SLACK_CHANNEL='#${CHANNEL_NAME}'

export AWS_ACCESS_KEY_ID=$YOUR_AWS_KEY
export AWS_SECRET_ACCESS_KEY=$YOUR_AWS_SECRET

export NS=monitoring
```

Create a project in order to isolate the components. Change `--node-selector` for your needs.

```
oadm new-project $NS --node-selector 'role=app'
```

Setup security policies.

```
oc -n $NS adm policy add-cluster-role-to-user eventrouter   -z eventrouter
oc -n $NS adm policy add-cluster-role-to-user cluster-admin -z prometheus
oc -n $NS adm policy add-scc-to-user          privileged    -z prometheus-node-exporter
```

Create config files

```
oc process -o yaml -f configs/alertmanager-config-template.yaml \
    -p PAGERDUTY_GENERIC_API_KEY=$PAGERDUTY_GENERIC_API_KEY \
    -p SLACK_HOOK_URL=$SLACK_HOOK_URL \
    -p SLACK_CHANNEL=$SLACK_CHANNEL \
    | oc create -f -

oc process -o yaml -f configs/prometheus-config-template.yaml \
    -p CLUSTER_NAME=$CLUSTER_NAME \
    -p ROUTER_BASIC_USERNAME=$ROUTER_BASIC_USERNAME \
    -p ROUTER_BASIC_PASSWORD=$ROUTER_BASIC_PASSWORD \
    | oc create -f -

oc process -o yaml -f configs/prometheus-config-template.yaml \
    -p 'PROXY_ADMIN_HTPASSWD_HASH=$PROXY_ADMIN_HTPASSWD_HASH' \
    | oc create -f -

oc process -o yaml -f configs/grafana-config-template.yaml \
    | oc create -f -

oc create -f configs/cloudwatch-exporter-config.yaml
oc create -f configs/eventrouter-exporter-config.yaml
```

Deploy stuff

```
oc -n $NS create -f node-exporter-deployment-template.yaml
oc -n $NS create -f kubestate-exporter-deployment-template.yaml
oc -n $NS create -f cloudwatch-exporter-deployment-template.yaml
oc -n $NS create -f eventrouter-deployment-template.yaml
oc -n $NS create -f alertmanager-deployment-template.yaml
oc -n $NS create -f prometheus-deployment-template.yaml
oc -n $NS create -f grafana-deployment-template.yaml
oc -n $NS create -f proxy-deployment-template.yaml
```

#oc -n $NS expose dc/kubestate-exporter
#oc -n $NS expose dc/cloudwatch-exporter
#oc -n $NS expose dc/eventrouter
#oc -n $NS expose dc/alertmanager
#oc -n $NS expose dc/prometheus
#oc -n $NS expose dc/grafana
#oc -n $NS expose dc/proxy --name=proxy

#oc annotate svc/kubestate-exporter prometheus.io/scrape=true -n $NS
#oc annotate svc/eventrouter prometheus.io/scrape=true -n $NS

#oc -n $NS expose svc/proxy --name=proxy-grafana --hostname=grafana.${APPS_DOMAIN}
#oc -n $NS expose svc/proxy --name=proxy-prometheus --hostname=prometheus.${APPS_DOMAIN}


### prometheus-config-template.yaml

```
apiVersion: v1
kind: Template
metadata:
  name: prometheus-config-template

parameters:
- name: CLUSTER_NAME
  required: true
- name: ROUTER_BASIC_USERNAME
  value: admin
  required: true
- name: ROUTER_BASIC_PASSWORD
  required: true

objects:
- apiVersion: v1
  kind: ConfigMap
  metadata:
    name: prom-k8s
  data:
    docker.rules: |
      ALERT DockerHasErrors
        IF irate(kubelet_docker_operations_errors{operation_type!~"remove_container|start_container|inspect_container|inspect_image|inspect_exec|operation_type||logs"}[10m])  > 0
        FOR 5m
        LABELS {
          cluster = "${CLUSTER_NAME}",
          service = "docker",
          severity = "critical",
        }
        ANNOTATIONS {
          summary = "Docker has errors",
          description = "Docker on instance {{ $labels.instance }} has errors with {{ $labels.operation_type}}",
        }

      ALERT DockerHasTimeout
        IF irate(kubelet_docker_operations_timeout[5m]) > 0
        FOR 1m
        LABELS {
          cluster = "${CLUSTER_NAME}",
          service = "docker",
          severity = "critical",
        }
        ANNOTATIONS {
          summary = "Docker Timeout",
          description = "Docker on {{ $labels.instance }} has timeouts",
        }

    kubernetes.rules: |
      # NOTE: These rules were kindly contributed by the SoundCloud engineering team.
      ### Container resources ###
      cluster_namespace_controller_pod_container:spec_memory_limit_bytes =
        sum by (cluster,namespace,controller,pod_name,container_name) (
          label_replace(
            container_spec_memory_limit_bytes{container_name!=""},
            "controller", "$1",
            "pod_name", "^(.*)-[a-z0-9]+"
          )
        )

      cluster_namespace_controller_pod_container:spec_cpu_shares =
        sum by (cluster,namespace,controller,pod_name,container_name) (
          label_replace(
            container_spec_cpu_shares{container_name!=""},
            "controller", "$1",
            "pod_name", "^(.*)-[a-z0-9]+"
          )
        )

      cluster_namespace_controller_pod_container:cpu_usage:rate =
        sum by (cluster,namespace,controller,pod_name,container_name) (
          label_replace(
            irate(
              container_cpu_usage_seconds_total{container_name!=""}[5m]
            ),
            "controller", "$1",
            "pod_name", "^(.*)-[a-z0-9]+"
          )
        )

      cluster_namespace_controller_pod_container:memory_usage:bytes =
        sum by (cluster,namespace,controller,pod_name,container_name) (
          label_replace(
            container_memory_usage_bytes{container_name!=""},
            "controller", "$1",
            "pod_name", "^(.*)-[a-z0-9]+"
          )
        )

      cluster_namespace_controller_pod_container:memory_working_set:bytes =
        sum by (cluster,namespace,controller,pod_name,container_name) (
          label_replace(
            container_memory_working_set_bytes{container_name!=""},
            "controller", "$1",
            "pod_name", "^(.*)-[a-z0-9]+"
          )
        )

      cluster_namespace_controller_pod_container:memory_rss:bytes =
        sum by (cluster,namespace,controller,pod_name,container_name) (
          label_replace(
            container_memory_rss{container_name!=""},
            "controller", "$1",
            "pod_name", "^(.*)-[a-z0-9]+"
          )
        )

      cluster_namespace_controller_pod_container:memory_cache:bytes =
        sum by (cluster,namespace,controller,pod_name,container_name) (
          label_replace(
            container_memory_cache{container_name!=""},
            "controller", "$1",
            "pod_name", "^(.*)-[a-z0-9]+"
          )
        )

      cluster_namespace_controller_pod_container:disk_usage:bytes =
        sum by (cluster,namespace,controller,pod_name,container_name) (
          label_replace(
            container_disk_usage_bytes{container_name!=""},
            "controller", "$1",
            "pod_name", "^(.*)-[a-z0-9]+"
          )
        )

      cluster_namespace_controller_pod_container:memory_pagefaults:rate =
        sum by (cluster,namespace,controller,pod_name,container_name,scope,type) (
          label_replace(
            irate(
              container_memory_failures_total{container_name!=""}[5m]
            ),
            "controller", "$1",
            "pod_name", "^(.*)-[a-z0-9]+"
          )
        )

      cluster_namespace_controller_pod_container:memory_oom:rate =
        sum by (cluster,namespace,controller,pod_name,container_name,scope,type) (
          label_replace(
            irate(
              container_memory_failcnt{container_name!=""}[5m]
            ),
            "controller", "$1",
            "pod_name", "^(.*)-[a-z0-9]+"
          )
        )

      ### Cluster resources ###
      cluster:memory_allocation:percent =
        100 * sum by (cluster) (
          container_spec_memory_limit_bytes{pod_name!=""}
        ) / sum by (cluster) (
          machine_memory_bytes
        )

      cluster:memory_used:percent =
        100 * sum by (cluster) (
          container_memory_usage_bytes{pod_name!=""}
        ) / sum by (cluster) (
          machine_memory_bytes
        )

      cluster:cpu_allocation:percent =
        100 * sum by (cluster) (
          container_spec_cpu_shares{pod_name!=""}
        ) / sum by (cluster) (
          container_spec_cpu_shares{id="/"} * on(cluster,instance) machine_cpu_cores
        )

      cluster:node_cpu_use:percent =
        100 * sum by (cluster) (
          rate(node_cpu{mode!="idle"}[5m])
        ) / sum by (cluster) (
          machine_cpu_cores
        )

      ### API latency ###
      # Raw metrics are in microseconds. Convert to seconds.
      cluster_resource_verb:apiserver_latency:quantile_seconds{quantile="0.99"} =
        histogram_quantile(
          0.99,
          sum by(le,cluster,job,resource,verb) (apiserver_request_latencies_bucket)
        ) / 1e6
      cluster_resource_verb:apiserver_latency:quantile_seconds{quantile="0.9"} =
        histogram_quantile(
          0.9,
          sum by(le,cluster,job,resource,verb) (apiserver_request_latencies_bucket)
        ) / 1e6
      cluster_resource_verb:apiserver_latency:quantile_seconds{quantile="0.5"} =
        histogram_quantile(
          0.5,
          sum by(le,cluster,job,resource,verb) (apiserver_request_latencies_bucket)
        ) / 1e6

      ### Scheduling latency ###
      cluster:scheduler_e2e_scheduling_latency:quantile_seconds{quantile="0.99"} =
        histogram_quantile(0.99,sum by (le,cluster) (scheduler_e2e_scheduling_latency_microseconds_bucket)) / 1e6
      cluster:scheduler_e2e_scheduling_latency:quantile_seconds{quantile="0.9"} =
        histogram_quantile(0.9,sum by (le,cluster) (scheduler_e2e_scheduling_latency_microseconds_bucket)) / 1e6
      cluster:scheduler_e2e_scheduling_latency:quantile_seconds{quantile="0.5"} =
        histogram_quantile(0.5,sum by (le,cluster) (scheduler_e2e_scheduling_latency_microseconds_bucket)) / 1e6

      cluster:scheduler_scheduling_algorithm_latency:quantile_seconds{quantile="0.99"} =
        histogram_quantile(0.99,sum by (le,cluster) (scheduler_scheduling_algorithm_latency_microseconds_bucket)) / 1e6
      cluster:scheduler_scheduling_algorithm_latency:quantile_seconds{quantile="0.9"} =
        histogram_quantile(0.9,sum by (le,cluster) (scheduler_scheduling_algorithm_latency_microseconds_bucket)) / 1e6
      cluster:scheduler_scheduling_algorithm_latency:quantile_seconds{quantile="0.5"} =
        histogram_quantile(0.5,sum by (le,cluster) (scheduler_scheduling_algorithm_latency_microseconds_bucket)) / 1e6

      cluster:scheduler_binding_latency:quantile_seconds{quantile="0.99"} =
        histogram_quantile(0.99,sum by (le,cluster) (scheduler_binding_latency_microseconds_bucket)) / 1e6
      cluster:scheduler_binding_latency:quantile_seconds{quantile="0.9"} =
        histogram_quantile(0.9,sum by (le,cluster) (scheduler_binding_latency_microseconds_bucket)) / 1e6
      cluster:scheduler_binding_latency:quantile_seconds{quantile="0.5"} =
        histogram_quantile(0.5,sum by (le,cluster) (scheduler_binding_latency_microseconds_bucket)) / 1e6

      ALERT K8SFailedSchedulingErrorTooHigh
        IF sum(rate(heptio_eventrouter_warnings_total{reason="FailedScheduling"}[5m]) > 0.1
        FOR 5m
        LABELS {
          cluster = "${CLUSTER_NAME}",
          service = "k8s",
          severity = "warning"
        }
        ANNOTATIONS {
          summary = "Too many scheduling failures",
          description = "Kubernetes is failing to schedule pods. Please check nodes readiness",
        }

      ALERT K8SNodeDown
        IF up{job="kubernetes-nodes"} == 0
        FOR 1h
        LABELS {
          cluster = "${CLUSTER_NAME}",
          service = "k8s",
          severity = "warning"
        }
        ANNOTATIONS {
          summary = "Kubelet cannot be scraped",
          description = "Prometheus could not scrape a {{ $labels.job }} for more than one hour",
        }

      ALERT K8SNodeNotReady
        IF sum(kube_node_status_condition{condition="Ready", status=~"(false|unknown)"}) by (instance) > 0
        FOR 1m
        LABELS {
          cluster = "${CLUSTER_NAME}",
          service = "k8s",
          severity = "warning",
        }
        ANNOTATIONS {
          summary = "Node status is NotReady",
          description = "The Kubelet on {{ $labels.node }} has not checked in with the API, or has set itself to NotReady, for more than 2 minutes",
        }

      ALERT K8SKubeletNodeExporterDown
        IF up{job="node-exporter"} == 0
        FOR 15m
        LABELS {
          cluster = "${CLUSTER_NAME}",
          service = "k8s",
          severity = "warning"
        }
        ANNOTATIONS {
          summary = "Kubelet node_exporter cannot be scraped",
          description = "Prometheus could not scrape a {{ $labels.job }} for more than one hour.",
        }

      ALERT K8SKubeletDown
        IF absent(up{job="kubernetes-nodes"}) or count by (cluster) (up{job="kubernetes-nodes"} == 0) / count by (cluster) (up{job="kubernetes-nodes"}) > 0.1
        FOR 1h
        LABELS {
          cluster = "${CLUSTER_NAME}",
          service = "k8s",
          severity = "critical"
        }
        ANNOTATIONS {
          summary = "Many Kubelets cannot be scraped",
          description = "Prometheus failed to scrape more than 10% of kubelets, or all Kubelets have disappeared from service discovery.",
        }

      ALERT K8SApiserverDown
        IF up{job="kubernetes-apiservers"} == 0
        FOR 15m
        LABELS {
          cluster = "${CLUSTER_NAME}",
          service = "k8s",
          severity = "warning"
        }
        ANNOTATIONS {
          summary = "API server unreachable",
          description = "An API server could not be scraped.",
        }

      # Disable for non HA kubernetes setups.
      ALERT K8SApiserverDown
        IF absent({job="kubernetes-apiservers"}) or (count by(cluster) (up{job="kubernetes-apiservers"} == 1) < count by(cluster) (up{job="kubernetes-apiservers"}))
        FOR 5m
        LABELS {
          cluster = "${CLUSTER_NAME}",
          service = "k8s",
          severity = "critical"
        }
        ANNOTATIONS {
          summary = "API server unreachable",
          description = "Prometheus failed to scrape multiple API servers, or all API servers have disappeared from service discovery.",
        }

      ALERT K8SConntrackTableFull
        IF 100*node_nf_conntrack_entries / node_nf_conntrack_entries_limit > 50
        FOR 10m
        LABELS {
          cluster = "${CLUSTER_NAME}",
          service = "k8s",
          severity = "warning"
        }
        ANNOTATIONS {
          summary = "Number of tracked connections is near the limit",
          description = "The nf_conntrack table is {{ $value }}% full.",
        }

      ALERT K8SConntrackTableFull
        IF 100*node_nf_conntrack_entries / node_nf_conntrack_entries_limit > 90
        LABELS {
          cluster = "${CLUSTER_NAME}",
          service = "k8s",
          severity = "critical"
        }
        ANNOTATIONS {
          summary = "Number of tracked connections is near the limit",
          description = "The nf_conntrack table is {{ $value }}% full.",
        }

      # To catch the conntrack sysctl de-tuning when it happens
      ALERT K8SConntrackTuningMissing
        IF node_nf_conntrack_udp_timeout > 10
        FOR 10m
        LABELS {
          cluster = "${CLUSTER_NAME}",
          service = "k8s",
          severity = "warning",
        }
        ANNOTATIONS {
          summary = "Node does not have the correct conntrack tunings",
          description = "Nodes keep un-setting the correct tunings, investigate when it happens.",
        }

      ALERT K8STooManyOpenFiles
        IF 100*process_open_fds{job=~"kubelet|kubernetes"} / process_max_fds > 50
        FOR 10m
        LABELS {
          cluster = "${CLUSTER_NAME}",
          service = "k8s",
          severity = "warning"
        }
        ANNOTATIONS {
          summary = "{{ $labels.job }} has too many open file descriptors",
          description = "{{ $labels.node }} is using {{ $value }}% of the available file/socket descriptors.",
        }

      ALERT K8STooManyOpenFiles
        IF 100*process_open_fds{job=~"kubelet|kubernetes"} / process_max_fds > 80
        FOR 10m
        LABELS {
          cluster = "${CLUSTER_NAME}",
          service = "k8s",
          severity = "critical"
        }
        ANNOTATIONS {
          summary = "{{ $labels.job }} has too many open file descriptors",
          description = "{{ $labels.node }} is using {{ $value }}% of the available file/socket descriptors.",
        }

      # Some verbs excluded because they are expected to be long-lasting:
      # WATCHLIST is long-poll, CONNECT is `kubectl exec`.
      ALERT K8SApiServerLatency
        IF histogram_quantile(
            0.99,
            sum without (instance,node,resource) (apiserver_request_latencies_bucket{verb=~"POST|GET|DELETE|PATCH"})
          ) / 1e6 > 2.0
        FOR 10m
        LABELS {
          cluster = "${CLUSTER_NAME}",
          service = "k8s",
          severity = "warning"
        }
        ANNOTATIONS {
          summary = "Kubernetes apiserver latency is high",
          description = "99th percentile Latency for {{ $labels.verb }} requests to the kube-apiserver is higher than 2s.",
        }

      ALERT K8SApiServerEtcdAccessLatency
        IF etcd_request_latencies_summary{quantile="0.99"} / 1e6 > 1.0
        FOR 15m
        LABELS {
          cluster = "${CLUSTER_NAME}",
          service = "k8s",
          severity = "warning"
        }
        ANNOTATIONS {
          summary = "Access to etcd is slow",
          description = "99th percentile latency for apiserver to access etcd is higher than 1s.",
        }

      ALERT K8SKubeletTooManyPods
        IF sum(label_replace(kubelet_running_pod_count, "node", "$1", "instance", "(.*)") * on(node) group_right() kube_node_labels) by(node) > sum(kube_node_status_allocatable_pods) by(node)
        LABELS {
          cluster = "${CLUSTER_NAME}",
          service = "k8s",
          severity = "warning",
        }
        ANNOTATIONS {
          summary = "Kubelet is close to pod limit",
          description = "Kubelet {{$labels.instance}} is running {{$value}} pods",
        }

    node.rules: |
      ALERT MasterCPUUsage
        IF ((1 - (avg(irate(node_cpu{mode="idle", role="master"}[5m])) BY (instance))) * 100) > 85
        FOR 10m
        LABELS {
          cluster = "${CLUSTER_NAME}",
          service = "infra",
          severity="critical"
        }
        ANNOTATIONS {
          summary = "High CPU usage detected",
          description = "{{$labels.instance}} ({{$labels.label_type}}): CPU usage is above 85% (current value is: {{ $value }})"
        }
      ALERT MasterLoadAverage
        IF node_load15{role="master"} / ON(server_name) group_right() machine_cpu_cores{role="master"} > 2
        FOR 10m
        LABELS {
          cluster = "${CLUSTER_NAME}",
          service = "infra",
          severity="critical"
        }
        ANNOTATIONS {
          summary = "High load average detected",
          description = "{{$labels.instance}} ({{$labels.label_type}}): load average (15) is high"
        }
      ALERT MasterMemoryUsage
        IF ((node_memory_MemTotal{role="master"} - node_memory_MemAvailable{role="master"}) / (node_memory_MemTotal{role="master"}) * 100) > 85
        FOR 10m
        LABELS {
          cluster = "${CLUSTER_NAME}",
          service = "infra",
          severity="critical"
        }
        ANNOTATIONS {
          summary = "High memory usage detected",
          description = "{{$labels.instance}} ({{$labels.label_type}}): Memory usage is above 85% (current value is: {{ $value }})"
        }

      ALERT InfraCPUUsage
        IF ((1 - (avg(irate(node_cpu{mode="idle", role="infra"}[5m])) BY (instance))) * 100) > 85
        FOR 10m
        LABELS {
          cluster = "${CLUSTER_NAME}",
          service = "infra",
          severity="critical"
        }
        ANNOTATIONS {
          summary = "High CPU usage detected",
          description = "{{$labels.instance}} ({{$labels.label_type}}): CPU usage is above 85% (current value is: {{ $value }})"
        }
      ALERT InfraLoadAverage
        IF node_load15{role="infra"} / ON(server_name) group_right() machine_cpu_cores{role="infra"} > 2
        FOR 10m
        LABELS {
          cluster = "${CLUSTER_NAME}",
          service = "infra",
          severity="critical"
        }
        ANNOTATIONS {
          summary = "High load average detected",
          description = "{{$labels.instance}} ({{$labels.label_type}}): load average (15) is high"
        }
      ALERT InfraMemoryUsage
        IF ((node_memory_MemTotal{role="infra"} - node_memory_MemAvailable{role="infra"}) / (node_memory_MemTotal{role="infra"}) * 100) > 85
        FOR 10m
        LABELS {
          cluster = "${CLUSTER_NAME}",
          service = "infra",
          severity="critical"
        }
        ANNOTATIONS {
          summary = "High memory usage detected",
          description = "{{$labels.instance}} ({{$labels.label_type}}): Memory usage is above 85% (current value is: {{ $value }})"
        }

      ALERT NodeCPUUsage
        IF ((1 - (avg(irate(node_cpu{mode="idle", role="app"}[5m])) BY (instance))) * 100) > 85
        FOR 10m
        LABELS {
          cluster = "${CLUSTER_NAME}",
          service = "infra",
          severity="critical"
        }
        ANNOTATIONS {
          summary = "High CPU usage detected",
          description = "{{$labels.instance}} ({{$labels.label_type}}): CPU usage is above 85% (current value is: {{ $value }})"
        }
      ALERT NodeLoadAverage
        IF node_load15{role="app"} / ON(server_name) group_right() machine_cpu_cores{role="app"} > 2
        FOR 10m
        LABELS {
          cluster = "${CLUSTER_NAME}",
          service = "infra",
          severity="critical"
        }
        ANNOTATIONS {
          summary = "High load average detected",
          description = "{{$labels.instance}} ({{$labels.label_type}}): load average (15) is high"
        }
      ALERT NodeMemoryUsage
        IF ((node_memory_MemTotal{role="app"} - node_memory_MemAvailable{role="app"}) / (node_memory_MemTotal{role="app"}) * 100) > 85
        FOR 10m
        LABELS {
          cluster = "${CLUSTER_NAME}",
          service = "infra",
          severity="critical"
        }
        ANNOTATIONS {
          summary = "High memory usage detected",
          description = "{{$labels.instance}} ({{$labels.label_type}}): Memory usage is above 85% (current value is: {{ $value }})"
        }

      #ALERT NodeLowRootDisk
      #  IF (label_replace(max(((node_filesystem_size{fstype="rootfs"} - node_filesystem_free{fstype="rootfs"} ) / node_filesystem_size{fstype="rootfs"} * 100)) by(instance), "node", "$1", "instance", "(.*)") * on(node) group_right() kube_node_labels) > 80
      #  FOR 10m
      #  LABELS {
      #    severity="critical"
      #  }
      #  ANNOTATIONS {
      #    summary = "Low root disk space",
      #    description = "{{$labels.instance}} ({{$labels.label_type}}): Root disk usage is above 80% (current value is: {{ $value }})"
      #  }
      #
      ## Docker Disk (80% full)
      #ALERT NodeLowDockerDisk
      #  IF (label_replace(max(((node_filesystem_size{device="rootfs"} - node_filesystem_free{device="rootfs"} ) / node_filesystem_size{device="rootfs"} * 100)) by(instance), "node", "$1", "instance", "(.*)") * on(node) group_right() kube_node_labels) > 80
      #  FOR 10m
      #  LABELS {
      #    severity="critical"
      #  }
      #  ANNOTATIONS {
      #    summary = "Low data disk space",
      #    description = "{{$labels.instance}} ({{$labels.label_type}}): Data disk usage is above 80% (current value is: {{ $value }})"
      #  }
      #
      ## OpenShift Disk (80% full)
      #ALERT NodeLowDataDisk
      #  IF (label_replace(max(((node_filesystem_size{device="/dev/xvdc"} - node_filesystem_free{device="/dev/xvdc"} ) / node_filesystem_size{device="/dev/xvdc"} * 100)) by(instance), "node", "$1", "instance", "(.*)") * on(node) group_right() kube_node_labels) > 80
      #  FOR 10m
      #  LABELS {
      #    severity="critical"
      #  }
      #  ANNOTATIONS {
      #    summary = "Low data disk space",
      #    description = "{{$labels.instance}} ({{$labels.label_type}}): Data disk usage is above 80% (current value is: {{ $value }})"
      #  }


    prometheus.yml: |
      global:
        scrape_interval: 30s
        scrape_timeout: 30s

      rule_files:
      - /etc/prometheus/*.rules

      scrape_configs:
        - job_name: prometheus
          static_configs:
            - targets: ['localhost:9090']
          relabel_configs:
          - source_labels: [__address__]
            action: replace
            regex: .*
            target_label: role
            replacement: prom

        - job_name: aws
          static_configs:
            - targets: ['cloudwatch-exporter:9106']
          relabel_configs:
          - source_labels: [__address__]
            action: replace
            regex: .*
            target_label: role
            replacement: ec2

        #- job_name: etcd
        #  scheme: https
        #  tls_config:
        #    insecure_skip_verify: true
        #  static_configs:
        #    - targets:
        #      - 'ip-10-0-1-129.ec2.internal:2379'
        #      - 'ip-10-0-2-63.ec2.internal:2379'
        #      - 'ip-10-0-3-166.ec2.internal:2379'

        - job_name: kubernetes-nodes
          scheme: https
          tls_config:
            ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
          bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token

          kubernetes_sd_configs:
          - role: node

          relabel_configs:
          - action: labelmap
            regex: __meta_kubernetes_node_label_(.+)
          - target_label: __address__
            replacement: kubernetes.default.svc:443
          - source_labels: [__meta_kubernetes_node_name]
            regex: (.+)
            target_label: __metrics_path__
            replacement: /api/v1/nodes/${1}/proxy/metrics

        - job_name: kubernetes-apiservers
          scheme: https
          tls_config:
            ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
          bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token

          kubernetes_sd_configs:
          - role: endpoints

          relabel_configs:
          - source_labels: [__meta_kubernetes_namespace, __meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]
            action: keep
            regex: default;kubernetes;https

        - job_name: kubernetes-services
          kubernetes_sd_configs:
          - role: endpoints

          relabel_configs:
          - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scrape]
            action: keep
            regex: true
          - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scheme]
            action: replace
            target_label: __scheme__
            regex: (https?)
          - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_path]
            action: replace
            target_label: __metrics_path__
            regex: (.+)
          - source_labels: [__address__, __meta_kubernetes_service_annotation_prometheus_io_port]
            action: replace
            target_label: __address__
            regex: ([^:]+)(?::\d+)?;(\d+)
            replacement: $1:$2
          - action: labelmap
            regex: __meta_kubernetes_service_label_(.+)
          - source_labels: [__meta_kubernetes_namespace]
            action: replace
            target_label: kubernetes_namespace
          - source_labels: [__meta_kubernetes_service_name]
            action: replace
            target_label: kubernetes_name
          ## drop this taget since we cant scrap basic user/pass into labels
          ## https://github.com/prometheus/prometheus/issues/2614
          - source_labels: [__meta_kubernetes_service_name]
            action: drop
            regex: router

        - job_name: "haproxy"
          basic_auth:
            username: ${ROUTER_BASIC_USERNAME}
            password: ${ROUTER_BASIC_PASSWORD}

          dns_sd_configs:
            - names:
              - haproxy.infra.getupcloud.com
              type: SRV
          relabel_configs:
          - source_labels: [__address__]
            action: replace
            regex: .*
            target_label: role
            replacement: infra

        - job_name: "master-nodes"
          dns_sd_configs:
            - names:
              - m.infra.getupcloud.com
              type: SRV
          relabel_configs:
          - source_labels: [__address__]
            action: replace
            target_label: server_name
            regex: ([^\.]+).*
          - source_labels: [__address__]
            action: replace
            regex: .*
            target_label: role
            replacement: infra

        - job_name: "infra-nodes"
          dns_sd_configs:
            - names:
              - i.infra.getupcloud.com
              type: SRV
          relabel_configs:
          - source_labels: [__address__]
            action: replace
            target_label: server_name
            regex: ([^\.]+).*
          - source_labels: [__address__]
            action: replace
            regex: .*
            target_label: role
            replacement: infra

        - job_name: "app-nodes"
          dns_sd_configs:
            - names:
              - a.infra.getupcloud.com
              type: SRV
          relabel_configs:
          - source_labels: [__address__]
            action: replace
            target_label: server_name
            regex: ([^\.]+).*
          - source_labels: [__address__]
            action: replace
            regex: .*
            target_label: role
            replacement: app
```

### prometheus-deployment-template.yaml

```
apiVersion: v1
kind: Template
metadata:
  name: prometheus-deployment-template

parameters:
- name: PROMETHEUS_IMAGE_VERSION
  value: v1.7.2

objects:
- apiVersion: v1
  kind: ServiceAccount
  metadata:
    name: prometheus

- apiVersion: v1
  kind: DeploymentConfig
  metadata:
    labels:
      app: prometheus
    name: prometheus
  spec:
    replicas: 1
    selector:
      app: prometheus
      deploymentconfig: prometheus
    strategy:
      activeDeadlineSeconds: 21600
      recreateParams:
        timeoutSeconds: 600
      type: Recreate
    template:
      metadata:
        labels:
          app: prometheus
          deploymentconfig: prometheus
      spec:
        containers:
        - command:
          - /bin/prometheus
          - -log.level=info
          - -config.file=/etc/prometheus/prometheus.yml
          - -storage.local.path=/prometheus
          - -web.console.libraries=/usr/share/prometheus/console_libraries
          - -web.console.templates=/usr/share/prometheus/consoles
          - -storage.local.target-heap-size=3221225472
          - -alertmanager.url=http://alertmanager:9093
          image: prom/prometheus:${PROMETHEUS_IMAGE_VERSION}
          imagePullPolicy: IfNotPresent
          name: prometheus
          ports:
          - containerPort: 9090
            protocol: TCP
          resources:
            limits:
              cpu: "2"
              memory: 4Gi
            requests:
              cpu: 100m
              memory: 3Gi
          volumeMounts:
          - mountPath: /prometheus
            name: prometheus-data
          - mountPath: /etc/prometheus
            name: prom-k8s
        dnsPolicy: ClusterFirst
        nodeSelector:
          role: infra
        restartPolicy: Always
        serviceAccount: prometheus
        serviceAccountName: prometheus
        terminationGracePeriodSeconds: 30
        volumes:
        - configMap:
            defaultMode: 420
            name: prom-k8s
          name: prom-k8s
        - name: prometheus-data
          persistentVolumeClaim:
            claimName: prometheus-data
    triggers:
    - type: ConfigChange
```


### alertmanager-config-template.yml

```
apiVersion: v1
kind: Template
metadata:
  name: alertmanager-config-template

parameters:
- name: PAGERDUTY_GENERIC_API_KEY
  required: true
- name: SLACK_HOOK_URL
  required: true
- name: SLACK_CHANNEL
  required: true

objects:
- apiVersion: v1
  kind: ConfigMap
  metadata:
    name: prom-alertmanager
  data:
    alertmanager.yml: |-
      global:
        # The smarthost and SMTP sender used for mail notifications.
        #smtp_smarthost: 'localhost:25'
        #smtp_from: 'alertmanager@example.org'

      # The root route on which each incoming alert enters.
      route:
        # The root route must not have any matchers as it is the entry point for
        # all alerts. It needs to have a receiver configured so alerts that do not
        # match any of the sub-routes are sent to someone.
        receiver: 'slack_operations'

        # The labels by which incoming alerts are grouped together. For example,
        # multiple alerts coming in for cluster=A and alertname=LatencyHigh would
        # be batched into a single group.
        group_by: ['alertname', 'cluster']

        # When a new group of alerts is created by an incoming alert, wait at
        # least 'group_wait' to send the initial notification.
        # This way ensures that you get multiple alerts for the same group that start
        # firing shortly after another are batched together on the first
        # notification.
        group_wait: 15s

        # When the first notification was sent, wait 'group_interval' to send a batch
        # of new alerts that started firing for that group.
        group_interval: 5m

        # If an alert has successfully been sent, wait 'repeat_interval' to
        # resend them.
        repeat_interval: 3h

        # All the above attributes are inherited by all child routes and can
        # overwritten on each.

        routes:
        # This routes performs a regular expression match on alert labels to
        # catch alerts that are related to a list of services.
        #- match_re:
        #    cluster: .*
        #  receiver: slack_operations
        #  continue: true
        #
        - match_re:
            alertname: .*
            cluster: .*
          receiver: pager_duty
          continue: true

      # Inhibition rules allow to mute a set of alerts given that another alert is
      # firing.
      # We use this to mute any warning-level notifications if the same alert is
      # already critical.
      inhibit_rules:
      - source_match:
          severity: 'critical'
        target_match:
          severity: 'warning'
        # Apply inhibition if the alertname is the same.
        equal: ['alertname', 'cluster']


      receivers:
      - name: slack_operations
        slack_configs:
        - send_resolved: true
          api_url: '${SLACK_HOOK_URL}'
          channel: '${SLACK_CHANNEL}'
          text: >-
                Summary: {{ .CommonAnnotations.summary }}
                Description: {{ .CommonAnnotations.description }}

      - name: pager_duty
        pagerduty_configs:
        - service_key: ${PAGERDUTY_GENERIC_API_KEY}
          send_resolved: true
```

### alertmanager-deployment-template.yaml

```
apiVersion: v1
kind: Template
metadata:
  name: alertmanager-deployment-template

parameters:
- name: ALERTMANAGER_IMAGE_VERSION
  value: v0.8.0

objects:
- apiVersion: v1
  kind: DeploymentConfig
  metadata:
    labels:
      app: alertmanager
    name: alertmanager
  spec:
    replicas: 1
    selector:
      app: alertmanager
      deploymentconfig: alertmanager
    strategy:
      activeDeadlineSeconds: 21600
      recreateParams:
        timeoutSeconds: 600
      type: Rolling
    template:
      metadata:
        labels:
          app: alertmanager
          deploymentconfig: alertmanager
      spec:
        containers:
        - name: alertmanager
          image: quay.io/prometheus/alertmanager:${ALERTMANAGER_IMAGE_VERSION}
          args:
            - -config.file=/etc/prometheus/alertmanager.yml
          imagePullPolicy: IfNotPresent
          name: prometheus
          ports:
          - containerPort: 9093
            protocol: TCP
          resources:
            limits:
              cpu: "100m"
              memory: 512Mi
            requests:
              cpu: 10m
              memory: 256Mi
          volumeMounts:
          - mountPath: /etc/prometheus
            name: prom-alertmanager
        dnsPolicy: ClusterFirst
        nodeSelector:
          role: infra
        restartPolicy: Always
        terminationGracePeriodSeconds: 30
        volumes:
        - configMap:
            defaultMode: 420
            name: prom-alertmanager
          name: prom-alertmanager
    triggers:
    - type: ConfigChange
```

### node-exporter-deployment-template.yaml

```
apiVersion: v1
kind: Template
metadata:
  name: node-exporter-deployment-template

parameters:
- name: NODE_EXPORTER_IMAGE_VERSION
  value: v0.14.0

objects:
- apiVersion: v1
  kind: ServiceAccount
  metadata:
    name: prometheus-node-exporter

- apiVersion: extensions/v1beta1
  kind: DaemonSet
  metadata:
    labels:
      component: node-exporter
    name: node-exporter
  spec:
    selector:
      matchLabels:
        component: node-exporter
    template:
      metadata:
        creationTimestamp: null
        labels:
          component: node-exporter
        name: node-exporter
      spec:
        containers:
        - image: quay.io/prometheus/node-exporter:${NODE_EXPORTER_IMAGE_VERSION}
          imagePullPolicy: IfNotPresent
          name: node-exporter
          args:
            - --collector.procfs
            - /host/proc
            - --collector.sysfs
            - /host/sys
            - --collector.filesystem.ignored-mount-points
            - "^/(sys|proc|dev|host|etc)($|/)"
          resources:
            limits:
              cpu: 100m
              memory: 512Mi
            requests:
              cpu: 10m
              memory: 256Mi
          securityContext:
            privileged: true
          volumeMounts:
          - mountPath: /host/proc
            name: proc
            readOnly: true
          - mountPath: /host/sys
            name: sys
            readOnly: true
        volumes:
        - hostPath:
            path: /proc
          name: proc
        - hostPath:
            path: /sys
          name: sys
        dnsPolicy: ClusterFirst
        hostNetwork: true
        restartPolicy: Always
        serviceAccount: prometheus-node-exporter
        serviceAccountName: prometheus-node-exporter
        terminationGracePeriodSeconds: 30
    templateGeneration: 1
    updateStrategy:
      rollingUpdate:
        maxUnavailable: 1
      type: RollingUpdate
```

### kubestate-exporter-deployment-template.yaml

```
apiVersion: v1
kind: Template
metadata:
  name: kubestate-exporter-deployment-template

parameters:
- name: KUBESTATE_IMAGE_VERSION
  value: v1.0.1

objects:
- apiVersion: v1
  kind: DeploymentConfig
  metadata:
    labels:
      app: kubestate-exporter
    name: kubestate-exporter
  spec:
    replicas: 1
    selector:
      app: kubestate-exporter
      deploymentconfig: kubestate-exporter
    strategy:
      activeDeadlineSeconds: 1200
      recreateParams:
        timeoutSeconds: 600
      type: Recreate
    template:
      metadata:
        labels:
          app: kubestate-exporter
          deploymentconfig: kubestate-exporter
      spec:
        containers:
        - image: quay.io/coreos/kube-state-metrics:${KUBESTATE_IMAGE_VERSION}
          imagePullPolicy: IfNotPresent
          name: kubestate
          ports:
          - containerPort: 8080
            protocol: TCP
          resources:
            limits:
              cpu: 100
              memory: 512Mi
            requests:
              cpu: 10m
              memory: 128Mi
        nodeSelector:
          role: infra
        restartPolicy: Always
        serviceAccount: prometheus
        serviceAccountName: prometheus
        terminationGracePeriodSeconds: 30
    triggers:
    - type: ConfigChange
```




### cloudwatch-exporter-config-template.yaml

```
apiVersion: v1
kind: Template
metadata:
  name: cloudwatch-exporter-config-template

objects:
- apiVersion: v1
  kind: ConfigMap
  metadata:
    name: cloudwatch-exporter
  data:
    config.yml: |-
      ---
      region: us-east-1
      metrics:
       - aws_namespace: AWS/ELB
         aws_metric_name: RequestCount
         aws_dimensions: [AvailabilityZone, LoadBalancerName]
         aws_dimension_select:
           LoadBalancerName: [infra, api-external, api-internal]
         aws_statistics: [Sum]

       - aws_namespace: AWS/ELB
         aws_metric_name: HealthyHostCount
         aws_dimensions: [AvailabilityZone, LoadBalancerName]
         aws_statistics: [Average]

       - aws_namespace: AWS/ELB
         aws_metric_name: UnHealthyHostCount
         aws_dimensions: [AvailabilityZone, LoadBalancerName]
         aws_statistics: [Average]

       - aws_namespace: AWS/ELB
         aws_metric_name: Latency
         aws_dimensions: [AvailabilityZone, LoadBalancerName]
         aws_statistics: [Average]

       - aws_namespace: AWS/ELB
         aws_metric_name: SurgeQueueLength
         aws_dimensions: [AvailabilityZone, LoadBalancerName]
         aws_statistics: [Maximum, Sum]
```

### cloudwatch-exporter-deployment-template.yaml

```
apiVersion: v1
kind: Template
metadata:
  name: cloudwatch-exporter-deployment-template

parameters:
- name: CLOUDWATCH_IMAGE_VERSION
  value: latest
- name: AWS_ACCESS_KEY_ID
  required: true
- name: AWS_SECRET_ACCESS_KEY
  required: true

objects:
- apiVersion: v1
  kind: DeploymentConfig
  metadata:
    labels:
      app: cloudwatch-exporter
    name: cloudwatch-exporter
  spec:
    replicas: 1
    selector:
      app: cloudwatch-exporter
    strategy:
    template:
      metadata:
        labels:
          app: cloudwatch-exporter
      spec:
        containers:
        - image: prom/cloudwatch-exporter:${CLOUDWATCH_IMAGE_VERSION}
          imagePullPolicy: IfNotPresent
          name: cloudwatch-exporter
          env:
          - name: AWS_ACCESS_KEY_ID
            value: "${AWS_ACCESS_KEY_ID}"
          - name: AWS_SECRET_ACCESS_KEY
            value: "${AWS_SECRET_ACCESS_KEY}"
          ports:
          - containerPort: 9106
          resources:
            limits:
              cpu: 100m
              memory: 512Mi
            requests:
              cpu: 10m
              memory: 256Mi
          volumeMounts:
          - mountPath: /config
            name: config
            readOnly: true
        volumes:
        - configMap:
            name: cloudwatch-exporter
          name: config
```





### eventrouter-config-template.yaml

```
apiVersion: v1
kind: Template
metadata:
  name: eventrouter-config-template

objects:
- apiVersion: v1
  metadata:
    name: eventrouter-config
  data:
    config.json: {"sink":"glog"}
  kind: ConfigMap
```

### eventrouter-deployment-template.yaml

```
apiVersion: v1
kind: Template
metadata:
  name: eventrouter-deployment-template

parameters:
- name: EVENTROUTER_IMAGE_VERSION
  value: v0.1

objects:
- apiVersion: v1
  kind: ClusterRole
  metadata:
    name: eventrouter
  rules:
  - apiGroups:
    - ""
    attributeRestrictions: null
    resources:
    - events
    verbs:
    - get
    - list
    - watch

- apiVersion: v1
  kind: ServiceAccount
  metadata:
    name: eventrouter

- apiVersion: v1
  kind: DeploymentConfig
  metadata:
    app: eventrouter
  spec:
    replicas: 1
    selector:
      app: eventrouter
    strategy:
      activeDeadlineSeconds: 21600
      resources: {}
      rollingParams:
        intervalSeconds: 1
        maxSurge: 25%
        maxUnavailable: 25%
        timeoutSeconds: 600
        updatePeriodSeconds: 1
      type: Rolling
    template:
      metadata:
        creationTimestamp: null
        labels:
          app: eventrouter
      spec:
        containers:
        - image: gcr.io/heptio-images/eventrouter:${EVENTROUTER_IMAGE_VERSION}
          imagePullPolicy: IfNotPresent
          name: kube-eventrouter
          resources:
            limits:
              cpu: 100
              memory: 512Mi
            requests:
              cpu: 10m
              memory: 128Mi
          terminationMessagePath: /dev/termination-log
          terminationMessagePolicy: File
          volumeMounts:
          - mountPath: /etc/eventrouter
            name: config
        dnsPolicy: ClusterFirst
        restartPolicy: Always
        schedulerName: default-scheduler
        serviceAccount: eventrouter
        serviceAccountName: eventrouter
        terminationGracePeriodSeconds: 30
        volumes:
        - configMap:
            defaultMode: 420
            name: eventrouter-config
          name: config
    triggers:
    - type: ConfigChange
```

### grafana-config-template.yaml

```
apiVersion: v1
kind: Template
metadata:
  name: grafana-config-template

parameters:
- name: GF_ADMIN_USERNAME
  value: admin
- name: GF_ADMIN_PASSWORD
  from: '[a-zA-Z0-9]{16}'
  generate: expression
- name: GF_SECRET_KEY
  from: '[a-zA-Z0-9]{16}'
  generate: expression

objects:
- apiVersion: v1
  kind: ConfigMap
  metadata:
    creationTimestamp: null
    name: grafanaini
  data:
    grafana.ini: |
      # Everything has defaults so you only need to uncomment things you want to
      # change

      # possible values : production, development
      ; app_mode = prod

      # instance name, defaults to HOSTNAME environment variable value or hostname if HOSTNAME var is empty
      ; instance_name = ${HOSTNAME}

      #################################### Paths ####################################
      [paths]
      # Path to where grafana can store temp files, sessions, and the sqlite3 db (if that is used)
      #
      ;data = /var/lib/grafana
      #
      # Directory where grafana can store logs
      #
      ;logs = /var/log/grafana
      #
      # Directory where grafana will automatically scan and look for plugins
      #
      ;plugins = /var/lib/grafana/plugins

      #
      #################################### Server ####################################
      [server]
      # Protocol (http, https, socket)
      ;protocol = http

      # The ip address to bind to, empty will bind to all interfaces
      ;http_addr =

      # The http port  to use
      ;http_port = 3000

      # The public facing domain name used to access grafana from a browser
      ;domain = localhost

      # Redirect to correct domain if host header does not match domain
      # Prevents DNS rebinding attacks
      ;enforce_domain = false

      # The full public facing url you use in browser, used for redirects and emails
      # If you use reverse proxy and sub path specify full url (with sub path)
      ;root_url = http://localhost:3000

      # Log web requests
      ;router_logging = false

      # the path relative working path
      ;static_root_path = public

      # enable gzip
      ;enable_gzip = false

      # https certs & key file
      ;cert_file =
      ;cert_key =

      # Unix socket path
      ;socket =

      #################################### Database ####################################
      [database]
      # You can configure the database connection by specifying type, host, name, user and password
      # as seperate properties or as on string using the url propertie.

      # Either "mysql", "postgres" or "sqlite3", it's your choice
      type = sqlite3
      ;host = 127.0.0.1:3306
      ;name = grafana
      ;user = root
      # If the password contains # or ; you have to wrap it with trippel quotes. Ex """#password;"""
      ;password =

      # Use either URL or the previous fields to configure the database
      # Example: mysql://user:secret@host:port/database
      ;url =

      # For "postgres" only, either "disable", "require" or "verify-full"
      ;ssl_mode = disable

      # For "sqlite3" only, path relative to data_path setting
      path = grafana.db

      # Max conn setting default is 0 (mean not set)
      ;max_idle_conn =
      ;max_open_conn =


      #################################### Session ####################################
      [session]
      # Either "memory", "file", "redis", "mysql", "postgres", default is "file"
      ;provider = file

      # Provider config options
      # memory: not have any config yet
      # file: session dir path, is relative to grafana data_path
      # redis: config like redis server e.g. `addr=127.0.0.1:6379,pool_size=100,db=grafana`
      # mysql: go-sql-driver/mysql dsn config string, e.g. `user:password@tcp(127.0.0.1:3306)/database_name`
      # postgres: user=a password=b host=localhost port=5432 dbname=c sslmode=disable
      ;provider_config = sessions

      # Session cookie name
      ;cookie_name = grafana_sess

      # If you use session in https only, default is false
      ;cookie_secure = false

      # Session life time, default is 86400
      ;session_life_time = 86400

      #################################### Data proxy ###########################
      [dataproxy]

      # This enables data proxy logging, default is false
      ;logging = true


      #################################### Analytics ####################################
      [analytics]
      # Server reporting, sends usage counters to stats.grafana.org every 24 hours.
      # No ip addresses are being tracked, only simple counters to track
      # running instances, dashboard and error counts. It is very helpful to us.
      # Change this option to false to disable reporting.
      ;reporting_enabled = true

      # Set to false to disable all checks to https://grafana.net
      # for new vesions (grafana itself and plugins), check is used
      # in some UI views to notify that grafana or plugin update exists
      # This option does not cause any auto updates, nor send any information
      # only a GET request to http://grafana.com to get latest versions
      ;check_for_updates = true

      # Google Analytics universal tracking code, only enabled if you specify an id here
      ;google_analytics_ua_id =

      #################################### Security ####################################
      [security]
      # default admin user, created on startup
      admin_user = ${GF_ADMIN_USERNAME}

      # default admin password, can be changed before first start of grafana, or in profile settings
      admin_password = ${GF_ADMIN_PASSWORD}

      # used for signing
      secret_key = ${GF_SECRET_KEY}

      # Auto-login remember days
      ;login_remember_days = 7
      ;cookie_username = grafana_user
      ;cookie_remember_name = grafana_remember

      # disable gravatar profile images
      ;disable_gravatar = false

      # data source proxy whitelist (ip_or_domain:port separated by spaces)
      ;data_source_proxy_whitelist =

      [snapshots]
      # snapshot sharing options
      ;external_enabled = true
      ;external_snapshot_url = https://snapshots-origin.raintank.io
      ;external_snapshot_name = Publish to snapshot.raintank.io

      # remove expired snapshot
      ;snapshot_remove_expired = true

      # remove snapshots after 90 days
      ;snapshot_TTL_days = 90

      #################################### Users ####################################
      [users]
      # disable user signup / registration
      ;allow_sign_up = true

      # Allow non admin users to create organizations
      ;allow_org_create = true

      # Set to true to automatically assign new users to the default organization (id 1)
      ;auto_assign_org = true

      # Default role new users will be automatically assigned (if disabled above is set to true)
      ;auto_assign_org_role = Viewer

      # Background text for the user field on the login page
      ;login_hint = email or username

      # Default UI theme ("dark" or "light")
      ;default_theme = dark

      [auth]
      # Set to true to disable (hide) the login form, useful if you use OAuth, defaults to false
      disable_login_form = true

      # Set to true to disable the signout link in the side menu. useful if you use auth.proxy, defaults to false
      ;disable_signout_menu = false

      #################################### Google Auth ##########################
      [auth.google]
      ;enabled = true
      ;allow_sign_up = true
      ;client_id = $GF_GOOGLE_AUTH_CLIENT_ID
      ;client_secret = $GF_GOOGLE_AUTH_CLIENT_SECRET
      ;scopes = https://www.googleapis.com/auth/userinfo.profile https://www.googleapis.com/auth/userinfo.email
      ;auth_url = https://accounts.google.com/o/oauth2/auth
      ;token_url = https://accounts.google.com/o/oauth2/token
      ;api_url = https://www.googleapis.com/oauth2/v1/userinfo
      ;allowed_domains = example.com

      #################################### Basic Auth ##########################
      [auth.basic]
      enabled = true

      #################################### SMTP / Emailing ##########################
      [smtp]
      ;enabled = false
      ;host = localhost:25
      ;user =
      # If the password contains # or ; you have to wrap it with trippel quotes. Ex """#password;"""
      ;password =
      ;cert_file =
      ;key_file =
      ;skip_verify = false
      ;from_address = admin@grafana.localhost
      ;from_name = Grafana

      [emails]
      ;welcome_email_on_sign_up = false

      #################################### Logging ##########################
      [log]
      # Either "console", "file", "syslog". Default is console and  file
      # Use space to separate multiple modes, e.g. "console file"
      mode = console

      # Either "trace", "debug", "info", "warn", "error", "critical", default is "info"
      level = info

      # optional settings to set different levels for specific loggers. Ex filters = sqlstore:debug
      ;filters =


      # For "console" mode only
      [log.console]
      ;level =

      # log line format, valid options are text, console and json
      ;format = console

      # For "file" mode only
      [log.file]
      ;level =

      # log line format, valid options are text, console and json
      ;format = text

      # This enables automated log rotate(switch of following options), default is true
      ;log_rotate = true

      # Max line number of single file, default is 1000000
      ;max_lines = 1000000

      # Max size shift of single file, default is 28 means 1 << 28, 256MB
      ;max_size_shift = 28

      # Segment log daily, default is true
      ;daily_rotate = true

      # Expired days of log file(delete after max days), default is 7
      ;max_days = 7

      [log.syslog]
      ;level =

      # log line format, valid options are text, console and json
      ;format = text

      # Syslog network type and address. This can be udp, tcp, or unix. If left blank, the default unix endpoints will be used.
      ;network =
      ;address =

      # Syslog facility. user, daemon and local0 through local7 are valid.
      ;facility =

      # Syslog tag. By default, the process' argv[0] is used.
      ;tag =


      #################################### AMQP Event Publisher ##########################
      [event_publisher]
      ;enabled = false
      ;rabbitmq_url = amqp://localhost/
      ;exchange = grafana_events

      ;#################################### Dashboard JSON files ##########################
      [dashboards.json]
      ;enabled = false
      path = /var/lib/grafana/dashboards

      #################################### Alerting ############################
      [alerting]
      # Disable alerting engine & UI features
      enabled = true
      # Makes it possible to turn off alert rule execution but alerting UI is visible
      ;execute_alerts = true

      #################################### Internal Grafana Metrics ##########################
      # Metrics available at HTTP API Url /api/metrics
      [metrics]
      # Disable / Enable internal metrics
      ;enabled           = true

      # Publish interval
      ;interval_seconds  = 10

      # Send internal metrics to Graphite
      [metrics.graphite]
      # Enable by setting the address setting (ex localhost:2003)
      ;address =
      ;prefix = prod.grafana.%(instance_name)s.

      #################################### Grafana.com integration  ##########################
      # Url used to to import dashboards directly from Grafana.com
      [grafana_com]
      ;url = https://grafana.com

      #################################### External image storage ##########################
      [external_image_storage]
      # Used for uploading images to public servers so they can be included in slack/email messages.
      # you can choose between (s3, webdav)
      ;provider =

      [external_image_storage.s3]
      ;bucket_url =
      ;access_key =
      ;secret_key =

      [external_image_storage.webdav]
      ;url =
      ;public_url =
      ;username =
      ;password =
```


### grafana-deployment-template.yaml

```
apiVersion: v1
kind: Template
metadata:
  name: grafana-deployment-template

parameters:
- name: GRAFANA_IMAGE_VERSION
  value: 4.5.2

objects:
- apiVersion: v1
  kind: DeploymentConfig
  metadata:
    labels:
      app: grafana
    name: grafana
  spec:
    replicas: 1
    selector:
      app: grafana
      deploymentconfig: grafana
    strategy:
      activeDeadlineSeconds: 21600
      rollingParams:
        intervalSeconds: 1
        maxSurge: 25%
        maxUnavailable: 25%
        timeoutSeconds: 600
        updatePeriodSeconds: 1
      type: Rolling
    template:
      metadata:
        labels:
          app: grafana
          deploymentconfig: grafana
      spec:
        containers:
        - image: grafana/grafana:${GRAFANA_IMAGE_VERSION}
          imagePullPolicy: IfNotPresent
          name: grafana
          ports:
          - containerPort: 3000
            protocol: TCP
          resources:
            limits:
              cpu: "1"
              memory: 2Gi
            requests:
              cpu: 100m
              memory: 512Mi
          volumeMounts:
          - mountPath: /etc/grafana
            name: grafanaini
          - mountPath: /var/lib/grafana
            name: grafana-data-volume
        dnsPolicy: ClusterFirst
        nodeSelector:
          role: infra
        restartPolicy: Always
        terminationGracePeriodSeconds: 30
        volumes:
        - configMap:
            defaultMode: 420
            name: grafanaini
          name: grafanaini
        - name: grafana-data-volume
          persistentVolumeClaim:
            claimName: grafana-data
    triggers:
    - type: ConfigChange
```

### proxy-deployment-template.yaml

```
apiVersion: v1
kind: Template
metadata:
  name: proxy-deployment-template

parameters:
- name: APPS_DOMAIN
  required: true
- name: PROXY_ADMIN_HTPASSWD_HASH
  required: true

objects:
- apiVersion: v1
  kind: ConfigMap
  metadata:
    name: proxy-htpasswd
  data:
    .htpasswd-prometheus: |
      ${PROXY_ADMIN_HTPASSWD_HASH}
    .htpasswd-alertmanager: |
      ${PROXY_ADMIN_HTPASSWD_HASH}

- apiVersion: v1
  kind: DeploymentConfig
  metadata:
    labels:
      app: proxy
    name: proxy
  spec:
    replicas: 1
    selector:
      deploymentconfig: proxy
    strategy:
      activeDeadlineSeconds: 21600
      resources: {}
      rollingParams:
        intervalSeconds: 1
        maxSurge: 25%
        maxUnavailable: 25%
        timeoutSeconds: 600
        updatePeriodSeconds: 1
      type: Rolling
    template:
      metadata:
        labels:
          app: proxy
          deploymentconfig: proxy
      spec:
        containers:
        - env:
          - name: PROXY_ROUTE_TO_SERVICE
            value: "prometheus.${APPS_DOMAIN},prometheus:9090 alertmanager.${APPS_DOMAIN},alertmanager:9093"
          image: getupcloud/nginx-proxy:latest
          imagePullPolicy: IfNotPresent
          name: proxy
          ports:
          - containerPort: 8080
            protocol: TCP
          - containerPort: 8443
            protocol: TCP
          resources:
            limits:
              cpu: 50m
              memory: 512Mi
            requests:
              cpu: 10m
              memory: 128Mi
          volumeMounts:
          - mountPath: /opt/app-root/auth
            name: proxy-htpasswd
        dnsPolicy: ClusterFirst
        restartPolicy: Always
        terminationGracePeriodSeconds: 30
        volumes:
        - configMap:
            defaultMode: 420
            name: proxy-htpasswd
          name: proxy-htpasswd
    triggers:
    - type: ConfigChange
```
