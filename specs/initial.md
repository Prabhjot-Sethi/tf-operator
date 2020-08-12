# 1. Introduction
Tungsten Fabric Controller has evolved over the time and has already moved
towards microservices architecture, where all the components of TF Controller
runs as containers/pods extending functionalities as CNI provider and/or
neutron network provider.

As part of which evolution of mulitple deployment methods also happened for
different types of deployment using:
* docker compose: with ansible playbooks or juju charms
* Kubernetes manifest: for kubernetes native rollout (HA not supported)
* Helm Charts: for OpenStack-Helm and Airship integration (HA not supported)

# 2. Problem statement
Deployment of Tungsten Fabric with Kubernetes native objects (pods/daemonsets
etc.) in the form of manifests or helm charts was always limited as it lacked
the control needed to ensure a staged rollout and upgrade process ensuring the
inter-dependencies of Tungsten Fabric components. Due to this extending HA has
been always a limitation with kubernetes native rollout.

#### Requirements
With the problems identified it is desired to extend user-friendly lifecycle
mangement support for Tungsten Fabric controller, that allows to achieve
* Converged deployment in Kubernetes environment
  * Operator to replace helm-chart for Tungsten fabric and Kubernetes manifest
  * Operator to handle deployment in any Kubernetes based environment like
Openstack-helm and Airship.
  * Essentially operator to be not limited to CNI solution
* Allow provisions for Operator lifecycle manager catalog and install plan
approach
  * Allowing hosting operator on operatorhub
* Deployment
  * Operator should be capable of deploying TF cluster with High Availability
  * Ease of deployment, Tungsten Fabric rollout needs to be simplified as much
as possible by reducing information needed to rollout the deployment. Allowing
faster adoption of Tungsten Fabric
* Upgrade
  * Upgrade should follow OLM (/like) process, no extra information to be
required from administrator during upgrade, only an operator catalog (operator
image, deployment and rbac) update should be enough to rollout the new release
  * Allow patch release, rolling out only a specific component during upgrade,
for example a release providing a patch only to config subsystem however other
components/project remaining unaffected.
* Internal dependency management while deployment or upgrade
  * Tungsten Fabric has hard dependencies with respect to the order in which
components are rollout or upgraded, operator need to ensure the process being
followed
  * On high level following order needs to be ensured
    * Infra (Zookeeper/Cassandra/Redis/Kafka/RabbitMQ)
    * Config nodes
    * Control nodes
    * Datapath components
    * Analytics and web-ui
* Auto-Scaling
  * Unless overridden with configuration default approach should be auto-scale,
where any master node joining can be considered to assume role controller role
running all the tungsten fabric controller components, initially we can limit
auto scale only to control nodes, web-ui and analytics.
  * More advanced logic can be inducted at later stages

# 3. Proposed solution
Operator Framework provides an open source toolkit (operator-sdk) to manage
kubernetes native applications, in an effective, automated and scalable way.

Tungsten Fabric operator based on operator sdk uses kubernetes
controller-runtime to extend a deployment manager/operator. where is defines
and consumes a CRD indicating deployment parameters to generating deployment
corresponding to it.

## 3.1 Alternatives considered
Already available alternatives kubernetes manifest and Helm chart lacks the
capability of addressing inter-dependencies of Tungsten Fabric components.

## 3.2 API schema changes
This Does not require any changes to Tungsten Fabric API schema.

## 3.3 User workflow impact
This solution is limited to deployment, updrage, cluster scaling events and
otherwise does not impact user workflow in anyway.

## 3.4 UI changes
NA

## 3.5 Notification impact
NA

# 4. Implementation
Operator framework constitute of CRDs / K8s APIs and Controllers to consume
generated CRDs. Since this operator is going to extend the deployment
functionality, implementation will also capture individual manifests for
tungsten fabric components.

## 4.1 Custom Resource Definitions
we look forward to begin with usage of one CRD, which defines the priliminary
deployment configuration for a cluster. This CRD will avoid any information
that operator can itself derived based on the release or based on the context
of the deployment itself.
sample cluster-wide CR could look like
```
apiVersion: tungsten.io/v1alpha1
kind: installation
metadata:
  name: default # only name that will be handled by the operator
spec:
  # can be provided to allow dev-env usage with custom build tag
  # however should be deprecated/not-supported for non-dev-env
  releaseTag: auto
  analyticsConfig:
    dbEnable: true
    alarmEnable: true
    snmpEnable: true
  authConfig:
    mode: no-auth
    password: tungsten
  cniConfig:  # section for kubernetes specific config
    clusterName: k8s
    ipFabricNetwork:
      cidr: 10.64.0.0/12
    podNetwork:
      cidr: 10.32.0.0/12
    serviceNetwork:
      cidr: 10.96.0.0/12
    ipForwarding: snat
    useHostNewtorkService: true
# additional configuration to include OpenStack rollout
```
where as information pertaining to following can be auto derived
* Release Tag: statically bound to an operator container image
* Node Roles: automatically map all the controller roles to master
nodes and datapath to all the nodes in cluster
* HostOS: avoid using HostOS config, by building kernel module during
deployment for any HostOS. however documentation can be provided for
recommended/supported kernel versions.
* DeploymentStrategy: detailed configuration on RollingUpdate should
either be left to default or managed internally as per need.

## 4.2 Controller
Have a controller corresponding to consume the above mentioned CRD,
which will be responsible for the actual lifecycle management of
Tungsten Fabric. This controller will have logic to do a staged
rollout or upgrade, ensuring the inter-component dependencies of
Tungsten Fabric are addressed. Every stage will have a list of
manifests that needs to be applied.

## 4.3 Tungsten Fabric Manifests
Import predefined manifest files to be used as is with teaks for
the configuration.

# 5. Performance and scaling impact
No impact on Tungsten Fabric controller performance or scaling

# 6. Upgrade
Simplifies upgrade process with operator lifecycle manager.

#### upgrade process
tungsten fabric operator will follow a different release/build scheme denoted
with a label of format va.b.c (like v0.0.1), where each label is bound with a
supported release, upgrading the cluster will require to move to the newer
label of the operator.

label format:
* "a" represents the version of crds 0 in our case to begin with v1alpha1.
* increment in "b" represents the accomodating newer release of tungsten fabric
controller, for example 0 points to R2011 and 1 points to R2105
* "c" represents a patch release of operator to include a fix into operator or
to allow moving to service release of tungsten fabric

# 7. Deprecations
Deprecates K8s manifest and helm based installation/upgrade process

# 8. Dependencies
There are no dependencies for this feature.

# 9. Testing

# 10. Documentation Impact
Tungsten Fabric documentaion needs to include the details on lifecycle
management using tungsten fabric operator.

# 11. References
[Tungsten Fabric Architecture](https://tungstenfabric.github.io/website/Tungsten-Fabric-Architecture.html)

[Operator Framework - SDK](https://github.com/operator-framework/operator-sdk)

[Kubernetes controller-runtime](https://github.com/kubernetes-sigs/controller-runtime)
