# h2o-ocp
This repo depicts a simple example on how to deploy a H2o instance into an Openshift 4 cluster.

Making use of templates the steps required are the following ones:

```sh
$ oc process -f templates/h2o-01-operator.yaml | oc apply -f -
$ oc process -f templates/h2o-02-instance.yaml | oc apply -f -
```

Once the instance is deployed, a `route` needs to be created to expose the service(`svc`)

```sh
$ oc expose svc/h2o-test
```

By executing `oc get routes` we can copy the route create it and access the GUI of H2o in our desired navigation explorer.
