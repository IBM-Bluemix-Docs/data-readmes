![](https://raw.githubusercontent.com/ibm-cloud-docs/images/master/discovery-icp4d-new-banner.png)

# IBM Watson Discovery

## Introduction
IBM Watson™ Discovery for IBM® Cloud Pak is an AI-powered search and content analytics engine that finds answers and insights from complex business content with speed and accuracy. Answers can be surfaced to users through a conversational dialog driven by Watson Assistant or embedded in your own user-interface. With its Smart Document Understanding training interface, Watson Discovery can learn where answers live in complex business content based on a visual understanding of documents. Further enhance Watson Discovery's ability to understand domain specific language by teaching it with Watson Knowledge Studio.

Watson Discovery brings together a functionally rich set of integrated and automated Watson APIs to:
  - Convert, enrich, and normalize data.
  - Securely explore your proprietary content as well as free and licensed public content.
  - Apply additional enrichments such as keywords through Natural Language Understanding (NLU).
  - Simplify development while still providing direct access to APIs.

## Chart Details

This chart deploys a single Watson Discovery node with a default pattern. It includes the endpoints listed [here](https://cloud.ibm.com/apidocs/discovery/discovery-data-v2).


## Prerequisites

Before installing Watson Discovery, you must install:

  - IBM® Cloud Pak for Data 2.5.0.0 or 2.1.0.2
    - [IBM® Cloud Pak for Data 2.5.0.0 installation instructions](https://www.ibm.com/support/knowledgecenter/SSQNUZ_2.5.0/cpd/install/install.html). (Also see [IBM Cloud Pak for Data 2.5.0.0 release notes](https://www.ibm.com/support/knowledgecenter/SSQNUZ_current/cpd/overview/relnotes-2.5.0.0.html)).
    - [IBM® Cloud Pak for Data 2.1.0.2 installation instructions](https://www.ibm.com/support/knowledgecenter/SSQNUZ_2.1.0/com.ibm.icpdata.doc/zen/install/ovu.html)

**IMPORTANT**: Portworx **must** be installed before you install Watson Discovery. A Portworx license is included with IBM® Cloud Pak for Data 2.5.0.0.

## Limitations

- Watson Discovery must be deployed within the same namespace as IBM Cloud Pak for Data (by default the `zen` namespace).
- A Watson Discovery deployment supports a single service instance.
- Watson Discovery can currently run only on Intel 64-bit architecture.
- This chart should be used with the default image tags provided. Other image versions might be incompatible.
- This chart must be installed through the CLI.
- This chart must be installed by a ClusterAdministrator
- This chart currently does not support upgrades or rollbacks. See [Backing up and restoring data](https://cloud.ibm.com/docs/services/discovery-data?topic=discovery-data-backup-restore) for instructions.


## Resources Required

In addition to the [System requirements for IBM Cloud Pak for Data](https://www.ibm.com/support/producthub/icpdata/docs/content/SSQNUZ_current/cpd/plan/rhos-reqs.html), IBM Watson Discovery has the following requirements.

For installation (at minimum):

|                                                           |Disk        | Docker registry |
|-----------------------------------------------------------|:----------:|:---------------:|
| Discovery                                                 | 50 GB      | 45 GB           |
| Additional needed for Discovery for Content Intelligence  | 40 GB      | 20 GB           |


For use:

|                      | Minimum VPC available | Minimum RAM available |
|----------------------|:---------------------:|:---------------------:|
| Development (non-HA) | 16                    | 104 GB                |
| Production (HA)      | 24                    | 159 GB                |


## Storage

Parenthetical numbers are the PVs required/created when deploying with the recommended high availability (HA) configuration. See [High availability (Production) configuration](#high-availability-configuration) for more information.

| Component      | Number of replicas | Space per PVC | Storage type            |
|----------------|--------------------|---------------|-------------------------|
| Postgres       |               2(3) |         20 Gi | portworx-db-gp3     |
| Etcd           |                  3 |         10 Gi | portworx-db-gp3     |
| Elastic Master |               1(3) |          2 Gi | portworx-db-gp3     |
| Elastic Data   |               1(2) |         30 Gi | portworx-db-gp3     |
| Elastic Backup |                  1 |         60 Gi | portworx-shared-gp2 |
| Minio          |                  4 |         25 Gi | portworx-db-gp3     |
| Hdp worker     |                  2 |        100 Gi | portworx-db-gp3     |
| Hdp nn         |                  1 |         10 Gi | portworx-db-gp3     |
| Core common    |                  1 |        100 Gi | portworx-shared-gp2 |
| Core ingestion |                  1 |          1 Gi | portworx-shared-gp2 |

If the `portworx-db-gp3` storage class does not exist in your cluster, but Portworx is installed, you can create it using the following command:
```console
kubectl apply -f - <<END
allowVolumeExpansion: true
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: portworx-db-gp3
parameters:
  io_profile: db
  repl: "3"
provisioner: kubernetes.io/portworx-volume
reclaimPolicy: Delete
volumeBindingMode: Immediate
END
```

If the `portworx-shared-gp2` storage class does not exist in your cluster, but Portworx is installed, you can create it using the following command:
```console
kubectl apply -f - <<END
allowVolumeExpansion: true
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: portworx-shared-gp2
parameters:
  priority_io: high
  repl: "2"
  shared: "true"
provisioner: kubernetes.io/portworx-volume
reclaimPolicy: Delete
volumeBindingMode: Immediate
END
```

**Note:** Gluster File System (GlusterFS) is not a supported storage option for Watson Discovery.

## Documentation

IBM Watson Discovery documentation:

- [Watson Discovery for IBM Cloud Pak for Data](https://cloud.ibm.com/docs/services/discovery-data)
- [API reference](https://cloud.ibm.com/apidocs/discovery/discovery-data-v2)

IBM® Cloud Pak for Data:
- [IBM Cloud Pak for Data V2.5.0 Knowledge Center](https://www.ibm.com/support/knowledgecenter/SSQNUZ_2.5.0/cpd/overview/welcome.html)
- [IBM Cloud Pak for Data V2.1.0 Knowledge Center](https://www.ibm.com/support/knowledgecenter/SSQNUZ_2.1.0/com.ibm.icpdata.doc/zen/overview/welcome.html)


## Pre-install steps

### Setting up an OpenShift Environment

**NOTE:** Skip this section if you are deploying to IBM Cloud Private. See [Setting up an IBM Cloud Private environment](#setting-up-an-ibm-cloud-private-environment).

If you're deploying to an OpenShift cluster, `helm` will be installed and configured as part of running `deploy.sh`.

1. Log into the Kubernetes cluster and target the namespace or project the chart will be deploy to. Use the `oc` cli.

    ```bash
    oc login https://{cluster_hostname}:8443 -u {user} -p {password}
    ```

2. Log into the cluster's docker registry using either `docker` or `podman`

    ```bash
    docker login $(oc get routes docker-registry -n default -o template={{.spec.host}}) -u {user} -p {password}
    ```

    ```bash
    podman login $(oc get routes docker-registry -n default -o template={{.spec.host}}) -u {user} -p {password}
    ```

3. Download Discovery from Passport Advantage (if you haven't already) and create a directory to extract its content to:

    ```
    mkdir ibm-watson-discovery-ppa
    tar -xJf ibm-watson-discovery-2.1.2.tar.xz -C ibm-watson-discovery-ppa
    cd ibm-watson-discovery-ppa
    ```

   If you have purchased `Discovery for Content Intelligence` along with `Watson Discovery`, also download and extract the `Discovery for Content Intelligence PPA`:

   ```
   mkdir ibm-discovery-content-intelligence-ppa
   tar -xJf <ci-service-ppa-archive>  -C ibm-discovery-content-intelligence-ppa
   ```
4. If you're deploying to an OpenShift cluster, there is a good chance that you'll need to increase the system's virtual memory max map count. ElasticSearch will not start unless `vm.max_map_count` is greater than or equal to `262144`. This should be set automatically as part of the cluster provisioning script for IBM Cloud Private clusters, but will need to be done manually for an OpenShift cluster.

To make these changes, run `systctl -w vm.max_map_count=262144`, and then add `vm.max_map_count=262144` to `/etc/sysctl.conf` on each node.


### Setting up an IBM Cloud Private environment

**NOTE:** Skip this section if you are deploying to OpenShift. See [Setting up an OpenShift Environment](#setting-up-an-openshift-environment).

1. Log into the Kubernetes cluster and target the namespace or project the chart will be deploy to. Use the `cloudctl` cli.

    ```bash
    cloudctl login -a https://{cluster_hostname}:8443 -u {user} -p {password}
    ```


2. Log into the cluster's docker registry.

    ```bash
    docker login {cluster_hostname}:8500 -u {user} -p {password}
    ```

3. Download Discovery from Passport Advantage (if you haven't already) and create a directory to extract its content to:

    ```
    mkdir ibm-watson-discovery-ppa
    tar -xJf ibm-watson-discovery-2.1.2.tar.xz -C ibm-watson-discovery-ppa
    cd ibm-watson-discovery-ppa
    ```

   If you have purchased `Discovery for Content Intelligence` along with `Watson Discovery`, also download and extract the `Discovery for Content Intelligence PPA`:

   ```
   mkdir ibm-discovery-content-intelligence-ppa
   tar -xJf <ci-service-ppa-archive>  -C ibm-discovery-content-intelligence-ppa
   ```

### Configuring Firewall Rules

**NOTE** If your cluster is running with a firewall between nodes, you must complete this step. Otherwise, skip to the next section.

Perform the following tasks on every node of your cluster:
1. Fix the ports that lockd uses. Edit `/etc/sysconfig/nfs` as following.
   ```bash
     # TCP port rpc.lockd should listen on.
     LOCKD_TCPPORT=32803
     # UDP port rpc.lockd should listen on.
     LOCKD_UDPPORT=32769
   ```
2. Restart NFS services
   ```bash
     systemctl restart nfs-config
     systemctl restart nfs-server
   ```
3. Confirm the ports used by `lockd` have been updated. If not, reboot the node. To check it, run `rpcinfo -p` and see the output. For instance, the output should have a part which looks like
   ```bash
     100021    1   udp  32769  nlockmgr
     100021    3   udp  32769  nlockmgr
     100021    4   udp  32769  nlockmgr
     100021    1   tcp  32803  nlockmgr
     100021    3   tcp  32803  nlockmgr
     100021    4   tcp  32803  nlockmgr
   ```
4. Open required ports on firewall
   ```bash
     firewall-cmd --add-service=nfs --permanent
     firewall-cmd --add-service=mountd --permanent
     firewall-cmd --add-service=rpc-bind --permanent
     firewall-cmd --add-port=32803/tcp --permanent
     firewall-cmd --add-port=32769/udp --permanent
     firewall-cmd --reload
   ```

## Loading Watson Discovery Docker images into your container registry

Before you can install Watson Discovery, you must load all of the Docker images distributed as part of the PPA archive into the cluster's internal container registry using the `loadImages.sh` script from the `bin` subdirectory.

* Run `./loadImages.sh -h` for help.
* By default the script runs in non-interactive mode, so all arguments must be specified using command-line flags. Use `--interactive true` to run the script in interactive mode
* The two required arguments to `loadImages.sh` are
    * `--registry HOST_PORT`: The base URL of the registry you want to push the images to.
    * `--namespace NAMESPACE`: The namespace of the cluster you will be installing Watson Discovery in.
* The `loadImages.sh` script is compatible with both `docker` and `podman`. If both executables are installed and available on your PATH, Watson Discovery will always default to using
`docker`. To override that behavior to use `podman`, or any other compatible docker container engine, use the `--exe` flag.

### Loading Watson Discovery Docker images into an OpenShift container registry

```bash
  ./bin/loadImages.sh --registry $(oc get routes docker-registry -n default -o template={{.spec.host}}) --namespace {namespace}
```

### Loading Watson Discovery Docker images into an IBM Cloud Private container registry

```bash
  ./bin/loadImages.sh --registry {cluster_hostname}:8500 --namespace {namespace}
```


## Installing Watson Discovery

Installing Watson Discovery deploys a single Watson Discovery application into an IBM Cloud Pak environment. You can deploy to a `Development` or `Production` environment. **By default, Watson Discovery will install in `Development` mode.** See [High availability (Production) configuration](#high-availability-configuration) for instructions on deploying to `Production`.

To install Watson Discovery to your cluster, run the `installDiscovery.sh` script from the `bin` subdirectory.

* Run `./installDiscovery.sh -h` for help.
* By default the script runs in non-interactive mode, so all arguments must be specified using command-line flags. Use `--interactive true` to run the script in interactive mode.
* The required arguments to `installDiscovery.sh` are
    * `--cluster-pull-prefix PREFIX`: Specify where the cluster will pull Watson Discovery images from
    * `--api-host HOSTNAME`: The host name (do not include a port or scheme in this value) to the Kubernetes API Endpoint for your cluster. You can retrieve this using `kubectl cluster-info`
    * `--api-ip IP_ADDR`: The IPv4 address of the API host name provided to `--api-host`
    * `--namespace NAMESPACE`: The namespace you want to install Watson Discovery into
    * `--storageclass STORAGE_CLASS`: The name of the storage class to use for Watson Discovery's ReadWriteOnce storage. When using Portworx, Watson Discovery recommends `portworx-db-gp3`
    * `--shared-storageclass STORAGE_CLASS`: The name of the storage class to use for Watson Discovery's ReadWriteMany storage. When using Portworx, Watson Discovery recommends `portworx-shared-gp2`
* To install Watson Discovery in High Availability mode, append `--production` to your command.
* If you would like to change the value of a parameter from its default value, you must create a yaml file, overriding any values you want in it and then pass it into your command with `--override {override-file}.yaml`.

**If you've purchased and downloaded Discovery for Content Intelligence, you must install it now**:
  1. Use the values provided in `ci-override.yaml` file in the `ibm-discovery-content-intelligence-ppa` folder, i.e., run the deploy script with `--override ci-override.yaml`.  If you already have an `{override-file.yaml}`, first append these values of `ci-override.yaml` into your `<override-file.yaml>` and then run the deploy script with your `--override {override-file}.yaml`.
  2. After Discovery for Content Intelligence is installed, add the `Software Identification Tags` to the required pod: 

      ```bash
      oc cp ibm-discovery-content-intelligence-ppa/ibm.com_IBM_Watson_Discovery_for_Cloud_Pak_for_Data_Content_Intelligence-2.1.0.swidtag core-discovery-gateway-0:/swidtag/ -c management
      ```

**Note**: The `Discovery for Content Intelligence PPA` includes the tag file ending with `.swidtag`.

**Tip**: Run `-h` to view the list of available flags and help.

### High availability configuration

To deploy in High Availability (Production) mode,

1. Append `--production` to your `installDiscovery.sh` command

### Verifying the Watson Discovery installation

#### On Linux

From the `bin` subdirectory, run the command:

```bash
./linux/helm test core
```

#### On macOS

From the `bin` subdirectory, run the command:

```bash
./macos/helm test core
```

The installation is complete.


### Installing the optional language pack

The following enrichments are supported in English only, unless you download and install the language extension pack `ibm-wat-dis-pack1` from Passport Advantage. For a breakdown of supported languages, see [Language support](https://cloud.ibm.com/docs/services/discovery-data?topic=discovery-data-language-support).

- Entities
- Keywords
- Sentiment of documents

Prerequisite:
- Watson Discovery must be installed and running

To install `ibm-watson-discovery-pack1`:

1. Unpack the language pack.

```bash
$ mkdir ibm-watson-discovery-language-pack
$ cd ibm-watson-discovery-language-pack
$ mv ~/Downloads/ibm-watson-discovery-pack1-2.1.0.tar.xz .
$ tar xJf ibm-watson-discovery-pack1-2.1.0.tar.xz
$ ls .
bin lib LICENSE README.md RELEASENOTES.md
```
2. Verify you are authenticated with Docker and your cluster (`oc` on OpenShift, or `cloudctl` for IBM Cloud Private). (See [Setting up an OpenShift Environment](#setting-up-an-openshift-environment) or [Setting up an IBM Cloud Private environment](#setting-up-an-ibm-cloud-private-environment)).

3. Run `loadImages.sh` from the `bin` subdirectory using the `--registry` and `--namespace` as you did to load the Docker images for Watson Discovery previously.

**NOTE** The copy of `loadImages.sh` from ibm-watson-discovery-pack1 and ibm-watson-discovery are not interchangeable.

4. Run `installLanguagePack.sh` from the `bin` subdirectory. The required arguments are
    * `--cluster-pull-prefix PREFIX`: Specify where the cluster will pull Watson Discovery images from
    * `--namespace NAMESPACE`: The namespace Watson Discovery is currently installed in that you want to add the language pack to.


```bash
./installLanguagePack.sh --cluster-pull-prefix {registry}/{namespace} --namespace {namespace}
```

### Collecting OpenShift Support Logs

From the `bin` subdirectory of either the ibm-watson-discovery or ibm-watson-discovery-pack1 PPAs, run `./openshiftCollector.sh -c OPENSHIFT_CLUSTER_HOST -n ICP4D_NAMESPACE -u OPENSHIFT_ADMIN_USERNAME -p OPENSHIFT_ADMIN_PASSWORD`, where
- `OPENSHIFT_CLUSTER_HOST` is the cluster without the protocol/port
- `ICP4D_NAMESPACE` is the namespace where ICP4D is installed (usually `zen`).
- `OPENSHIFT_ADMIN_USERNAME` is the username for the openshift administrator
- `OPENSHIFT_ADMIN_PASSWORD` is the password for the openshift administrator

For more information and options, run `./openshiftCollector.sh --help`

This will produce a `.tgz` file in your current directory with the format `${OPENSHIFT_CLUSTER_HOST}_${DATE_TIME}.tgz`. This file will be what the support team can use to debug issues on the cluster.

You can follow the process of the log collection by running a command like `tail -f ./${OPENSHIFT_CLUSTER_HOST}_${DATE_TIME}/watson-diagnostics-data.log` in the directory that you started the command.

### Uninstalling Watson Discovery

#### Uninstalling Watson Discovery 2.1.2

To remove Watson Discovery from an OpenShift or IBM Cloud Private cluster, use the `uninstallDiscovery.sh` script provided in the `bin` subdirectory.

* Use `./uninstallDiscovery.sh -h` for help.
* The argument to `--namespace` should be the namespace Watson Discovery is running in

```bash
./uninstallDiscovery.sh --namespace {namespace}
```

By default this script will not remove persistent volume claims or specific secrets required to access or retrieve any data stored in Watson Discovery. To delete all objects associated with this instance of Watson Discovery, including any and all ingested data, include the `--force` flag.

```bash
./uninstallDiscovery.sh --namespace {namespace} --force
```

#### Uninstalling Watson Discovery 2.1.1, 2.1.0, 2.0.1, or 2.0.0

If you are upgrading from Watson Discovery 2.1.1 or earlier you must uninstall that version of Watson Discovery before you can install Watson Discovery 2.1.2. To delete the resources from a previous Watson Discovery installation named `my-release` in `my-namespace`, run this command:

```bash
kubectl delete --namespace=my-namespace all,configmaps,jobs,networkpolicies,persistentvolumeclaims,poddisruptionbudgets,roles,rolebindings,clusterroles,clusterrolebindings,secrets,serviceaccounts --selector=release=my-release
kubectl delete --namespace=my-namespace configmaps stolon-cluster-my-release-postgresql my-release.v1
```

**NOTE** You should run the backup scripts to export your data before performing this uninstallation procedure. Any data in Watson Discovery 2.1.1 will be deleted and unreachable after the uninstall completes. See the documentation on [backing up and restoring data](https://cloud.ibm.com/docs/services/discovery-data?topic=discovery-data-backup-restore) for more information.

## Security reference

**NOTE**: This information is provided for reference. The install will create the security requirements for you.

### PodSecurityPolicy Requirements

**NOTE**: This information is provided for reference. The install will create the security requirements for you.

This chart requires a PodSecurityPolicy to be bound to the target namespace prior to installation. The predefined PodSecurityPolicy name: [`ibm-restricted-psp`](https://ibm.biz/cpkspec-psp) has been verified for this chart.

These PodSecurityPolicy resources can also be created manually. A cluster admin can save these templates to separate yaml files and run the command below for each file:

```
kubectl create -f <some-resource->.yaml
```

Template for a PodSecurityPolicy definition, currently equivalent `ibm-restricted-psp`, to bind to and create your own RBAC resources.

Custom PodSecurityPolicy definition:
  ```yaml
apiVersion: extensions/v1beta1
kind: PodSecurityPolicy
metadata:
  annotations:
    kubernetes.io/description: This policy is the most restrictive, requiring pods
      to run with a non-root UID, and preventing pods from accessing the host.
    seccomp.security.alpha.kubernetes.io/allowedProfileNames: docker/default
    seccomp.security.alpha.kubernetes.io/defaultProfileName: docker/default
  name: ibm-discovery-custom-psp
spec:
  allowPrivilegeEscalation: false
  forbiddenSysctls:
  - '*'
  fsGroup:
    ranges:
    - max: 65535
      min: 1
    rule: MustRunAs
  requiredDropCapabilities:
  - ALL
  runAsUser:
    rule: MustRunAsNonRoot
  seLinux:
    rule: RunAsAny
  supplementalGroups:
    ranges:
    - max: 65535
      min: 1
    rule: MustRunAs
  volumes:
  - configMap
  - emptyDir
  - projected
  - secret
  - downwardAPI
  - persistentVolumeClaim
```

Template for custom Role resource to replace the default privileged role:
  ```yaml
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name:  discovery-custom-priv-role
rules:
- apiGroups: ["", "batch", "extensions"]
  resources: ["jobs", "jobs/status", "secrets", "pods", "pods/exec", "configmaps"]
  verbs: ["get", "watch", "create", "apply", "list", "update", "patch", "delete"]
- apiGroups: ["policy"]
  resources: ["podsecuritypolicies"]
  resourceNames: [ibm-discovery-custom-psp]
  verbs: ["use"]
- apiGroups: [""]
  resources: ["resourcequotas", "resourcequotas/status"]
  verbs: ["get", "list", "watch"]
```

To use a custom Role resource you must have custom ServiceAccount resource.

**NOTE:** You **must** replace <pull-secret-name> with the name of the image pull secret in the namespace of your cluster. For IBM Cloud Private Foundations that secret name is sa-<namespace-name>. For RedHat OpenShift you can find the secret name by running
```bash
oc get secrets \
  --namespace "${NAMESPACE}" \
  --output=jsonpath='{ range .items[*] }{@.metadata.name}{"\n"}{end}' \
| grep default-dockercfg \
| tr -d '[:space:]'
```

Template for custom ServiceAccount resource to replace the default privileged service account:
  ```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: discovery-custom-priv-service-account
imagePullSecrets:
- name: <pull-secret-name>
```

To bind your service accounts to your role you need to create a rolebinding.

Template for role binding your custom privileged service account to your custom privileged role:
  ```yaml
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name:  discovery-custom-priv-role-binding
subjects:
- kind: ServiceAccount
  name: discovery-custom-priv-service-account
roleRef:
  kind: Role
  name:  discovery-custom-priv-role
  apiGroup: rbac.authorization.k8s.io
```

Finally, you can specify the name of the custom service account when installing the chart. For example,if you want to override the privileged service account with `discovery-custom-priv-service-account`, add the following to your helm install command:

```
<...> --set global.privilegedServiceAccount.name=discovery-custom-priv-service-account --set global.privilegedServiceAccount.create=false --set global.privilegedRbac.create=false
```

### Red Hat OpenShift SecurityContextConstraints Requirements

**NOTE**: This information is provided for reference. The install will create the security requirements for you.

If running in a Red Hat OpenShift cluster, this chart requires a `SecurityContextConstraints` to be bound to the target namespace prior to installation. The predefined PodSecurityPolicy name: [`restricted`](https://ibm.biz/cpkspec-scc) has been verified for this chart.

The SecurityContextConstraint resource can also be created manually. A cluster admin can save this template to a yaml file and run the command below:

```
kubectl create -f ibm-discovery-prod-scc.yaml
```

Template for a SecurityContextConstraints definition, currently equivalent `restricted`, to bind to your namespace.

  * Custom `SecurityContextConstraints` definition:

  ```yaml
apiVersion: security.openshift.io/v1
kind: SecurityContextConstraints
metadata:
  annotations:
    kubernetes.io/description: "This policy is the most restrictive,
      requiring pods to run with a non-root UID, and preventing pods from accessing the host."
    cloudpak.ibm.com/version: "1.0.0"
  name: ibm-discovery-prod-scc
allowHostDirVolumePlugin: false
allowHostIPC: false
allowHostNetwork: false
allowHostPID: false
allowHostPorts: false
allowPrivilegedContainer: false
allowPrivilegeEscalation: false
allowedCapabilities: []
allowedFlexVolumes: []
allowedUnsafeSysctls: []
defaultAddCapabilities: []
defaultPrivilegeEscalation: false
forbiddenSysctls:
  - "*"
fsGroup:
  type: MustRunAs
  ranges:
  - max: 65535
    min: 1
readOnlyRootFilesystem: false
requiredDropCapabilities:
- ALL
runAsUser:
  type: MustRunAsNonRoot
seccompProfiles:
- docker/default
seLinuxContext:
  type: RunAsAny
supplementalGroups:
  type: MustRunAs
  ranges:
  - max: 65535
    min: 1
volumes:
- configMap
- downwardAPI
- emptyDir
- persistentVolumeClaim
- projected
- secret
priority: 0
```

After creating the scc, you can bind the SCC to the namespace with this command, replacing `$namespace` with the namespace you're deploying to:

```
oc adm policy add-scc-to-group ibm-discovery-prod-scc system:serviceaccounts:$namespace
```

## Configuration

Contact Support.

## Backup and Restore

This chart currently does not support upgrades or rollbacks. See [Backing up and restoring data](https://cloud.ibm.com/docs/services/discovery-data?topic=discovery-data-backup-restore) for instructions.

## Integration with other IBM Watson services

Watson Discovery is one of many IBM Watson services. Additional Watson services on IBM Cloud Pak for Data and the IBM Public Cloud allow you to bring Watson's AI platform to your business application, and to store, train, and manage your data in the most secure cloud.

For the full list of available Watson services, see:

- [IBM Cloud Pak for Data](https://www.ibm.com/products/cloud-pak-for-data)
- [IBM Watson Public Cloud catalog](https://cloud.ibm.com/catalog?category=ai)

Watson services are currently organized into the following categories for different requirements and use cases:

  - **Assistant**: Integrate diverse conversation technology into your application
  - **Empathy**: Understand tone, personality, and emotional state
  - **Knowledge**: Get insights through accelerated data optimization capabilities
  - **Language**: Analyze text and extract metadata from unstructured content
  - **Speech**: Convert text and speech with the ability to customize models
  - **Vision**: Identify and tag content, then analyze and extract detailed information found in images

_Copyright©  IBM Corporation 2020. All Rights Reserved._
