# IBM Watson Knowledge Studio

IBM Watson Knowledge Studio is an application that enables developers and domain experts to collaborate on the creation of custom annotator components that can be used to identify mentions and relations in unstructured text.

# Introduction

## Summary

Use IBM Watson™ Knowledge Studio (WKS) to create a machine learning model that understands the linguistic nuances, meaning, and relationships specific to your industry or to create a rule-based model that finds entities in documents based on rules that you define.

To become a subject matter expert in a given industry or domain, Watson must be trained. You can facilitate the task of training Watson with Knowledge Studio.

## Features

- Intuitive: Use a guided experience to teach Watson nuances of natural language without writing a single line of code
- Collaborative: SMEs work together to infuse domain knowledge in cognitive applications
- Cost effective: Create and deploy domain knowledge infused annotators faster than ever before using an integrated development environment

# CASE Details

This CASE provides a helm chart of IBM Watson Knowledge Studio.

## Chart Details

This chart installs an IBM Cloud Pak for Data (CP4D) add-on of Watson Knowledge Studio. Once the installation is completed, Watson Knowledge Studio add-on becomes available on CP4D console.

This chart deploys the following microservices per install:

- **Watson Knowledge Studio Front-end**: Provides the WKS tooling user interface.
- **Service Broker**: Manages provision/de-provision instances.
- **Dispatcher**: Dispatches requests from gateway to Watson Knowledge Studio Front-end.
- **SIREG**: Tokenizers/parsers by Statistical Information and Relation Extraction (SIRE).
- **SIRE Job queue**: Machine Learning Training framework that allows to queue and schedule jobs on Kubernetes.
- **SIRE Train Facade**: Manages interaction with the training framework and Minio storage.
- **Model Management API**: Manages interaction with WKS Front-end and Train Facade.
- **Watson Gateway**: Publishes WKS service as CP4D add-on and handles incoming requests.
- **Glimpse**: Provides dictionary suggestions feature with model builder and serve runtime.
- **Advanced Rule Editor**: Provides Advanced Rule Editor (ARE) UI and runtime.

This chart installs the following stores:

- **PostgreSQL**: Stores training metadata.
- **MongoDB**: Stores WKS project data.
- **Minio**: Stores ML models and training data.
- **etcd**: Stores Glimpse model state. This is not deployed when dictionary suggestions feature is disabled.

## Prerequisites

- IBM Cloud Pak for Data 2.1 or 2.5
- Kubernetes 1.11 or later
- Helm 2.9.0 or later
- [`PodDisruptionBudgets`](https://kubernetes.io/docs/tasks/run-application/configure-pdb/) are recommended for high resiliency in an application during risky operations, such as draining a node for maintenance or scaling down a cluster.

### PodSecurityPolicy Requirements

### ICP PodSecurityPolicy Requirements

This chart requires a `PodSecurityPolicy` to be bound to the target namespace prior to installation. The predefined `PodSecurityPolicy` name: `ibm-restricted-psp` has been verified for this chart, if your target namespace is bound to this `PodSecurityPolicy` resource you can proceed to install the chart.

- ICPv3.1 - Predefined  PodSecurityPolicy name: [`ibm-restricted-psp`](https://ibm.biz/cpkspec-psp)
- Custom PodSecurityPolicy definition:

  ```yaml
  apiVersion: extensions/v1beta1
  kind: PodSecurityPolicy
  metadata:
    annotations:
      kubernetes.io/description: "This policy is the most restrictive,
        requiring pods to run with a non-root UID, and preventing pods from accessing the host."
      #apparmor.security.beta.kubernetes.io/allowedProfileNames: runtime/default
      #apparmor.security.beta.kubernetes.io/defaultProfileName: runtime/default
      seccomp.security.alpha.kubernetes.io/allowedProfileNames: docker/default
      seccomp.security.alpha.kubernetes.io/defaultProfileName: docker/default
    name: ibm-watson-ks-psp
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

### Red Hat OpenShift SecurityContextConstraints Requirements

This chart requires a `SecurityContextConstraints` (SCC) to be bound to the target namespace prior to installation. To meet this requirement there may be cluster scoped as well as namespace scoped pre and post actions that need to occur.

The default `SecurityContextConstraints` name: [`restricted`](https://ibm.biz/cpkspec-scc) has been verified for this chart, if your target namespace is bound to this `SecurityContextConstraints` resource you can proceed to install the chart.

- Custom SecurityContextConstraints definition:

  ```yaml
  apiVersion: security.openshift.io/v1
  kind: SecurityContextConstraints
  metadata:
    annotations:
      kubernetes.io/description: "restricted denies access to all host features and requires
        pods to be run with a UID, and SELinux context that are allocated to the namespace.  This
        is the most restrictive SCC and it is used by default for authenticated users."
    name: ibm-watson-ks-scc
  allowHostDirVolumePlugin: false
  allowHostIPC: false
  allowHostNetwork: false
  allowHostPID: false
  allowHostPorts: false
  allowPrivilegeEscalation: true
  allowPrivilegedContainer: false
  allowedCapabilities: null
  defaultAddCapabilities: null
  fsGroup:
    type: MustRunAs
  groups:
  - system:authenticated
  priority: null
  readOnlyRootFilesystem: false
  requiredDropCapabilities:
  - KILL
  - MKNOD
  - SETUID
  - SETGID
  runAsUser:
    type: MustRunAsRange
  seLinuxContext:
    type: MustRunAs
  supplementalGroups:
    type: RunAsAny
  users: []
  volumes:
  - configMap
  - downwardAPI
  - emptyDir
  - persistentVolumeClaim
  - projected
  - secret
  ```

### Resources Required

In addition to the [general hardware requirements and recommendations](https://www.ibm.com/support/knowledgecenter/en/SSQNUZ_2.1.0/com.ibm.icpdata.doc/zen/install/reqs-ent.html), this chart has the following requirements:

- x86 is the only architecture supported.
- Minimum CPU - 11
- Minimum Memory - 80Gi

## Storage

NFS volumes, Local volumes and Portworx can be used as a storage type for this chart. Following numbers of persistent volumes are required by data stores. Note that Local volumes is supported only on IBM Cloud Private.

| Component  | Number of replicas | Space per PVC | Number of PVs |
| ---------- | :----------------: | ------------- | :-----------: |
| PostgreSQL |         2          | 10 Gi         |       2       |
| Minio      |         4          | 50 Gi         |       4       |
| MongoDB    |         2          | 20 Gi         |       2       |
| etcd       |         2          | 10 Gi         |       2       |
| ARE        |         2          | 20 Gi         |       1       |

## Storage Class

[NFS](https://kubernetes.io/docs/concepts/storage/volumes/#nfs) and [Portworx](https://portworx.com/) can be used in ICP and OpenShift cluster. [Local volumes](https://kubernetes.io/docs/concepts/storage/volumes/#local) can be used in ICP cluster.

| Storage type  | ICP       | OpenShift     |
| ------------- | :-------: | :-----------: |
| NFS volumes   | Supported | Supported     |
| Local Volumes | Supported | Not Supported |
| Portworx       | Supported | Supported     |

### NFS volumes
A NFS volume allows an existing NFS (Network File System) share to be mounted into Kubernetes Pod.

https://kubernetes.io/docs/concepts/storage/volumes/#nfs

During installation process, specify NFS server address mount path.

Note: Dynamic Volume Provisioner is not automatically installed in CP4D 2.5 cluster. [NFS Client Provisioner](https://github.com/helm/charts/tree/master/stable/nfs-client-provisioner) needs to be manually installed.

### Local volumes

A local volume represents a mounted local storage device such as a disk, partition or directory.

https://kubernetes.io/docs/concepts/storage/volumes/#local

Local volumes can only be used as a statically created PersistentVolume. Dynamic provisioning is not supported yet. To create a PersistentVolume with `local-storage` storage class, you can define each volume configuration in a YAML file, and then use the `kubectl` command line to push the configuration changes to the cluster.

Note: Local volumes are not supported in OpenShift cluster. ARE does not work with Local volumes.

1. Create a `local-storage` storage class on your cluster. See [Kubernetes documentation](https://kubernetes.io/docs/concepts/storage/storage-classes/#local) for more detail.

    ```yaml
    apiVersion: storage.k8s.io/v1
    kind: StorageClass
    metadata:
      name: local-storage
    provisioner: kubernetes.io/no-provisioner
    volumeBindingMode: WaitForFirstConsumer
    ```

1. Prepare directories to be used by PVs on worker nodes.

1. Create 11 YAML files, one for each volume. The Number of required volumes can be different if you change the number of replicas of data stores. Default is 11.

    ```yaml
    apiVersion: v1
    kind: PersistentVolume
    metadata:
      name: {name}
    spec:
      capacity:
        storage: {size}Gi
      accessModes:
      - ReadWriteOnce
      persistentVolumeReclaimPolicy: Retain
      storageClassName: local-storage
      local:
        path: {path}
      nodeAffinity:
        required:
          nodeSelectorTerms:
          - matchExpressions:
            - key: kubernetes.io/hostname
              operator: In
              values:
              - {ip}
    ```

    - `{name}`: The name of the PV to be created.
    - `{size}`: Reflect the size specified in the storage requirement table above.
    - `{path}`: Path on the worker node where you create in the previous step. For example, `/mnt/local-storage/storage/{dir_name}`. For `{dir_name}`, use the same value that you use for `{name}` so you can map the volume name to its physical location.
    - `{ip}`: IP address of the worker node where you create the persistent volume.

1. Run the apply command on each YAML file that you create.

    For example: `kubectl apply -f pv_001.yaml`. Rerun this command for each file up to `kubectl apply -f pv_011.yaml`.

See the [installation document](https://cloud.ibm.com/docs/services/watson-knowledge-studio-data?topic=watson-knowledge-studio-data-install) for more detail.

### Portworx

Portworx is a persistent storage for stateful containers including HA, snapshots, backup & encryption.

https://portworx.com/

A `StorageClass` for Portworx needs to be created. A dynamic provisioner is automatically enabled. See [online document](https://docs.portworx.com/portworx-install-with-kubernetes/storage-operations/kubernetes-storage-101/volumes/#storageclass) for more details.

Note: `ReadWriteMany` mode is required for ARE to share a mounted volume between multiple replicas. See [online document](https://docs.portworx.com/portworx-install-with-kubernetes/storage-operations/kubernetes-storage-101/volumes/#readwritemany-and-readwriteonce) to enable `ReadWriteMany` mode.

# Installing IBM Watson Knowledge Studio

This CASE provides a helm chart of IBM Watson Knowledge Studio.

## Installing the Chart into CP4D (v2.1.0.2 or earlier) on IBM Cloud Private

### Setup environment

1. SSH login to a master node

1. Login to your cluster.

    ```bash
    cloudctl login -a https://{cluster_CA_domain}:8443 -u {user} -p {password}
    ```

    See the [installation documentation](https://cloud.ibm.com/docs/services/watson-knowledge-studio-data?topic=watson-knowledge-studio-data-install) for more detail.

1. Create a Kubernetes namespace.

   - Create a Kubernetes namespace with [`ibm-restricted-psp`](https://ibm.biz/cpkspec-psp) PSP. See the [installation documentation](https://cloud.ibm.com/docs/services/watson-knowledge-studio-data?topic=watson-knowledge-studio-data-install) for more detail.

1. Login to the docker registry like the following.

    ```bash
    docker login {cluster_CA_domain}:8500 -u {user} -p {password}
    ```

1. Change target namespace

    ```bash
    cloudctl target -n {namespace_name}
    ```

    - `{namespace_name}` is the namespace you created in Step 3.

1. Create a YAML file like the following, then run the apply command on the YAML file that you create. For example `kubectl apply -f image_policy.yaml`. For further details of ImagePolicy, see [Enforcing container image security](https://www.ibm.com/support/knowledgecenter/en/SSBS6K_3.1.2/manage_images/image_security.html)

    ```yaml
    apiVersion: securityenforcement.admission.cloud.ibm.com/v1beta1
    kind: ImagePolicy
    metadata:
      name: icp-image-policy
      namespace: {namespace_name}
    spec:
      repositories:
      - name: '{cluster_CA_domain}:8500/{namespace_name}/*'
    ```

    - `{namespace_name}`: your target namespace
    - `{cluster_CA_domain}`: your cluster CA domain name

1. To meet network security policy for CP4D addon, update label of the namespaces by the following 2 commands.

    ```bash
    kubectl label --overwrite namespace zen ns=zen
    kubectl label --overwrite namespace {namespace_name} ns={namespace_name}
    ```

### Deployment

To install the addon, run the following command:

```bash
cd /ibm/InstallPackage/components/
./deploy.sh -d {compressed_file_name} -e {release_name_postfix}
```

- `{compressed_file_name}` is the name of the file that you downloaded from Passport Advantage.
- `{release_name_postfix}` is the postfix of Helm release name of this installation.

The command will interactively ask you the following information, so answer like the following.

| Question                                                      | Answer                                                                                                                                |
| ------------------------------------------------------------  | ------------------------------------------------------------------------------------------------------------------------------------- |
| Do you agree with the terms and conditions in {license_URL}?  | If you accept the license agreement, answer `a`. The chart can be installed without this acceptance.                                  |
| Which namespace do you want to install in?                    | Namespace name same as previous `{namespace_name}`                                                                                    |
| Where the Docker repository                                   | `{cluster_CA_domain}:8500/{namespace_name}`.                                                                                          |
| Which StorageClass do you want to use                         | `NFS` or `local-storage`                                                                                                              |
| [In using NFS only] NFS server IP or hostname, and mount path | Your NFS server IP or hostname, and mount path.                                                                                       |

## Installing the Chart into CP4D (v2.1.0.2 or earlier) on OpenShift

### Setup environment

1. SSH login to a master node

1. Login to your cluster.

    ```bash
    oc login -u {user} -p {password}
    ```
    See the [Basic Setup and Login](https://docs.openshift.com/container-platform/3.11/cli_reference/get_started_cli.html#basic-setup-and-login) for more detail

1. Create a new OpenShift project.

    Create a new project with [`restricted`](https://docs.openshift.com/enterprise/3.0/architecture/additional_concepts/authorization.html#security-context-constraints) SCC. For example, you can create the new project by the following command. No operation to change SCC is required because default SCC is `restricted`.
    ```bash
    oc new-project {namespace_name} --description="{description_text}" --display-name="{display_name}"
    ```
    - `{namespace_name}` is name of creating project. This command also creates namespace named same as the project.
    - `{description_text}` is text to set as description of this project.
    - `{display_name}` is string to set as display name of this project.

    See the [Projects](https://docs.openshift.com/container-platform/3.11/dev_guide/projects.html) and [Add an SCC to a User, Group, or Project](https://docs.openshift.com/container-platform/3.11/admin_guide/manage_scc.html#add-scc-to-user-group-project) for more detail.

1. Login to the docker registry like the following.

    ```bash
    docker login docker-registry.default.svc:5000 -u $(oc whoami) -p $(oc whoami -t)
    ```

1. Change target namespace

    ```bash
    oc project {namespace_name}
    ```

    - `{namespace_name}` is the namespace you created in Step 3.

1. To meet network security policy for CP4D addon, update label of the namespaces by the following 2 commands.

    ```bash
    kubectl label --overwrite namespace zen ns=zen
    kubectl label --overwrite namespace {namespace_name} ns={namespace_name}
    ```

### Deployment

To install the addon, run the following command:

```bash
oc get secret | grep -Eo 'default-dockercfg[^ ]*' | xargs -n 1 oc get secret -o yaml | sed 's/default-dockercfg[^ ]*/sa-{namespace_name}/g' | oc create -f -
cd /ibm/InstallPackage/components/
./deploy.sh -o -d {compressed_file_name} -e {release_name_postfix}
```

- `{compressed_file_name}` is the name of the file that you downloaded from Passport Advantage.
- `{release_name_postfix}` is the postfix of Helm release name of this installation.

The first command line creates a copy of `default-dockercfg-****` secret named `sa-{namespace_name}` used in the installation.

`deploy.sh` command will interactively ask you the following information, so answer like the following.

| Question                                                     | Answer                                                                                                                                |
| ------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------- |
| Do you agree with the terms and conditions in {license_URL}? | If you accept the license agreement, answer `a`. The chart can be installed without this acceptance.                                  |
| Which namespace do you want to install in?                   | Namespace name same as previous `{namespace_name}`                                                                                    |
| Where the Docker repository                                  | `docker-registry.default.svc:5000/{namespace_name}`.                                                                                  |
| Which StorageClass do you want to use                        | `NFS`                                                                                                                                 |
| NFS server IP or hostname, and mount path                    | Your NFS server IP or hostname, and mount path.                                                                                       |


If you meet `ErrImagePull` error in the addon installation, try the followoing command to update the secret `sa-{namespace_name}`.

```bash
oc delete secret sa-{namespace_name}
oc get secret | grep -Eo 'default-dockercfg[^ ]*' | xargs -n 1 oc get secret -o yaml | sed 's/default-dockercfg[^ ]*/sa-{namespace_name}/g' | oc create -f -
```

## Installing the Chart into CP4D v2.5 on OpenShift
### Preparation

First, log in to master node of your OpenShift cluster or a Linux client machine that can access to the master node.

To install WKS, you need cpd utilities. To obtain them, see CP4D online document.
Before starting installation, please download the compressed addon image and utilities files into the machine that you log in to.

After then, please execute the following commands for preparation.

```bash
mkdir ~/{your_working_dir}
cd ~/{your_working_dir}

# extract PPA
mkdir ibm-watson-knowledge-studio-ppa
tar -xvf {path_to_compressed_addon_file} -C ibm-watson-knowledge-studio-ppa

# extract utilities
tar -xvf {path_to_compressed_utilitis_file}
```

- `{your_working_dir}` is a directory to use installation operation.
- `{path_to_compressed_addon_file}`: is the path to the compressed addon file you have downloaded.
- `{path_to_compressed_utilitis_file}`: is the path to the compressed utilities file you have downloaded.

### Log in to cluster

Log in to the OpenShift cluster and the internal docker registry by the following commands.

```bash
# Login to cluster
oc login https://{cluster_CA_domain}:8443 -u {admin_username} -p {admin_password}
oc project {cp4d_namespace}
docker login {docker_registry_external_url} -u {docker_username} -p $(oc whoami -t)
```

- `{cluster_CA_domain}` is your cluster CA domain name.
- `{admin_username}` is a username of the OpenShift administrator.
- `{admin_password}` is the password of the administrator user.
- `{cp4d_namespace}` is the namespace where CP4D is installed. In CP4D v2.5, you are able to install Watson Knowledge Studio into only the namespace where CP4D is installed.
- `{docker_registry_external_url}` is the external URL of the docker reigstry. You can obtain this URL by `oc get -n default route docker-registry --template '{{.spec.host}}'`
- `{docker_username}` is the username to login docker registry. This migth be different from `{admin_username}`

If you cannot login to docker registry due to TLS certification issue (i.e. You see error message "x509: certificate signed by unknown authority" on `docker login` command),
you can avoid this issue by the following commands that adds the CA certification of the docker registry as trusted CA certification.

```bash
pushd /etc/docker/certs.d/
mkdir ./{docker_registry_external_url}
openssl s_client -showcerts -servername {docker_registry_external_url} -connect {docker_registry_external_url}:443  </dev/null | sed -ne '/BEGIN CERTIFICATE/,/END CERTIFICATE/p;/END CERTIFICATE/q' > ./{docker_registry_external_url}/node-client-ca.crt  # put ca.crt of docker reigstry
popd
docker login {docker_registry_external_url} -u {docker_username} -p $(oc whoami -t)
```

### Create service accounts for Watson Knowledge Studio

You can create service account for Watson Knowledge Studio in your cluster by the following commands.

```bash
# Render rbac resource file
mkdir helm-templates
helm template ./ibm-watson-knowledge-studio-ppa/wks/charts/ibm-watson-ks-prod-1.1.1.tgz  -n {release_name} --output-dir helm-templates/
oc create -f helm-templates/ibm-watson-ks-prod/templates/role.yaml -f helm-templates/ibm-watson-ks-prod/templates/service-account.yaml -f helm-templates/ibm-watson-ks-prod/templates/role-binding.yaml
```

- `{release_name}` is helm release name of Watson Knowledge Studio

### Deploy Watson Knowledge Studio

First, prepare a yaml file named `valuesOverride.yaml` like the following.

```yaml
global:
  existingServiceAccount: {release_name}-ibm-watson-ks
  icpDockerRepo: "docker-registry.default.svc.cluster.local:5000/{cp4d_namespace}/"
  image:
    repository: "docker-registry.default.svc.cluster.local:5000/{cp4d_namespace}"

creds:
  image:
    repository: "docker-registry.default.svc.cluster.local:5000/{cp4d_namespace}"

mongodb:
  config:
    image:
      repository: "docker-registry.default.svc.cluster.local:5000/{cp4d_namespace}"
  mongodbInstall:
    image:
      repository: "docker-registry.default.svc.cluster.local:5000/{cp4d_namespace}"
  mongodb:
    image:
      repository: "docker-registry.default.svc.cluster.local:5000/{cp4d_namespace}"
  creds:
    image:
      repository: "docker-registry.default.svc.cluster.local:5000/{cp4d_namespace}"
  test:
    image:
      repository: "docker-registry.default.svc.cluster.local:5000/{cp4d_namespace}"
  persistentVolume:
    storageClass: "{storage_class}"
minio:
  persistence:
    storageClass: "{storage_class}"
postgresql:
  persistence:
    storageClassName: "{storage_class}"
etcd:
  dataPVC:
    storageClassName: "{storage_class}"

glimpse:
  creds:
    image:
      repository: "docker-registry.default.svc.cluster.local:5000/{cp4d_namespace}"
  builder:
    image:
      repository: "docker-registry.default.svc.cluster.local:5000/{cp4d_namespace}/wks-glimpse-ene-builder"
  query:
    modelmesh:
      image:
        repository: "docker-registry.default.svc.cluster.local:5000/{cp4d_namespace}/model-mesh"
    glimpse:
      image:
        repository: "docker-registry.default.svc.cluster.local:5000/{cp4d_namespace}/wks-ene-expand"
  helmTest:
    image:
      repository: "docker-registry.default.svc.cluster.local:5000/{cp4d_namespace}/opencontent-icp-cert-gen-1"

wcn:
  sch:
    image:
      repository: "docker-registry.default.svc.cluster.local:5000/{cp4d_namespace}"
  addonService:
    zenNamespace: "{cp4d_namespace}"

awt:
  persistentVolume:
    storageClassName: "{storage_class}"
```

- `{release_name}` is the helm release name of Watson Knowledge Studio.
- `{cp4d_namespace}` is the namespace where CP4D is installed.
- `{storage_class}` is the StorageClass name specified in the StorageClass definition.

After then, you can deploy Watson Knowledge Studio by the following commands.

```bash
cd bin/
./deploy.sh --docker_registry_prefix=docker-registry.default.svc.cluster.local:5000/{cp4d_namespace} --external_registry_prefix={docker_registry_external_url}/{cp4d_namespace} --target_namespace={cp4d_namespace} --storage_class={storage_class} -O ~/{your_working_dir}/valuesOverride.yaml -d ~/{your_working_dir}/ibm-watson-knowledge-studio-ppa/wks -e {release_name}
```

- `{cp4d_namespace}` is the namespace where CP4D is installed.
- `{docker_registry_external_url}` is the external URL of the docker reigstry. You can obtain this URL by `oc get -n default route docker-registry --template '{{.spec.host}}'`
- `{storage_class}` is the StorageClass name specified in the StorageClass definition.
- `{your_working_dir}` is the directory to use installation operation.
- `{release_name}` is the helm release name of Watson Knowledge Studio.

If the deploy.sh returns timeout error, you can set timeout by editing `deploy.sh` like the following.

```diff
   #start installing chart
-  cmd="${CUR_DIR}/utils/helm install -f ${CUR_DIR}/utils/deploy-utils/install.yaml -f ${CUR_DIR}/utils/deploy-utils/global.yaml ${OVERRIDES_ARGS} ${EXTERNAL_OVERRIDES_ARGS} --name ${release_name} --namespace ${namespace} ${target_dir}/charts/*.tgz --tls --tls-ca-cert ${ca_pem_file_path} --tls-cert ${cert_pem_file_path} --tls-key ${key_pem_file_path}"
+  cmd="${CUR_DIR}/utils/helm install --timeout 1800 -f ${CUR_DIR}/utils/deploy-utils/install.yaml -f ${CUR_DIR}/utils/deploy-utils/global.yaml ${OVERRIDES_ARGS} ${EXTERNAL_OVERRIDES_ARGS} --name ${release_name} --namespace ${namespace} ${target_dir}/charts/*.tgz --tls --tls-ca-cert ${ca_pem_file_path} --tls-cert ${cert_pem_file_path} --tls-key ${key_pem_file_path}"
   echo "running ${cmd}"
   eval "${cmd}"
```

After then, you can retry the following commands.

```bash
# Delete helm release
./utils/helm delete {release_name} --purge --tiller-namespace={cp4d_namespace} --tls --tls-ca-cert ${HOME}/.helm/ca.pem --tls-cert ${HOME}/.helm/cert.pem --tls-key ${HOME}/.helm/key.pem
oc delete deployment,statefulset,jobs,serviceaccount,secret,pvc,role,rolebinding -l release={release_name}

# Recreate service account
cd ..
oc create -f helm-templates/ibm-watson-ks-prod/templates/role.yaml -f helm-templates/ibm-watson-ks-prod/templates/service-account.yaml -f helm-templates/ibm-watson-ks-prod/templates/role-binding.yaml

# Retry to deploy
cd bin/
./deploy.sh --docker_registry_prefix=docker-registry.default.svc.cluster.local:5000/{cp4d_namespace} --external_registry_prefix={docker_registry_external_url}/{cp4d_namespace} --target_namespace={cp4d_namespace} --storage_class=portworx-wks -O ~/{your_working_dir}/{valuesOverride_yaml_file} -d ~/{your_working_dir}/ibm-watson-knowledge-studio-ppa/wks -e {release_name}
```

## Verification of deployment

After installation completed successfully, you can verify the deployment by running `helm test`.

```bash
helm test {release_name} --cleanup --timeout 600 --tls
```

### Accessing WKS tooling UI

After the successful installation, a WKS add-on tile with the release name is shown up on your CP4D console. You can provision a WKS instance and launch your WKS tooling application there.

1. Open `https://{cluster_CA_domain}:31843` (CP4D v2.1.0.2 or earlier) or `https://{namespace}-cpd-{namespace}.apps.{cluster_CA_domain}` (CP4D v2.5) by your web browser and login to CP4D console.

1. Move to Add-on catalog. You can find the add-on of IBM Watson Knowledge Studio with the release name.

1. Click the Watson Knowledge Studio add-on tile and provision an instance.

1. Open the created instance and click `Launch Tool` button.

1. You can start using IBM Watson Knowledge Studio.

## Uninstalling the chart

1. Delete the existing WKS instances of the release on CP4D console.

1. Run the following command to uninstall and delete the existing release.

    ```bash
    helm delete --tls {release_name}
    ```

    - `{release_name}` is the Helm release name of this installation.

    To irrevocably uninstall and delete the release, run the following command:

    ```bash
    helm delete --tls --no-hooks --purge {release_name}
    ```

If you omit the `--purge` option, Helm deletes all resources for the deployment but retains the record with the release name. This approach allows you to roll back the deletion. If you include the `--purge` option, Helm removes all records for the deployment so that the name can be used for another installation.

Before you can install a new version of the service on the same cluster, you must remove all content from any persistent volumes and persistent volume claims that were used for the previous deployment.

## Configuration

The following table lists the configurable parameters of the WKS chart and their default values.
These value can be changed on installation by specifying `-O` option of `deploy.sh`. For example, `./deploy.sh -O your_override_values.yaml`.

### Global parameters

|    Parameter    |                                              Description                                              |     Default     |
| --------------- | ----------------------------------------------------------------------------------------------------- | --------------- |
| `global.clusterDomain` | Cluster domain used by Kubernetes Cluster (the suffix for internal KubeDNS names).                    | `cluster.local` |

### Affinity parameters

Following table lists the affinity parameters for the components in the WKS chart. See [Affinity and anti-affinity](https://kubernetes.io/docs/concepts/configuration/assign-pod-node/#affinity-and-anti-affinity) for the details of arguments.

|     Parameter     |                                    Description                                     |     Default     |
| ----------------- | ---------------------------------------------------------------------------------- | --------------- |
| `affinity` | Node/pod affinities for WKS Frontend/Broker/Dispatcher pods. If specified overrides default affinity to run on any amd64 node. | `{}` |
| `mongodb.affinityMongodb` | Node/pod affinities for Mongodb statefulset only. If specified overrides default affinity to run on any amd64 node. | `{}` |
| `minio.affinityMinio` | Node/pod affinities for Minio statefulset only. If specified overrides default affinity to run on any amd64 node. | `{}` |
| `postgresql.sentinel.affinity` | Affinity settings for sentinel pod assignment. | `{}` |
| `postgresql.proxy.affinity` | Affinity settings for proxy pod assignment. | `{}` |
| `postgresql.keeper.affinity` | Affinity settings for keeper pod assignment. | `{}` |
| `etcd.affinityEtcd` | Affinities for Etcd stateful set. Overrides the generated affinities if provided. | `{}` |
| `glimpse.affinity`  | Node/pod affinities for Glimpse pods. If specified overrides default affinity to run on any amd64 node. | `{}`    |
| `sire.affinity`  | Node/pod affinities for SIRE pods. If specified overrides default affinity to run on any amd64 node. | `{}`    |
| `wcn.affinity`  | Node/pod affinities for Watson Gateway pods. If specified overrides default affinity to run on any amd64 node. | `{}`    |

### WKS Front-end parameters

|       Parameter        |                                                           Description                                                            | Default |
| ---------------------- | -------------------------------------------------------------------------------------------------------------------------------- | ------- |
| `replicaCount`         | Number of replicas for WKS Front-end deployment.                                                                                  | `2`     |

### Service Broker parameters

|       Parameter        |                                                           Description                                                            | Default |
| ---------------------- | -------------------------------------------------------------------------------------------------------------------------------- | ------- |
| `replicas`         | Number of replicas for Service Broker deployment.                                                                                  | `2`     |

### Dispatcher parameters

|       Parameter        |                                                           Description                                                            | Default |
| ---------------------- | -------------------------------------------------------------------------------------------------------------------------------- | ------- |
| `replicas`         | Number of replicas for Dispatcher deployment.                                                                                  | `2`     |

### SIRE parameters

|                          Parameter                          |                                                                                          Description                                                                                           | Default |
| ----------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------- |
| `global.highAvailabilityMode`                               | It this values is `true`, multiple replicas are created for SIRE training service (SIREG, SIRE Job queue, SIRE Train Facade). If set `false`, a single replica is created for each deployment. | `true`  |
| `sire.jobq.tenants.train.max_queued_and_active_per_user`    | Maximum amount of queued and active training jobs, any additional jobs will be rejected with an error.                                                                                        | `10`    |
| `sire.jobq.tenants.train.max_active_per_user`               | Maximum amount of training jobs running in parallel even if enough cpu/mem quota available.                                                                                                    | `2`     |
| `sire.jobq.tenants.evaluate.max_queued_and_active_per_user` | Maximum amount of queued and active evaluate jobs, any additional jobs will be rejected with an error.                                                                                        | `10`    |
| `sire.jobq.tenants.evaluate.max_active_per_user`            | Maximum amount of evaluate jobs running in parallel even if enough cpu/mem quota available.                                                                                                    | `2`     |
| `sire.sireg.languages.en.enabled`                                | Toggle language support for English                                                                                                                                                            | `true`  |
| `sire.sireg.languages.es.enabled`                                | Toggle language support for Spanish                                                                                                                                                            | `true`  |
| `sire.sireg.languages.ar.enabled`                                | Toggle language support for Arabic                                                                                                                                                             | `true`  |
| `sire.sireg.languages.de.enabled`                                | Toggle language support for German                                                                                                                                                             | `true`  |
| `sire.sireg.languages.fr.enabled`                                | Toggle language support for French                                                                                                                                                             | `true`  |
| `sire.sireg.languages.it.enabled`                                | Toggle language support for Italian                                                                                                                                                            | `true`  |
| `sire.sireg.languages.ja.enabled`                                | Toggle language support for Japanese                                                                                                                                                           | `true`  |
| `sire.sireg.languages.ko.enabled`                                | Toggle language support for Korean                                                                                                                                                             | `true`  |
| `sire.sireg.languages.nl.enabled`                                | Toggle language support for Dutch                                                                                                                                                              | `true`  |
| `sire.sireg.languages.pt.enabled`                                | Toggle language support for Portuguese                                                                                                                                                         | `true`  |
| `sire.sireg.languages.zh.enabled`                                | Toggle language support for Chinese (simplified)                                                                                                                                               | `true`  |
| `sire.sireg.languages.zht.enabled`                               | Toggle language support for Chinese (traditional)                                                                                                                                              | `true`  |

### MMA parameters

|   Parameter    |        Description         | Default |
| -------------- | -------------------------- | ------- |
| `mma.replicas` | Number of replicas for MMA | `2`     |

### MongoDB parameters

|                 Parameter                 |                                            Description                                             |      Default      |
| ----------------------------------------- | -------------------------------------------------------------------------------------------------- | ----------------- |
| `mongodb.replicas`                        | Number of replicas for MongoDB StatefulSet.                                                       | `2`               |
| `mongodb.persistentVolume.enabled`        | If `true`, persistent volume claims are created for MongoDB.                                       | `true`            |
| `mongodb.persistenceVolume.useDynamicProvisioning` | If enabled, the PVC will use a storageClassName to bind the volume for MongoDB.                                                                                                                                                                         | `true`         |
| `mongodb.persistentVolume.storageClass`   | Persistent volume storage class for MongoDB.                                                       | `local-storage`              |
| `mongodb.persistentVolume.accessMode`     | Persistent volume access modes for MongoDB.                                                        | `[ReadWriteOnce]` |
| `mongodb.persistentVolume.size`           | Persistent volume size for MongoDB.                                                                | `20Gi`            |
| `mongodb.persistentVolume.selector.label` | Label for persistent volume claim selectors to control how pvc's bind/reserve storage for MongoDB. | `""`              |
| `mongodb.persistentVolume.selector.value` | Value for persistent volume claim selectors to control how pvc's bind/reserve storage for MongoDB. | `""`              |
| `mongodb.persistentVolume.annotations`    | Persistent volume annotations for MongoDB.                                                         | `{}`              |

### Minio (Object Storage) parameters

|                 Parameter                  |                                                                                                                      Description                                                                                                                      |     Default     |
| ------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------- |
| `minio.replicas`                           | Number of nodes. Must be 4 <= x <= 32                                                                                                                                                                     | `4`             |
| `minio.persistence.enabled`                | Use PV to store data on Minio.                                                                                                                                                                                                                        | `true`          |
| `minio.persistence.size`                   | Size of persistent volume claim (PVC) for Minio.                                                                                                                                                                                                      | `10Gi`          |
| `minio.persistence.storageClass`           | Storage Class to bind PVC for Minio.                                                                                                                                 | `local-storage`          |
| `minio.persistence.accessMode`             | Persistent volume storage class for Minio.                                                                                                                                                                               | `ReadWriteOnce` |
| `minio.persistence.subPath`                | Mount a sub directory of the persistent volume for Minio, if a sub directory is set.                                                                                                                                                                  | `""`            |

### PostgreSQL parameters

|              Parameter               |                                                                                                                                   Description                                                                                                                                    |     Default     |
| ------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------- |
| `postgresql.sentinel.replicas`                  | Number of sentinel nodes                                                                                                                                                                                                                                                         | `2`             |
| `postgresql.proxy.replicas`                     | Number of proxy nodes                                                                                                                                                                                                                                                            | `2`             |
| `postgresql.keeper.replicas`                    | Number of keeper nodes                                                                                                                                                                                                                                                           | `2`             |
| `postgresql.persistence.enabled`                | Use a PVC to persist data                                                                                                                                                                                                                                                        | `true`          |
| `postgresql.persistence.useDynamicProvisioning` | Enables dynamic binding of Persistent Volume Claims to Persistent Volumes                                                                                                                                                                                                        | `true`          |
| `postgresql.persistence.storageClassName`       | Storage class name of backing PVC                                                                                                                                                                                                                                                | `local-storage` |
| `postgresql.persistence.accessMode`             | Persistent volume storage class for PostgreSQL.                                                                                                                                                                                                                                              | `ReadWriteOnce` |
| `postgresql.persistence.size`                   | Size of data volume                                                                                                                                                                                                                                                              | `10Gi`          |
| `postgresql.dataPVC.name`                       | Prefix that gets the created Persistent Volume Claims                                                                                                                                                                                                                            | `stolon-data`   |
| `postgresql.dataPVC.selector.label`             | In case the persistence is enabled and useDynamicProvisioning is disabled the labels can be used to automatically bound persistent volumes claims to precreated persistent volumes. The persistent volumes to be used must have the specified label. Disabled if label is empty. | `""`            |
| `postgresql.dataPVC.selector.value`             | In case the persistence is enabled and useDynamicProvisioning is disabled the labels can be used to automatically bound persistent volumes claims to precreated persistent volumes. The persistent volumes to be used must have label with the specified value.                  | `""`            |

### etcd parameters

| Parameter                            | Description                                                                                                                                                                                                                                                                      | Default         |
| ------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------- |
| `etcd.replicaCount`                  | Number of etcd nodes                                                                                                                                                                                                                                                             | `2`             |
| `etcd.maxEtcdThreads`                | Maximum Number of Threads Etcd Can Use                                                                                                                                                                                                                                           | `2`             |
| `etcd.persistence.enabled`           | Enables use of Persistent Volumes                                                                                                                                                                                                                                                | `true`          |
| `etcd.dataPVC.accessMode`            | Access Mode for the Persistent Volume                                                                                                                                                                                                                                            | `ReadWriteOnce` |
| `etcd.dataPVC.name`                  | Prefix that gets the created Persistent Volume Claims                                                                                                                                                                                                                            | `data`          |
| `etcd.dataPVC.storageClassName`      | In case the persistence is enabled. The StorageClass for created persistent volumes claims that holds etcd data. If empty value is used, the default StorageClass will be used.                                                                                                  | ` `             |
| `etcd.dataPVC.selector.label`        | In case the persistence is enabled and useDynamicProvisioning is disabled the labels can be used to automatically bound persistent volumes claims to precreated persistent volumes. The persistent volumes to be used must have the specified label. Disabled if label is empty. | `""`            |
| `etcd.dataPVC.selector.value`        | n case the persistence is enabled and useDynamicProvisioning is disabled the labels can be used to automatically bound persistent volumes claims to precreated persistent volumes. The persistent volumes to be used must have label with the specified value.                   | `""`            |
| `etcd.dataPVC.size`                  | Size of the Persistent Volume Claim                                                                                                                                                                                                                                              | `10Gi`          |

### Glimpse parameters

|              Parameter               |                                                                                                                                   Description                                                                                                                                    |     Default     |
| ------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------- |
| `glimpse.create`                  | If true, Glimpse services and etcd is deployed and dictionary suggestions feature gets available on WKS UI. If false, they are not deployed, and dictionary suggestions feature doens't become available on WKS UI. | `true`             |
| `glimpse.builder.replicas`                          | Number of replicas for Glimpse builder.                  | `2`                             |
| `glimpse.builder.resources.requests.cpu`            | CPU resource request for Glimpse builder. To reduce processing time to build Glimpse model, increasing this can be a good idea. | `100m`                         |
| `glimpse.builder.resources.requests.memory`         | Memory resource request for Glimpse builder. It is recommended to increase this value to get suggestions from large corpus. Typically `8000M` is enough. | `1000M`                         |
| `glimpse.builder.resources.limits.cpu`             | CPU resource limit for Glimpse builder. To reduce processing time to build Glimpse model, increasing this can be a good idea. | `4000m`                         |
| `glimpse.builder.resources.limits.memory`           | Memory resource limit for Glimpse builder.             | `8000M`                         |
| `glimpse.query.replicas`                                          | Number of replicas for Glimpse query server.                                                    | `2`                             |
| `glimpse.query.glimpse.resources.requests.cpu`                    | CPU resource request for Glimpse query server.                                             | `150m`                         |
| `glimpse.query.glimpse.resources.requests.memory`                 | Memory resource request for Glimpse query server.                                          | `1000M`                         |
| `glimpse.query.glimpse.resources.limits.cpu`                      | CPU resource limit for Glimpse query server.                                               | `4000m`                         |
| `glimpse.query.glimpse.resources.limits.memory`                   | Memory resource request for Glimpse query server.                                          | `8000M`                         |

### Advanced Rule Editor parameters

| Parameter                                     | Description                                                                                                                                                                                            | Default   |
| --------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | --------- |
| `awt.create`                                  | If true, Advaned Rule Editor service is deployed, and you can select project type for Advanced Rules on WKS UI. If false, they are not deployed, and you cannot select Advanced Rules projects.        | `true`    |
| `awt.replicas`                                | Number of replicas for Advanced Rule Editor                                                                                                                                                            | `2`       |
| `awt.persistentVolume.storageClassName`       | Storage class name of backing PVC                                                                                                                                                                      | `""`      |
| `awt.persistentVolume.useDynamicProvisioning` | Enables dynamic binding of Persistent Volume Claims to Persistent Volumes                                                                                                                              | `false`   |
| `awt.persistentVolume.size`                   | Size of data volume                                                                                                                                                                                    | `20Gi`    |

## Storage

Please refer to the section "Resources Required".

## Limitations

- Only the `x86` architecture is supported.
- The chart must be installed by a cluster administrator.

## Documentation

Find out more about IBM Watson Knowledge Studio by reading the [product documentation](https://cloud.ibm.com/docs/services/watson-knowledge-studio-data?topic=watson-knowledge-studio-data-wks_overview_full).

**Note**: The documentation link takes you out of IBM Cloud Private to the public IBM Cloud.