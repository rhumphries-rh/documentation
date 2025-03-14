## Deploying Grafana on Openshift 4

OpenShift users want access to a Grafana interface in order to build custom dashboards for their cluster and application workloads. The Grafana that shipped with OpenShift was read-only and has been [deprecated in OpenShift 4.11 and removed in OpenShift 4.12](https://issues.redhat.com/browse/MON-1591?focusedCommentId=19239654&page=com.atlassian.jira.plugin.system.issuetabpanels%3Acomment-tabpanel#comment-19239654).

Since OpenShift uses Prometheus for both Cluster and User Workload metrics, its fairly straight forward to deploy a Grafana instance using the Grafana Operator and then view those cluster metrics and create custom Dashboards.

### Grafana Operator

1. Create a namespace for the Grafana Operator to be installed in

    ```bash
    oc new-project grafana-operator
    ```

1. Deploy the Grafana Operator

    ```yaml
    cat << EOF | oc create -f -
    apiVersion: operators.coreos.com/v1
    kind: OperatorGroup
    metadata:
      generateName: grafana-operator-
      namespace: grafana-operator
    spec:
      targetNamespaces:
      - grafana-operator
    ---
    apiVersion: operators.coreos.com/v1alpha1
    kind: Subscription
    metadata:
      generateName: grafana-operator-
      namespace: grafana-operator
    spec:
      channel: v4
      name: grafana-operator
      installPlanApproval: Automatic
      source: community-operators
      sourceNamespace: openshift-marketplace
    EOF
    ```

1. Wait for the Operator to be ready

    ```bash
    oc -n grafana-operator rollout status \
      deployment grafana-operator-controller-manager
    ```

### Deploy Grafana

1. Add the Managed OpenShift Black Belt helm repo

    ```bash
    helm repo add mobb https://mobb.github.io/helm-charts
    ```

1. Deploy Grafana

    ```bash
    helm upgrade --install -n grafana-operator \
      grafana mobb/grafana-cr --set "basicAuthPassword=myPassword"
    ```

### Create Prometheus DataSource

1. Give the Grafana service account the `cluster-monitoring-view` role

    ```bash
    oc adm policy add-cluster-role-to-user \
      cluster-monitoring-view -z grafana-serviceaccount
    ```

1. Get a bearer token for the grafana service account

    ```bash
    BEARER_TOKEN=$(oc create token grafana-serviceaccount)
    ```

1. Deploy the prometheus data source

    ```yaml
    cat << EOF | oc apply -f -
    apiVersion: integreatly.org/v1alpha1
    kind: GrafanaDataSource
    metadata:
      name: prometheus-grafanadatasource
    spec:
      datasources:
        - access: proxy
          editable: true
          isDefault: true
          jsonData:
            httpHeaderName1: 'Authorization'
            timeInterval: 5s
            tlsSkipVerify: true
          name: Prometheus
          secureJsonData:
            httpHeaderValue1: 'Bearer ${BEARER_TOKEN}'
          type: prometheus
          url: 'https://thanos-querier.openshift-monitoring.svc.cluster.local:9091'
      name: prometheus-grafanadatasource.yaml
    EOF
    ```

1. Deploy a Dashboard

    ```bash
    oc apply -f https://raw.githubusercontent.com/kevchu3/openshift4-grafana/master/dashboards/crds/cluster_metrics.ocp412.grafanadashboard.yaml
    ```

1. Get the URL for Grafana

    ```bash
    oc get route grafana-route -o jsonpath='{"https://"}{.spec.host}{"\n"}'
    ```

1. Browse to the URL from above, log in via your OCP credentials



1. Click **Dashboards** -> **Manage** -> **grafana-operator** -> **Cluster Metrics**

![grafana dashboard](./grafana-dashboard.png)

