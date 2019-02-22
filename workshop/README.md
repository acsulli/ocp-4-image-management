# Image Registry Workshop


## Deployment status

The registry deployment is controlled by the operator, however the administrator can request that the operator deploy/undeploy the registry by modifying the configuration:

```shell
# undeploy the registry
oc patch configs.imageregistry.operator.openshift.io/instance -n openshift-image-registry --type merge --patch '{ "spec": { "managementState": "Removed" } }'

# deploy and manage the registry
oc patch configs.imageregistry.operator.openshift.io/instance -n openshift-image-registry --type merge --patch '{ "spec": { "managementState": "Managed" } }'
```

The status of the image registry is reported as a part of the `configs.imageregistry.operator.openshift.io/instance` object.  By viewing this object we can see a detailed description of the status and any errors which may be happening.

```console
[ansulliv-redhat.com@bastion ~]$ oc get configs.imageregistry.operator.openshift.io/instance  -o yaml -n openshift-image-registry
apiVersion: imageregistry.operator.openshift.io/v1
kind: Config
[...]
status:
  conditions:
  - lastTransitionTime: 2019-02-18T16:16:00Z
    message: Deployment has minimum availability
    status: "True"
    type: Available
  - lastTransitionTime: 2019-02-22T20:17:19Z
    message: Everything is ready
    status: "False"
    type: Progressing
  - lastTransitionTime: 2019-02-18T15:34:58Z
    status: "False"
    type: Failing
  - lastTransitionTime: 2019-02-18T15:34:58Z
    status: "False"
    type: Removed
  - lastTransitionTime: 2019-02-18T16:15:41Z
    reason: S3 Bucket Exists
    status: "True"
    type: StorageExists
  - lastTransitionTime: 2019-02-18T16:15:41Z
    message: UserTags were successfully applied to the S3 bucket
    reason: Tagging Successful
    status: "True"
    type: StorageTagged
  - lastTransitionTime: 2019-02-18T16:15:41Z
    message: Default encryption was successfully enabled on the S3 bucket
    reason: Encryption Successful
    status: "True"
    type: StorageEncrypted
  - lastTransitionTime: 2019-02-18T16:15:41Z
    message: Default cleanup of incomplete multipart uploads after one (1) day was
      successfully enabled
    reason: Enable Cleanup Successful
    status: "True"
    type: StorageIncompleteUploadCleanupEnabled
[...]
```

If the registry has successfully deployed, there will be one (or more, depending on the number of replicas configured) `image-registry-xxx-xxx` pods in the `openshift-image-registry` namespace.

```console
[you@bastion ~]$ oc get pod
NAME                                               READY   STATUS    RESTARTS   AGE
image-registry-b7c59d799-qzgvm                     1/1     Running   2          4d2h
```

### Identifying errors

If the registry is not deployed, there can be a number of reasons.  

1. First, ensure that the Image Registry Operator has been successfully deployed by the Cluster Version Operator (CVO).
   
   ```console
   [you@bastion ~]$ oc get clusteroperator | grep image
   cluster-image-registry-operator         v4.0.0-0.150.0.0-dirty   True        False         False     4d
   ```
   
   If this operator is not listed, then something has gone wrong with the CVO, which much be remedied before the image registry operator will deploy.
   
1. Check the status of the operator deployment
   
   The CVO will create the deployment for the image registry operator.  Check the status of this deployment and it's pod(s) to ensure that the operator has deployed successfully.

   ```console
   [you@bastion ~]$ # check for the deployment status
   [you@bastion ~]$ oc get deployment cluster-image-registry-operator -o yaml -n openshift-image-registry
   apiVersion: extensions/v1beta1
   kind: Deployment
   [...]
   status:
     availableReplicas: 1
     conditions:
     - lastTransitionTime: 2019-02-18T15:24:57Z
       lastUpdateTime: 2019-02-18T15:24:57Z
       message: Deployment has minimum availability.
       reason: MinimumReplicasAvailable
       status: "True"
       type: Available
     - lastTransitionTime: 2019-02-18T15:24:39Z
       lastUpdateTime: 2019-02-18T15:24:57Z
       message: ReplicaSet "cluster-image-registry-operator-xxx" has successfully
         progressed.
       reason: NewReplicaSetAvailable
       status: "True"
       type: Progressing
     observedGeneration: 1
     readyReplicas: 1
     replicas: 1
     updatedReplicas: 1
   [you@bastion ~]$ 
   [you@bastion ~]$ # check for the operator pods
   [you@bastion ~]$ oc get pods -n openshift-image-registry | grep cluster-image-registry-operator
   cluster-image-registry-operator-xxx-xxx   1/1     Running   2          4d5h
   [you@bastion ~]$ 
   [you@bastion ~]$ # check the logs of the operator
   [youm@bastion ~]$ oc logs cluster-image-registry-operator-xxx-xxx -n openshift-image-registry
   I0222 15:34:42.776028       1 main.go:24] Cluster Image Registry Operator Version: v4.0.0-0.150.0.0-dirty
   I0222 15:34:42.776874       1 main.go:25] Go Version: go1.10.3
   I0222 15:34:42.776882       1 main.go:26] Go OS/Arch: linux/amd64
   [...]
   ```
   
   If the logs report any errors, they will need to be addressed before the image registry deployment can succeed.
   
1. Check the image registry deployment and pod logs
   
   ```console
   [you@bastion ~]$ # check for the deployment status
   [you@bastion ~]$ oc get deployment image-registry -o yaml -n openshift-image-registry
   apiVersion: extensions/v1beta1
   kind: Deployment
   [...]
   status:
     availableReplicas: 1
     conditions:
     - lastTransitionTime: 2019-02-18T16:14:58Z
       lastUpdateTime: 2019-02-18T16:15:59Z
       message: ReplicaSet "image-registry-xxx" has successfully progressed.
       reason: NewReplicaSetAvailable
       status: "True"
       type: Progressing
     - lastTransitionTime: 2019-02-22T15:35:23Z
       lastUpdateTime: 2019-02-22T15:35:23Z
       message: Deployment has minimum availability.
       reason: MinimumReplicasAvailable
       status: "True"
       type: Available
     observedGeneration: 6
     readyReplicas: 1
     replicas: 1
     updatedReplicas: 1
   [you@bastion ~]$ 
   [you@bastion ~]$ # check for the operator pods
   [you@bastion ~]$ oc get pods -n openshift-image-registry | grep image-registry | grep -v operator
   image-registry-xxx-xxx                     1/1     Running   2          4d2h
   [you@bastion ~]$ 
   [you@bastion ~]$ # check the logs of the image registry pod
   [you@bastion ~]$ oc logs image-registry-xxx-xxx -n openshift-image-registry
   time="2019-02-22T15:34:54.180978294Z" level=info msg="start registry" distribution_version=v2.6.0+unknown go.version=go1.9.4 openshift_version=v4.0.0-0.149.0
   time="2019-02-22T15:34:54.183078214Z" level=info msg="caching project quota objects with TTL 1m0s" go.version=go1.9.4
   time="2019-02-22T15:34:54.197635145Z" level=info msg="redis not configured" go.version=go1.9.4
   time="2019-02-22T15:34:54.200792265Z" level=info msg="Starting upload purge in 11m0s" go.version=go1.9.4
   time="2019-02-22T15:34:54.250996677Z" level=info msg="using openshift blob descriptor cache" go.version=go1.9.4
   [...]
   ```

This combination of steps should identify any deployment errors which may be happening in your environment.

## Customizing the registry deployment

There are many reasons why the registry may need to be customized, ranging from providing your own certificates to modifying storage parameters.  The steps below lay out modifying the most comon aspects of the OpenShift Container Registry when it's managed by the Image Registry Operator.

### Configure a proxy

```
oc patch configs.imageregistry.operator.openshift.io/instance -n openshift-image-registry --type merge --patch '
    {
      "spec": {
        "proxy": {
          "http": "proxy.company.tld:80",
          "https": "proxy.company.tld:443"
        }
      }
    }'
```

### Replicas

By default, the operator creates a single instance of the registry.  If you want to have more instances, modify the `spec.replicas` setting to your desired value:

```shell
oc patch configs.imageregistry.operator.openshift.io/instance -n openshift-image-registry --type merge --patch '{ "spec": { "replicas": 3 } }'
```

Remember that instances are deployed to master nodes by default.

### Storage

As of OCP 4.0, S3 is the only supported storage type for the image registry.

#### S3 Credentials

Customizing the S3 credentials for the registry deployment can be done using a secret.  The operator will request credentials from the Cloud Credential Operator, however if you have an existing bucket which you want to use with this registry deployment, providing different credentials may be necessary.  

Creating the secret is straightforward and handled like any other secret:

```shell
export ACCESSKEY=<AWS Access Key>
export SECRETKEY=<AWS Secret Key>
oc create secret generic image-registry-private-configuration-user \
    -n openshift-image-registry \
    --from-literal=REGISTRY_STORAGE_S3_ACCESSKEY=${ACCESSKEY} \
    --from-literal=REGISTRY_STORAGE_S3_SECRETKEY=${SECRETKEY}
```

#### Specify the S3 region

If you need to provide a region for your S3 bucket for any reason, modifying the configuration is a simple patch to the configuration:

```shell
oc patch configs.imageregistry.operator.openshift.io/instance -n openshift-image-registry --type merge --patch '{ "spec": { "storage": "s3": { "region": "us-east-1" } } }'
```

#### Specify an existing bucket

Specifying a custom bucket, whether a new bucket or existing, can be done using the config:

```shell
oc patch configs.imageregistry.operator.openshift.io/instance -n openshift-image-registry --type merge --patch '{ "spec": { "storage": "s3": { "bucket": "my-registry-bucket" } } }'
```

### Specifying API request limits

```shell
oc patch configs.imageregistry.operator.openshift.io/instance -n openshift-image-registry --type merge --patch '
    {
    "spec": {
        "requests": {
          "read": {
            "maxrunning": 10,
            "maxinqueue": 10,
            "maxwaitinqueue": "2m"
          },
          "write": {
            "maxrunning": 10,
            "maxinqueue": 10,
            "maxwaitinqueue": "2m"
          }
        }
      }
    }'
```

### Routes

#### Default route

To create the default route, edit the configuration for the registry operator.  This can be done using the command `oc edit configs.imageregistry.operator.openshift.io/instance -n openshift-image-registry`, or using a patch to edit in one command:

```shell
oc patch configs.imageregistry.operator.openshift.io/instance \
    -n openshift-image-registry \
    --type merge \
    --patch '{ "spec": { "defaultRoute": "true" } }'
```

#### Custom routes

See here on the [demo page](../demos/README.md#accessing-the-registry-externally).

## Using the registry

### Red Hat registry authentication

TODO: this section

### Providing additional CAs for upstream registries

TODO: this section

### Migrating embedded registry image references

```shell
oc adm migrate image-references registry.svc.ci.openshift.org/*=docker-registry.default.svc:5000/* --confirm
```

