# h2o-ocp
This repo depicts a simple example on how to deploy a H2o instance into an Openshift 4 cluster.

The first step is to define the `OperatorGroup`, which selects target namespaces in which to generate required RBAC access for its member Operators, and a `Subscription` to subscribe a namespace to an Operator. The `h2o-01-operator.yaml` template does all that:

```sh
$ oc process -f templates/h2o-01-operator.yaml | oc apply -f -
```

At this point, OLM is now aware of the selected Operator. A cluster service version (CSV) for the Operator should appear in the target namespace, and APIs provided by the Operator should be available for creation.
Let's check it:

```sh
$ oc get crd | grep "h2o"
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
$ oc process -f templates/h2o-02-instance.yaml | oc apply -f -
```

Once the instance is deployed, a `route` needs to be created to expose the service(`svc`)

```sh
$ oc expose svc/h2o-test
```

By executing `oc get routes` we can copy the route create it and access the GUI of H2o in our desired navigation explorer.
