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
oc adm new-project $NS --node-selector 'role=app'
```

Setup security policies.

```
oc -n $NS create -f configs/rbac.yaml
oc -n $NS adm policy add-cluster-role-to-user eventrouter-exporter -z eventrouter-exporter
oc -n $NS adm policy add-cluster-role-to-user cluster-reader -z kubestate-exporter
oc -n $NS adm policy add-cluster-role-to-user cluster-reader -z prometheus
oc -n default adm policy add-scc-to-user privileged -z node-exporter
oc -n $NS adm policy add-scc-to-user privileged -z alertmanager
```

Create config files

```
oc -n $NS process -o yaml -f configs/alertmanager-config-template.yaml \
    -p PAGERDUTY_GENERIC_API_KEY=$PAGERDUTY_GENERIC_API_KEY \
    -p SLACK_HOOK_URL=$SLACK_HOOK_URL \
    -p SLACK_CHANNEL=$SLACK_CHANNEL \
    | oc -n $NS create -f -

oc -n $NS process -o yaml -f configs/prometheus-config-template.yaml \
    -p CLUSTER_NAME=$CLUSTER_NAME \
    -p ROUTER_BASIC_USERNAME=$ROUTER_BASIC_USERNAME \
    -p ROUTER_BASIC_PASSWORD=$ROUTER_BASIC_PASSWORD \
    | oc -n $NS create -f -

oc -n $NS process -o yaml -f configs/proxy-config-template.yaml \
    -p 'PROXY_ADMIN_HTPASSWD_HASH=$PROXY_ADMIN_HTPASSWD_HASH' \
    | oc -n $NS create -f -

oc -n $NS process -o yaml -f configs/grafana-config-template.yaml \
    | oc -n $NS create -f -

oc -n $NS create -f configs/cloudwatch-exporter-config.yaml
oc -n $NS create -f configs/eventrouter-exporter-config.yaml
```

Deploy stuff

```
oc -n $NS process -o yaml -f exporters/node-exporter-deployment-template.yaml \
    | oc -n default create -f -

oc -n $NS process -o yaml -f exporters/kubestate-exporter-deployment-template.yaml \
    | oc -n $NS create -f -

oc -n $NS process -o yaml -f exporters/cloudwatch-exporter-deployment-template.yaml \
    -p AWS_ACCESS_KEY_ID=${AWS_ACCESS_KEY_ID} \
    -p AWS_SECRET_ACCESS_KEY=${AWS_SECRET_ACCESS_KEY} \
    | oc -n $NS create -f -

oc -n $NS process -o yaml -f exporters/eventrouter-exporter-deployment-template.yaml \
    | oc -n $NS create -f -

oc -n $NS process -o yaml -f alertmanager/alertmanager-deployment-template.yaml \
    | oc -n $NS create -f -

oc -n $NS process -o yaml -f prometheus/prometheus-deployment-template.yaml \
    | oc -n $NS create -f -

oc -n $NS process -o yaml -f grafana/grafana-deployment-template.yaml \
    -p APPS_DOMAIN=${APPS_DOMAIN} \
    | oc -n $NS create -f -

oc -n $NS process -o yaml -f proxy/proxy-deployment-template.yaml \
    -p APPS_DOMAIN=${APPS_DOMAIN} \
    | oc -n $NS create -f -
```
