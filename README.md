# H2O Operator Openshift 4
This repo depicts a simple example on how to deploy a H2o instance into an Openshift 4 cluster.

The first step is to define the `OperatorGroup`, which selects target namespaces in which to generate required RBAC access for its member Operators, and a `Subscription` to subscribe a namespace to an Operator. The `h2o-01-operator.yaml` template does all that:

```sh
OPERATOR_NAMESPACE=h2o-opertaor # Define the namespace
$ oc process -f templates/h2o-01-operator.yaml -p OPERATOR_NAMESPACE=${OPERATOR_NAMESPACE} | oc apply -f -
```

At this point, OLM is now aware of the selected Operator. A cluster service version (CSV) for the Operator should appear in the target namespace, and APIs provided by the Operator should be available for creation.
Let's check it:

```sh
$ oc get crd -n ${OPERATOR_NAMESPACE}| grep "h2o"
h2os.h2o.ai                                                       2021-04-19T11:32:49Z

$ oc describe crd h2os.h2o.ai
Name:         h2os.h2o.ai
Namespace:    
Labels:       operators.coreos.com/h2o-operator.h2o-operator1=
Annotations:  <none>
API Version:  apiextensions.k8s.io/v1
Kind:         CustomResourceDefinition
Metadata:
  Creation Timestamp:  2021-04-19T11:32:49Z
  Generation:          1
  Managed Fields:
    API Version:  apiextensions.k8s.io/v1beta1
    Fields Type:  FieldsV1
    ...
Spec:
  Conversion:
    Strategy:  None
  Group:       h2o.ai
  Names:
    Kind:                   H2O
    List Kind:              H2OList
    Plural:                 h2os
    Singular:               h2o
  Preserve Unknown Fields:  true
  Scope:                    Namespaced
  Versions:
    Name:  v1beta
    Schema:
      openAPIV3Schema:
        Properties:
          Spec:
            One Of:
              Required:
                version
              Required:
                customImage
            Properties:
              Custom Image:
                Properties:
                  Command:
                    Type:  string
                  Image:
                    Type:  string
                Required:
                  image
                Type:  object
              Nodes:
                Type:  integer
              Resources:
                Properties:
                  Cpu:
                    Minimum:  1
                    Type:     integer
                  Memory:
                    Pattern:  ^([+-]?[0-9.]+)([eEinumkKMGTP]*[-+]?[0-9]*)$
                    Type:     string
                  Memory Percentage:
                    Maximum:  100
                    Minimum:  1
                    Type:     integer
                Required:
                  cpu
                  memory
                Type:  object
              Version:
                Type:  string
            Required:
              nodes
              resources
            Type:  object
          Status:
            Type:  object
        Type:      object
    Served:        true
    Storage:       true
    Subresources:
      Status:
Status:
  Accepted Names:
    Kind:       H2O
    List Kind:  H2OList
    Plural:     h2os
    Singular:   h2o
  Conditions:
    ...
  Stored Versions:
    v1beta
```

Now, a CR of kind `H2O` can be deployed into the namespace:

```sh
$ oc process -f templates/h2o-02-instance.yaml -p OPERATOR_NAMESPACE=${OPERATOR_NAMESPACE} | oc apply -f -
```

Once the instance is deployed, a `route` needs to be created to expose the service(`svc`)

```sh
$ oc expose svc/h2o-test -n ${OPERATOR_NAMESPACE}
```

By executing `oc get routes` we can copy the route create it and access the GUI of H2o in our desired navigation explorer.

## Monitoring
A typical OpenShift monitoring stack includes Prometheus for monitoring both systems and services, and Grafana for analyzing and visualizing metrics.

Administrators are often looking to write custom queries and create custom dashboards in Grafana. However, Grafana instances provided with the monitoring stack (and its dashboards) are read-only. To solve this problem, we can use the community-powered Grafana operator provided by OperatorHub. I will follow the implementation accurately explained [here](https://github.com/alvarolop/rhdg8-server#4-monitoring-rhdg-with-grafana).

As with the H2O operator, we first need to subscribe and deploy the operator using the following template:

```sh
OPERATOR_NAMESPACE="grafana"
$ oc process -f templates/grafana-01-operator.yaml -p OPERATOR_NAMESPACE=${OPERATOR_NAMESPACE}| oc apply -f -
```

Now, a Grafana instance is created using the operator:

```sh
oc process -f templates/grafana-02-instance.yaml -p OPERATOR_NAMESPACE=${OPERATOR_NAMESPACE}| oc apply -f
```

A `GrafanaDataSource`, that points to the Prometheus metrics, is created:

```sh
oc adm policy add-cluster-role-to-user cluster-monitoring-view -z grafana-serviceaccount -n ${OPERATOR_NAMESPACE}
BEARER_TOKEN=$(oc serviceaccounts get-token grafana-serviceaccount -n ${OPERATOR_NAMESPACE})
oc process -f templates/grafana-03-datasource.yaml -p BEARER_TOKEN=${BEARER_TOKEN} | oc apply -f -
```

And finally the Grafana dashboard is to be created:

```sh
DASHBOARD_NAME="grafana-dashboard-h2o"
# Create a configMap containing the Dashboard
oc create configmap $DASHBOARD_NAME --from-file=dashboard=grafana/$DASHBOARD_NAME.json -n ${OPERATOR_NAMESPACE}
# Create a Dashboard object that automatically updates Grafana
oc process -f templates/grafana-04-dashboard.yaml -p DASHBOARD_NAME=$DASHBOARD_NAME | oc apply -f -
```
