# hazelcast-operator

This project explain how to deply hazelcast as an operator in an Openshift Cluster.  
This will let us to offer one or more hazelcastClusters for different scopes.


  - [Prerequisites](#prerequisites)
  - [References](#references)
  - [Installation](#installation)
  	- [1) Hazelcast Platform Operator](#1-hazelcast-platform-operator)
  	- [2) Hazelcast Cluster](#2-hazelcast-cluster)
  	- [3) Hazelcast Cluster Manager (optional)](#3-hazelcast-cluster-manager-optional)  
  - [Check and Delete](#check-and-delete)


#### Prerequisites:
- Openshift 4.9+

#### References:
- https://docs.hazelcast.com/operator/latest/


###### Notes
- Using this procedure will fix some rbac issue related to FHIR when using autodetection of an hazelcastcluster, the RoleBinding it's created by this procedure in the right namespace.
- All the components are installed with NO PERSISTANCE, it is possibile to configure it.  
If you persist hazelcast cluster manager read [here](#Things-to-know-when-you-enable-persistence-to-the-ManagementCenter)


## Installation:
This is the procedure to install the Hazelcast Platform Operator and to instantiate a working cluster, the following components will be installed and configured:
1. Hazelcast Platform Operator
2. Hazelcast Cluster
3. Hazelcast Cluster Manager (optional)

- All this object will be installed with NO PERSISTANCE  
- A RoleBinding will be attached to the ServiceAccount "default", in this way the services using this ServiceAccount can perform an autodiscovery of the HazelCast
- A specific Dedalus Secret will be attached to the needed ServiceAccounts for pulling the images

### 1) Hazelcast Platform Operator

Following this procedure will install the certified Hazelcast Platform Operator  

use this command:
```
NAMESPACE=@type_here_the_namespace@

oc process -f ./deploy/hazelcastoperator.template.yml -p NAMESPACE=${NAMESPACE} | oc -n ${NAMESPACE} create -f -
```
here an example:
```
NAMESPACE=hazelcast

oc process -f ./deploy/hazelcastoperator.template.yml -p NAMESPACE=${NAMESPACE} | oc -n ${NAMESPACE} create -f -
```
Cause we define the InstallPlan as "Manual" we need to patch it to complete the installation you can use this command to do it:
```
oc patch InstallPlan/$(oc get --no-headers  InstallPlan | grep hazelcast-platform-operator | cut -d' ' -f1) --type merge --patch='{"spec":{"approved":true}}' -n ${NAMESPACE}
```



### 2) Hazelcast Cluster
Following this procedure will istantiate an HazelCast Cluster.

use this command:
```
NAMESPACE=@type_here_the_namespace@

oc process -f./deploy/hazelcastcluster.template.yml -p NAMESPACE=${NAMESPACE} | oc -n ${NAMESPACE} create -f -
```

here an example:

```
NAMESPACE=hazelcast

oc process -f ./deploy/hazelcastcluster.template.yml -p NAMESPACE=${NAMESPACE} | oc -n ${NAMESPACE} create -f -
```
---
**NOTE:**  _Here the list of parameters accepted by this template and their default._
```
parameters:
- name: NAMESPACE
  displayName: Namespace where the grafana Operator will be installed in
  description: Type the Namespace where the grafana Operator will be installed in
  required: true
  value:
- name: HAZ_SERVER_NAME
  displayName: Hazelcast StatefulSet Name
  description: Type the name for the StatefulSet created by the operator
  required: true
  value: hazelcast-node
- name: HAZ_CLUSTER_NAME
  displayName: Hazelcast Cluster Name
  description: Type the Name for the Hazelcast Cluster created by the members of the statefulset
  required: true
  value: x1v1fhir_terminology
- name: AWS_TOKEN
  displayName: Secret for AWS token
  description: Secret used to store the AWS token. default value for Openshift 4.x
  required: true
  value: aws-ecr-token
```


---

This template will create a RoleBinding to the service account _default_, this will let hazelcast client to use the automatic discovery if its own pod is using the sa _default_.
The clusterrole _hazelcast-cluster-role_ will be created by the operator itself

```
#Extracted from hazelcastcluster.template.yml
- apiVersion: rbac.authorization.k8s.io/v1
  kind: RoleBinding
  metadata:
    name: hazelcast-cluster-role-binding
    namespace: ${NAMESPACE}
    labels:
      app: hazelcast-dedalus
  roleRef:
    apiGroup: rbac.authorization.k8s.io
    kind: ClusterRole
    name: hazelcast-cluster-role
  subjects:
    - kind: ServiceAccount
      name: default
      namespace: ${NAMESPACE}
```
The template will add the secret _aws-ecr-token_ to the service account created by the hazelcast cluster, it will let the service account downloads the images from the dedalus repository.

```
#Extracted from hazelcastcluster.template.yml
- apiVersion: v1   
  kind: ServiceAccount
  metadata:  
    name: ${HAZ_SERVER_NAME}
    namespace: ${NAMESPACE}
  imagePullSecrets:
    - name: ${AWS_TOKEN}
  secrets:
    - name: ${AWS_TOKEN}`
```
_The service account name depends on the value of HAZ_SERVER_NAME parameter_


### 3) Hazelcast Cluster Manager (optional)

Following this procedure will instantiate an HazelCast Cluster Manger.

use this command:
```
NAMESPACE=@type_here_the_namespace@

oc process -f ./deploy/hazelcastclustermanager.template.yml -p NAMESPACE=${NAMESPACE} | oc -n ${NAMESPACE} create -f -
```
here an example:
```
NAMESPACE=hazelcast

oc process -f ./deploy/hazelcastclustermanager.template.yml -p NAMESPACE=${NAMESPACE} | oc -n ${NAMESPACE} create -f -
```
---
**NOTE:** _Here the list of parameters accepted by this template and their default._
```
parameters:
- name: NAMESPACE
  displayName: Namespace where the grafana Operator will be installed in
  description: Type the Namespace where the grafana Operator will be installed in
  required: true
  value:
- name: HAZ_SERVER_NAME
  displayName: Hazelcast StatefulSet Name
  description: Type the name for the StatefulSet created by the operator
  required: true
  value: hazelcast-node
- name: HAZ_CLUSTER_NAME
  displayName: Hazelcast Cluster Name
  description: Type the Name for the Hazelcast Cluster created by the members of the statefulset
  required: true
  value: x1v1fhir_terminology
- name: AWS_TOKEN
  displayName: Secret for AWS token
  description: Secret used to store the AWS token. default value for Openshift 4.x
  required: true
  value: aws-ecr-token
```
---

The template will add the secret _aws-ecr-token_ to the service account created by the hazelcast cluster manager, it will let the service account downloads the images from the dedalus repository.

```
#Extracted from hazelcastclustermanager.template.yml
- apiVersion: v1
  kind: ServiceAccount
  metadata:
    name: managementcenter-hazelcast
    namespace: ${NAMESPACE}
  imagePullSecrets:
    - name: ${AWS_TOKEN}
  secrets:
    - name: ${AWS_TOKEN}
```
###### Notes about persistance storage and the ManagementCenter
- with no persistance at the first login you will create a local admin, you have to do so all the times that you recreate the pod.
- When you enable the persistent storage the Manager Center will have some issue to restart due to a lock on the filesystem.  
Please refer to this link for additional information:
https://docs.hazelcast.com/management-center/4.2021.06/configuring#skipping-lock-file-check

## Check and Delete:

```
oc get $(oc api-resources --verbs=list -o name | awk '{printf "%s%s",sep,$0;sep=","}')  --ignore-not-found --all-namespaces -o=custom-columns=KIND:.kind,NAME:.metadata.name,NAMESPACE:.metadata.namespace --sort-by='.kind'  -l app=hazelcast-dedalus
```
Here an example of the output, you can ignore the Warnings  
NOTE: The RoleBinding is showed 2 times but it is only none

```
Warning: v1 ComponentStatus is deprecated in v1.19+
Warning: policy/v1beta1 PodSecurityPolicy is deprecated in v1.21+, unavailable in v1.25+
KIND               NAME                             NAMESPACE
Hazelcast          hazelcast-node                   hazelcast
ManagementCenter   managementcenter-hazelcast       hazelcast
OperatorGroup      hazelcast-operator-group         hazelcast
RoleBinding        hazelcast-cluster-role-binding   hazelcast
RoleBinding        hazelcast-cluster-role-binding   hazelcast
ServiceAccount     hazelcast-node                   hazelcast
ServiceAccount     managementcenter-hazelcast       hazelcast
Status             <none>                           <none>
Subscription       hazelcast-operator               hazelcast
```
Here the list of command to delete all the objects
```
NAMESPACE=hazelcast

oc delete managementcenter managementcenter-hazelcast -n ${NAMESPACE}
oc delete hazelcast hazelcast-node -n ${NAMESPACE}
oc delete ServiceAccount hazelcast-node  -n ${NAMESPACE}
oc delete ServiceAccount managementcenter-hazelcast  -n ${NAMESPACE}
oc delete RoleBinding hazelcast-cluster-role-binding -n ${NAMESPACE}
oc delete OperatorGroup hazelcast-operator-group -n ${NAMESPACE}
oc delete Subscription hazelcast-operator -n ${NAMESPACE}
oc delete clusterserviceversion/$(oc get --no-headers clusterserviceversion | grep  hazelcast-platform-operator | cut -d' ' -f1) -n ${NAMESPACE}
oc delete InstallPlan/$(oc get --no-headers  InstallPlan | grep hazelcast-platform-operator | cut -d' ' -f1) -n ${NAMESPACE}
```  
