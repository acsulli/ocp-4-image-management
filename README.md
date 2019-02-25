# OpenShift 4 Image Management

The goal of this module is to cover aspects of container image management and utilization for OCP 4.  This includes, but is not limited to:

* [Deploying and configuring](https://gist.github.com/acsulli/c23469c68d4a0988f5d6c3f1c5be6977#modifying-the-registry-configuration)
* [Accessing the registry](https://docs.openshift.com/container-platform/4.0/registry/accessing-the-registry.html)
* [Registry metrics](https://docs.openshift.com/container-platform/4.0/registry/accessing-the-registry.html#registry-viewing-contents-accessing-the-registry)

Specific tasks to explore these concepts can be found here:

* [Demos](./demos/)
* [Workshop](./workshop/)

## Image registry overview

The "cluster image registry" is the instance of the image registry which is deployed and managed by the `cluster-image-registry-operator`.  OCP 4.0 deploys the registry, by default, and configures it to use S3 storage.

The registry is no longer managed using `oc adm registry` commands, instead it is managed using the operator and config objects.  Some basic operations are shown below:

* To view the status of the registry, use the command `oc get clusteroperator`.
  
  ```
  [you@bastion ~]$ oc get clusteroperator
  NAME                                    VERSION                  AVAILABLE   PROGRESSING   FAILING   SINCE
  cluster-autoscaler-operator             v4.0.0-0.150.0.0-dirty   True        False         False     38m
  cluster-image-registry-operator         v4.0.0-0.150.0.0-dirty   True        False         False     25m
  cluster-monitoring-operator                                      True        True          False     32m
  [...]
  ```
  
  We can see that the line for `cluster-image-registry-operator` indicates that it is avaiable.  The progressing and failing columns indicate whether the service is still changing (progressing) or if something is not functioning (failing).

* Registry resources are found in the `openshift-image-registry` project
  
  The registry uses several resources for configuration, deployment, and access...
  
  * Configuration
    
    Configuration of the registry instance is managed through the custom resource definition (CRD) `configs.imageregistry.operator.openshift.io`.  This object serves several purposes, including showing the current configuration of the registry and providing state/status information to the administrator.
    
    To view the object use the command `oc get -o yaml configs.imageregistry.operator.openshift.io/instance -n openshift-image-registry`
    
  * Deployment(s) and ReplicaSets
    
    The namespace contains two deployments.
    
    1. The first deployment, `cluster-image-registry-operator`, is the operator itself.  This is deployed and managed by the Cluster Version Operator (CVO) and should not need to be managed by the administrator.
    1. The second deployment, `image-registry` is created, and managed, by the image registry operator.  This deployment is created and managed by the operator and is managed indirectly by the administrator through configuration of the operator, as described above.
    
    To view the deployments, use the `oc get deployment -n openshift-image-registry` command.
    
    ```
    [you@bastion ~]$ oc get deployment -n openshift-image-registry
    NAME                              DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
    cluster-image-registry-operator   1         1         1            1           65m
    image-registry                    1         1         1            1           14m
    ```
    
    Exploring the `image-registry` deployment shows the configuration which is being applied.  The registry container image uses environment variables to self-configure, the deployment contains those variables, along with network ports, and the other expected configuration which will be applied to the pods deployed.
    
    To view the deployment configuration, use the command `oc get -o yaml deployment/image-registry -n openshift-image-registry`.
    
    Like other deployoments, a ReplicaSet is created to control and version the configuration of the pods.  Use the command `oc get replicaset -n openshift-image-registry` to view the ReplicaSets which have been created, including seeing which is the currently active set.
    
    ```
    [ansulliv-redhat.com@bastion ~]$ oc get replicaset -n openshift-image-registry
    NAME                                         DESIRED   CURRENT   READY   AGE
    cluster-image-registry-operator-6d6b45bfdf   1         1         1       3h33m
    image-registry-78d897f7bc                    0         0         0       163m
    image-registry-86bcc4dcf7                    0         0         0       163m
    image-registry-b7c59d799                     1         1         1       162m
    image-registry-d8987b84c                     0         0         0       163m
    ```
    
  * Pod(s)
    
    The image registry pod(s) can be viewed in the traditional manner: `oc get pod -n openshift-image-registry`

    ```
    [you@bastion ~]$ oc get pod -n openshift-image-registry
    NAME                                               READY   STATUS    RESTARTS   AGE
    cluster-image-registry-operator-6d6b45bfdf-rkhkd   1/1     Running   0          78m
    image-registry-b7c59d799-82xj4                     1/1     Running   0          27m
    node-ca-2gvqm                                      1/1     Running   0          24m
    node-ca-5l89x                                      1/1     Running   0          25m
    node-ca-9xv74                                      1/1     Running   0          24m
    node-ca-dtdst                                      1/1     Running   0          27m
    node-ca-dvlk9                                      1/1     Running   0          26m
    node-ca-x8zf2                                      1/1     Running   0          23m
    ```
    
    An `image-registry-xxx-xxx` pod will exist for each replica which has been configured (by default only a single replica exists).  The registry pods are deployed to *master* nodes, however the operator pods are deployed to *worker* nodes.
    
  * Secrets
    
    While deploying, the operator will request credentials to manage the S3 storage from the [Cloud Credential Operator](https://github.com/openshift/cloud-credential-operator).  These will be stored in a secret named `installer-cloud-credentials` in the `openshift-image-registry` namespace.
    
    The administrator can provide specific credentials for the registry user by creating a secret named `image-registry-private-configuration-user`.
    
    
    
  * Services
    
    The registry provides a single service which can be viewed using the command `oc get service -n openshift-image-registry`.
    
    ```
    [you@bastion ~]$ oc get service
    NAME             TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
    image-registry   ClusterIP   172.30.205.200   <none>        5000/TCP   159m
    ```
    
  * Routes
    
    By default, no route is created for accessing the image registry externally.  However, the administrator can request the operator create a route with a default naming convention, or they may specify a list of routes which the operator will create.
    
    To view the routes, use the `oc get route -n openshift-image-registry` command:

## Addtional materials

The image registry in OCP 4.x is managed by the [Image Registry operator](https://github.com/openshift/cluster-image-registry-operator/).  The most up-to-date information regarding the configuration, operation, and troubleshooting of the registry can be found from the documentation.

Some consolidated information regarding the registry is available in [this gist](https://gist.github.com/acsulli/c23469c68d4a0988f5d6c3f1c5be6977).  Including specific instructions on how to modify the configuration.