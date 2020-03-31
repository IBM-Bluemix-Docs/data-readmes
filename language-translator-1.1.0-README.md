# ibm-watson-lt-prod

[IBM Watson™ Language Translator](https://www.ibm.com/watson/services/language-translator/index.html#about) allows you to translate text programmatically from one language into another language.

## Introduction

Expand to new markets by instantly translating your documents, apps, and webpages. Create multilingual chatbots to communicate with your customers on their terms. What will you build with the power of language at your fingertips?

## Chart Details

This chart installs the Watson Language Translator as a **service for IBM Cloud Pak for Data** (https://www.ibm.com/products/cloud-pak-for-data).

After a successful installation, go to your Cloud Pak for Data UI and you will find the Language Translator service listed in the catalog.

This chart creates several pods, statefulsets, services, and secrets to create the Language Translator offering.

Deployments:

* `{release-name}-ibm-watson-lt-api` (api server)
* `{release-name}-ibm-watson-lt-documents` (document translation)
* `{release-name}-ibm-watson-lt-lid` (language identification backend)
* `{release-name}-ibm-watson-lt-segmenter` (sentence splitting backend)
* For each installed language pair `<source>-<target>` (e.g. `en-de`):
  * : `{release-name}-ibm-watson-lt-engine-<source>-<target>` (translation backend for a language pair)


Statefulsets:

* `{release-name}-ibm-minio` (S3-API compatible storage)
* `{release-name}-ibm-postgresql-keeper` (highly available PostgreSQL)

ConfigMaps:

* `{release-name}-ibm-minio`
* `{release-name}-ibm-watson-lt-api-config`
* `{release-name}-ibm-watson-lt-documents-config`
* `{release-name}-ibm-watson-lt-model-config`
* `{release-name}-ibm-watson-lt-language-translator-gateway`
* `stolon-cluster-{release-name}-postgres`


Secrets:

* `{release-name}-ibm-postgresql-auth-secret` (PostgreSQL authentication)
* `{release-name}-ibm-postgresql-tls-secret` (PostgreSQL TLS certs)
* `{release-name}-ibm-minio-auth` (MinIO authentication)
* `{release-name}-ibm-minio-tls` (MinIO TLS certs)

## Prerequisites

* IBM® Cloud Pak for Data 2.1 or 2.5

  Before installing Watson Language Translator, __you must install and configure [IBM Cloud Pak for Data](https://www.ibm.com/support/knowledgecenter/SSQNUZ_current/com.ibm.icpdata.doc/zen/overview/relnotes-2.1.0.0.html)__.
* For a production mode installation, persistent volumes are set up, prior to installation; see [Storage](#storage-class-and-persistent-volume-set-up) section.

## Language Support via separate *Language Paks*

Translation models are provided in three separate installation packages. You need to download and install at least one language pak as a prerequisite to installing the Watson Language Translator service.

A language pak is a simple `tar` archive that contains a collection of Docker images, one for each language pair that you might want to install. Each Docker image for a language pair (e.g. English to German translation) has a size between 1GB and 2.5GB on disk.

**Note**: Please see steps 5, 6 and 8 in the [Setting up the Environment](#setting-up-environment) section for more installation details.

Languages are grouped into the following packages:

### IBM Watson Language Translator Language Pak 1 (CC47TML)

For each of the following languages, the package contains a translation model for *English* to the language and a reverse translation model for the language into *English*:

| Language       | Language Code |
|----------------|---------------|
| Arabic   | `ar`                |
| Chinese (simplified)| `zh`     |
| Chinese (traditional)| `zh-TW` |
| French| `fr`                   |
| German| `de`                   |
| Hebrew| `he`                   |
| Italian| `it`                  |
| Portuguese (Brazilian)| `pt`   |
| Russian| `ru`                  |
| Spanish| `es`                  |
| Turkish| `tr`                  |


### IBM Watson Language Translator Language Pak 2 (CC47UML)

For each of the following languages, the package contains a translation model for *English* to the language and a reverse translation model for the language into *English*:

| Language       | Language Code |
|----------------|---------------|
| Hindi          | `hi`          |
| Indonesian     | `id`          |
| Japanese       | `ja`          |
| Korean         | `ko`          |
| Malay          | `ms`          |
| Thai           | `th`          |

### IBM Watson Language Translator Language Pak 3 (CC47VML)

For each of the following languages, the package contains a translation model for *English* to the language and a reverse translation model for the language into *English*:

| Language       | Language Code |
|----------------|---------------|
| Bulgarian      | `bg`          |
| Croatian       | `hr`          |
| Czech          | `cs`          |
| Danish         | `da`          |
| Dutch          | `nl`          |
| Estonian       | `et`          |
| Finnish        | `fi`          |
| Greek          | `el`          |
| Hungarian      | `hu`          |
| Irish          | `ga`          |
| Lithuanian     | `lt`          |
| Norwegian Bokmål | `nb`        |
| Polish         | `pl`          |
| Romanian       | `ro`          |
| Slovak         | `sk`          |
| Slovenian      | `sl`          |
| Swedish        | `sv`          |

#### Extra Non-English Language Pairs in Language Pak 3:

| Language Pair  | Language Pair Codes   |
|----------------|-----------------------|
| Catalan <-> Spanish | `ca-es`, `es-ca` |
| German <-> French   | `de-fr`, `fr-de` |
| German <-> Italian  | `de-it`, `it-de` |
| French <-> Spanish  | `fr-es`, `es-fr` |

## PodSecurityPolicy Requirements

This chart requires a PodSecurityPolicy to be bound to the target namespace prior to installation. To meet this requirement there is a cluster scoped as well as namespace scoped pre-install script that must be run. The predefined PodSecurityPolicy name: [`ibm-restricted-psp`](https://ibm.biz/cpkspec-psp) has been verified for this chart. **If your target namespace is bound to this predefined policy, you can skip this section**.

These PodSecurityPolicy resources can also be created manually. A cluster admin can save these templates to separate YAML files and run the command below for each file:

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
  name: ibm-chart-prod-psp
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
  name:  lt-custom-priv-role
rules:
- apiGroups: ["", "batch", "extensions"]
  resources: ["jobs", "jobs/status", "secrets", "pods", "pods/exec", "configmaps"]
  verbs: ["get", "watch", "create", "apply", "list", "update", "patch", "delete"]
- apiGroups: ["policy"]
  resources: ["podsecuritypolicies"]
  resourceNames: [ibm-lt-custom-psp]
  verbs: ["use"]
- apiGroups: [""]
  resources: ["resourcequotas", "resourcequotas/status"]
  verbs: ["get", "list", "watch"]
```

To use a custom Role resource you must have custom ServiceAccount resource.

**NOTE:** You **must** replace `<namespace-name>` with the actual name of the namespace you're planning to deploy to because these Kubernetes resources are being created outside of a Helm release.

Template for custom ServiceAccount resource to replace the default privileged service account:
  ```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: lt-custom-priv-service-account
imagePullSecrets:
- name: sa-<namespace-name>
```

To bind your service accounts to your role, you need to create a rolebinding.

Template for role binding your custom privileged service account to your custom privileged role:
  ```yaml
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name:  lt-custom-priv-role-binding
subjects:
- kind: ServiceAccount
  name: lt-custom-priv-service-account
roleRef:
  kind: Role
  name:  lt-custom-priv-role
  apiGroup: rbac.authorization.k8s.io
```

Finally, you can specify the name of the custom service account when installing the chart. For example,if you want to override the privileged service account with `lt-custom-priv-service-account`, add the following to your helm install command:

```bash
<...> --set global.privilegedServiceAccount.name=lt-custom-priv-service-account --set global.privilegedServiceAccount.create=false --set global.privilegedRbac.create=false
```

We recommend, however, to just run the pre-install script described in step 6 of the [Setting up Environment](#setting-up-environment) to create the necessary security prerequisite resources for you.

## Red Hat OpenShift SecurityContextConstraints Requirements

If running in a Red Hat OpenShift cluster, this chart requires a `SecurityContextConstraints` to be bound to the target namespace prior to installation. To meet this requirement there is a cluster scoped as well as namespace scoped pre-install script that must be run. The predefined PodSecurityPolicy name: [`restricted`](https://ibm.biz/cpkspec-scc) has been verified for this chart. **If your target namespace is bound to this predefined policy, you can skip this section**.

The SecurityContextConstraint resource can also be created manually. A cluster admin can save this template to a yaml file and run the command below:

```bash
kubectl create -f ibm-lt-prod-scc.yaml
```

Template for a SecurityContextConstraints definition, currently equivalent `restricted`, to bind to your namespace.

* Custom SecurityContextConstraints definition:

```yaml
apiVersion: security.openshift.io/v1
kind: SecurityContextConstraints
metadata:
  annotations:
    kubernetes.io/description: "This policy is the most restrictive,
      requiring pods to run with a non-root UID, and preventing pods from accessing the host."
    cloudpak.ibm.com/version: "1.1.0"
  name: ibm-lt-prod-scc
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

```bash
oc adm policy add-scc-to-group ibm-lt-prod-scc system:serviceaccounts:$namespace
```

We recommend, however, to just run the pre-install script described in step 6 of the [Setting up Environment](#setting-up-environment) to create the necessary security prerequisite resources for you.

## Resources Required

In addition to the [general hardware requirements and recommendations](https://docs-icpdata.mybluemix.net/docs/content/SSQNUZ_current/com.ibm.icpdata.doc/zen/install/reqs-ent.html#reqs-ent__hardware), the IBM Language Translator service has the following requirements:

| Resource       | Dev  | Prod (HA) |
|----------------|----- |-----------|
| Minimum CPU    | 8    | 16        |
| Minimum Memory | 30GB | 80GB      |


The dev requirements are based on:

* single replicas for service components
* 2 installed translation models

The prod (HA) requirements are based on:

* 2 replicas (highly available mode) for service components
* 6 installed translation models

## Storage Requirements

| Datastore      | Space per PVC | Storage type | Supported Storage Classes |
|----------------|---------------|--------------|---------------------------|
| PostgreSQL     | 10 GB | Block Storage | portworx, vsphere                |
| Minio          | 10 GB | Block Storage | portworx, vsphere                |

### Storage Class and Persistent Volume Set Up

A Persistent Volume (PV) is a unit of storage in the cluster. In the same way that a node is a cluster resource, a persistent volume is also a resource in the cluster. For an overview, see Persistent Volumes in the [Cloud Pak for data storage add-ons documentation](https://www.ibm.com/support/knowledgecenter/SSQNUZ_2.1.0/com.ibm.icpdata.doc/zen/admin/install-storage-add-ons.html).

You can use a Cloud Pak for Data storage add-on, or a storage option that is hosted outside the cluster, such as the [vSphere Cloud Provider](https://www.ibm.com/support/knowledgecenter/SSBS6K_3.1.2/manage_cluster/vsphere_land.html).

Note the storage class volume type requirements below when selecting your storage options.

To see the available storage classes in your cluster, or to verify that you have properly set up persistent volumes and storage classes, run the applicable command and confirm the storage class you configured is listed:

On ICP with Kubernetes:

```bash
kubectl get storageclass
```

On OpenShift:

```bash
oc get storageclass
```

## Pre-install steps

This script has to be run once per cluster by a cluster admin. Run:

```bash
cd /path/to/unpacked/ppa-package
$ ./deploy/pre-install/clusterAdministration/labelNamespace.sh <CP4D_NAMESPACE>
 ```

where `<CP4D_NAMESPACE>` is the namespace where CP4D is installed (usually `zen`).

(The CP4D_NAMESPACE namespace must have a label for the NetworkPolicy to correctly work. Only nginx and zen pods will be able allowed to communicate with the pods in the namespace where this chart is installed.)

### Setting up Environment

1. **Initialize Helm**: Initialize the Helm client by running the following command. For further details of Helm CLI setup, see [Installing the Helm CLI](https://www.ibm.com/support/knowledgecenter/SSBS6K_3.1.2/app_center/create_helm_cli.html).

    ```bash
    export HELM_HOME={helm_home_dir}
    helm init --client-only
    ```

    * `{helm_home_dir}` is your Helm config directory. For example, `~/.helm`.

2. **Kubernetes/OpenShift Login**: Log into the Kubernetes cluster and target the namespace or project the chart will be deploy to. If you're deploying to an ICP cluster, use the `cloudctl` CLI.

    ```bash
    cloudctl login -a https://{cluster_hostname}:8443 -u {user} -p {password}
    ```

    If you're deploying to an OpenShift cluster, use the `oc` CLI.

    ```bash
    oc login https://{cluster_hostname}:8443 -u {user} -p {password}
    ```

3. **Docker registry**: Log into the cluster's docker registry.

    ICP cluster:

    ```bash
    docker login {cluster_hostname}:8500 -u {user} -p {password}
    ```

    OpenShift cluster:

    If you do not know the remote cluster's Docker registry hostname, run:

    ```bash
    oc get routes --all-namespaces | grep docker
    ```

    ```bash
    docker login -u ocadmin -p $(oc whoami -t) <remote registry>
    ```

4. **Main PPA**: Download the main PPA (`CC42HEN`) and create a directory to extract its content to, for example:

    ```bash
    mkdir ibm-watson-lt-install
    cd ibm-watson-lt-install
    tar -xvf /path/to/main-ppa-archive
    ```

    Unpacking the main PPA should have created the following directory tree:

    ```bash
    ibm-watson-lt-install
    |__ ibm-watson-lt
       |__ charts
       |__ images
       |__ deploy
       |__ dev-values-override.yaml
       |__ prod-values-override.yaml
    ```

5. **Language Pak(s) PPA**: You need to download and extract at least one of the separate language pak PPAs:

    ```bash
    cd ibm-watson-lt-install
    tar -xvf /path/to/language-pak-ppa-archive
    ```

    For example, unpacking the language pak1 PPA should have created the following directory tree:

    ```bash
    ibm-watson-lt-install
    |__ ibm-watson-lt-pak1-1.1.0
       |__ images
          |__ lt_ar-ar_en-us_general_1.1.0.tar.gz
          |__ lt_de-de_en-us_general_1.1.0.tar.gz
          |__ ...
          |__ lt_zh-tw_en-us_general_1.1.0.tar.gz
    ```

6. **Copy language pair images**: For each language pair that you want to install, copy its packed Docker image into the main PPA images directory. You do **not** need to unpack these tar.gz files.

    As an example, if you want to install the two language pairs Arabic to English (`ar-ar_en-us`) and German to English (`de-de_en-us`) from language pak1:

    ```bash
    cd ibm-watson-lt-install/
    cp ibm-watson-lt-pak1-1.1.0/images/lt_ar-ar_en-us_general_1.1.0.tar.gz ibm-watson-lt/images
    cp ibm-watson-lt-pak1-1.1.0/images/lt_de-de_en-us_general_1.1.0.tar.gz ibm-watson-lt/images
    ```

7. **Configure chart**: Edit the chart configuration YAML files to prepare the installation.

    The chart comes with two pre-defined default configurations, one for development mode and one for production mode.

    * `ibm-watson-lt/dev-values-override.yaml`: deploys the default development configuration.
    * `ibm-watson-lt/prod-values-override.yaml`: deploys a highly available (HA) configuration.

    Set the internal cluster Docker registry and the namespace where images are pushed to and pulled from. For OpenShift installations, the Docker registry is usually `docker-registry.default.svc:5000` and the target namespace is `zen`. For ICP cluster installations the registry is usually `{cluster_hostname}:8500`. In the following snippet, replace `<registry_host>` with the Docker registry host and `<target_namespace>` with the target namespace:

    ```bash
    global:
      icpDockerRepo: <registry_host>/<target_namespace>
      image:
        repository: <registry_host>/<target_namespace>
    ```

    Example for OpenShift:

    ```bash
    global:
      icpDockerRepo: docker-registry.default.svc:5000/zen
      image:
        repository: docker-registry.default.svc:5000/zen
    ```

    If the <target_namespace> is different from `zen`, set the following parameter:

    ```bash
    gateway:
      addonService:
        zenNamespace: <target_namespace>
    ```

    If installing a production configuration (`ibm-watson-lt/prod-values-override.yaml`) and using persistent volumes for the data stores, set the storage class if it is different from `portworx-sc` (`portworx-sc` is the default in the production configuration):

    For MinIO (s3):

    ```bash
    s3:
      persistence:
        enabled: true
        storageClass: "portworx-sc" # <-- set this to the desired storage class
        size: 10Gi
    ```

    For PostgreSQL:

    ```bash
    postgres:
      persistence:
        enabled: true
        storageClass: "portworx-sc" # <-- set this to the desired storage class
        size: 10Gi
    ```

8. **Enabling Language Support**:

    You need to **enable at least one language pair** in the installation configuration before proceeding with the Helm chart installation. In the predefined *development* and *production* configurations, all language pairs are disabled by default. To enable a language pair, edit the desired configuration file. In the configuration file, go to the very bottom section which lists all available language pairs. For the language pairs that you want to install, set the `enabled` parameter to `true`. For example, to enable German to English translation, set `translationModels.de-en.enabled: true`:

    ```bash
    translationModels:
      ...
      de-en:
        enabled: true  # <--- set this to true to enable the German to English translation model
        image:
          name: lt_de-de_en-us_general
          tag: 1.1.0
      ...
    ```

    **Important**: For every language pair that you enable in the configuration, you must have *previously copied its corresponding Docker image* .tar.gz file into the `ibm-watson-lt/images` folder, please see step 6 for details.

    Leave all other language pair configurations as is.

    Please note that the **CPU and memory requirements are based on installation of 2 language pairs for development and 6 language pairs for production** (also see section [Resources Required](#resources-required)).


## Installing the Chart

### Installation on CPD 2.1.0.2 - OpenShift 3.x and ICP

Locate the `deploy.sh` utility that comes with the Cloud Pak for Data 2.1.x installation. Usually, you can find it installed on the master node in directory `/ibm/InstallPackage/components/`.

For help, run

```bash
./deploy.sh -h
```

For the installation, the flags you most likely want to include:

```bash
./deploy.sh -d /absolute/path/to/ibm-watson-lt -O <override-file>.yaml -o -e <release-name-prefix>
```

**Important**:

* If installing on an OpenShift cluster, you must add the `-o` option.
* If installing on an ICP cluster, remove the `-o` option.

In your deploy command `-d` should point to the `ibm-watson-lt` directory that you have extracted in step 4.

Set the `-O` (capital letter O) option pointing to either the `dev-values-override.yaml` or the `prod-values-override.yaml` configuration.

By default, the script will look for the tiller pod in the `kube-system` namespace. If your tiller pod is located in a different namespace, you can override it with the `-w` flag. You can also find the tiller namespace by running:

```bash
oc get pods --all-namespaces | grep "til"
```


The final name of your helm release will be `<namespace>-<release-name-prefix>`. The total length of `<namespace>-<release-name-prefix>` must be [20 characters or less or the installation will fail](#limitations).

The rest of the flags can be filled in interactively by the terminal prompt:

`-c`. The console port of ICP4D. Most likely `31843`.

`-s`. The storage class used for the PVCs of the deployment (also see step 7)

`-n`. The namespace you're deploying to (usually `zen`).

`-r`. The internal Docker registry prefix of the docker image used by the cluster to pull the images.

`-R`. The Docker registry hostname used by the script to push Docker images to. It will be the same as the above when installing from a cluster node (recommended).

For ICP clusters, the values of `-r` and `-R` usually is `<master_hostname>:8500/<namespace>`.

For OpenShift clusters, the values of `-r` and `-R` is `docker-registry.default.svc:5000/<namespace>`.

### Installation on CPD 2.5 and 2.1.0.2 lite - OpenShift 3.x

Locate the `deploy.sh` utility that comes with your Cloud Pak for Data 2.5 installation. For lite installations, we also package the utility in the Watson Language Translator PPA. You can find the `deploy.sh` in folder `ibm-watson-lt/deploy/utils-cpd-lite/bin/`. This version of the `deploy.sh` script works for 2.5 and 2.1.0.2 lite installations.

1. Set an environment variable to specify the namespace of the Tiller pod:

    Find the tiller namespace:

    ```bash
    oc get pods --all-namespaces | grep "til"
    ```

    Set the environment variable:

    ```bash
    export TILLER_NAMESPACE=<tiller-namespace>
    ```

1. Set more required environment variables for installation:

    Set the namespace (project) you intend to deploy to; usually the same namespace as the TILLER_NAMESPACE, must match what you had set in step 7 in [Setting up Environment](#setting-up-environment):

    ```bash
    export TARGET_NAMESPACE=<target-namespace>
    ```

    Set the external Docker registry hostname, see step 3 in [Setting up Environment](#setting-up-environment):

    ```bash
    export EXTERNAL_DOCKER_REGISTRY=<docker-registry-hostname>
    ```

    Set the absolute path to the unpacked version of the Watson Language Translator archive directory:

    ```bash
    export PATH_TO_ARCHIVE=</absolute/path/to/unpacked/ppa>
    ```

    Set the storage class that was also setup in step 7 in [Setting up Environment](#setting-up-environment). This value will be ignored if you do a development deploy without persistence enabled:

    ```bash
    export STORAGE_CLASS=<storage-class>
    ```

    Set the filename of the override values YAML file, e.g. `dev-values-override.yaml`:

    ```bash
    export CONFIGURATION_FILE=<filename of the config file>
    ```

    Set the release name for this installation, i.e. a short name to identify your Watson Language Translator installation:

    ```bash
    export RELEASE_NAME=<release-name>
    ```

1. Install the Watson Language Translator service:

    ```bash
    ./deploy.sh \
      --docker_registry_prefix=docker-registry.default.svc:5000/${TARGET_NAMESPACE} \
      --external_registry_prefix=${EXTERNAL_DOCKER_REGISTRY}/${TARGET_NAMESPACE} \
      --target_namespace=${TARGET_NAMESPACE} \
      -d ${PATH_TO_ARCHIVE} \
      -s ${STORAGE_CLASS} \
      -O ${PATH_TO_ARCHIVE}/${CONFIGURATION_FILE} \
      -n ${TILLER_NAMESPACE} \
      -e ${RELEASE_NAME}
    ```

### Verifying the installation

In the following, note that on CPD 2.1 installations, the configured release name is usually prefixed with the target namespace, e.g. `zen-MyRelease`.

1. Wait for the installation to finish. Once the `{release-name}-ibm-watson-lt-api` deployment is in ready state, the installation is complete. Run the following command to monitor the rollout status of this deployment:

    ```bash
    oc rollout status deploy {release-name}-ibm-watson-lt-api
    ```

1. See the instruction (from NOTES.txt within chart) after the helm installation completes for chart verification. The instruction can also be viewed by running the command:

    ```bash
    helm status {release-name} --tls
    ```

1. Test the installation by running:

    ```bash
    helm test {release-name} --tls --cleanup
    ```

    Note:

    * Remove the `--tls` flag for the described helm commands if your helm installation is not secured over tls.
    * Remove the `--cleanup` flag if you want to keep the test pod, e.g. to look at its logs.


1. Navigate to your Cloud Pak for Data home page and provision a Watson Language Translator service instance:

    Get the hostname of the remote cluster where Watson Language Translator is being installed:

    ```bash
    oc get routes -n ${TARGET_NAMESPACE}
    ```

    In a browser, enter `https://<hostname>:31843` in the address field and log in. Open the Add-ons page or Services page (located near the top right corner of the page) and select the Watson Language Translator tile. Select Provision instance in the menu.

### Uninstalling the Chart

To uninstall and delete the {release-name} deployment, run the following command:

```bash
helm delete --tls {release-name}
```

To irrevocably uninstall and delete the `{release-name}` deployment, run the following command:

```bash
helm delete --purge --tls {release-name}
```

If you omit the `--purge` option, Helm deletes all resources for the deployment, but retains the record with the release name. This allows you to roll back the deletion. If you include the `--purge` option, Helm removes all records for the deployment, so that the name can be used for another installation.

## Helpful Hints

### Use Helm for a Remote Cluster

* Set the local environment variable export TILLER_NAMESPACE
* Run helm init --client-only
* Run helm list --tls to see the installed helm releases
* If certificates for helm are needed, try running:

    ```bash
    ca_pem_file_path="${HOME}/.helm/ca.pem"
    cert_pem_file_path="${HOME}/.helm/cert.pem"
    key_pem_file_path="${HOME}/.helm/key.pem"

    echo "write ca.pem file to ${ca_pem_file_path}"
    echo `oc get secret -n ${TARGET_NAMESPACE} helm-secret -o go-template --template='{{ index .data "ca.cert.pem"}}' |  base64 --decode > ${ca_pem_file_path}`
    oc get secret -n ${TARGET_NAMESPACE} helm-secret -o go-template --template='{{ index .data "ca.cert.pem"}}' |  base64 --decode > ${ca_pem_file_path}

    echo "write helm.key.pem to ${key_pem_file_path}"
    echo `oc get secret -n ${TARGET_NAMESPACE} helm-secret -o go-template --template='{{ index .data "helm.key.pem"}}' |  base64 --decode > ${key_pem_file_path}`
    oc get secret -n ${TARGET_NAMESPACE} helm-secret -o go-template --template='{{ index .data "helm.key.pem"}}' |  base64 --decode > ${key_pem_file_path}

    echo "write helm.cert.pem to ${cert_pem_file_path}"
    echo `oc get secret -n ${TARGET_NAMESPACE} helm-secret -o go-template --template='{{ index .data "helm.cert.pem"}}' |  base64 --decode > ${cert_pem_file_path}`
    oc get secret -n ${TARGET_NAMESPACE} helm-secret -o go-template --template='{{ index .data "helm.cert.pem"}}' |  base64 --decode > ${cert_pem_file_path}
    ```

### How to Remove a Watson Language Translator Installation

For example, in cases of misconfiguration, a Helm installation might fail or a Helm deinstallation might not proceed properly. To fully delete a Helm release, add the following function to .bashrc:

```bash
function helm_clean (){
  set -x
  helm delete "$@" --tls --purge
  kubectl delete jobs -l release="$@"
  kubectl delete deployments -l release="$@"
  kubectl delete replicaset -l release="$@"
  kubectl delete pod -l release="$@"
  kubectl delete serviceaccount -l release="$@"
  kubectl delete role -l release="$@"
  kubectl delete rolebindings -l release="$@"
  kubectl delete secret -l release="$@"
  kubectl delete service -l release="$@"
  kubectl delete configmaps -l release="$@"
  kubectl delete statefulsets -l release="$@"
  kubectl delete pvc -l release="$@"
  set +x
}
```

And then run:

```bash
. <path to your bashrc>
helm_clean <release-name>
```

## Limitations

* Watson Language Translator can currently run only on Intel 64-bit architecture.
* Datastores (PostgreSQL, MinIO) only support block storage for persistence.
* This chart should only use the default image tags provided with the chart. Different image versions might not be compatible with different versions of this chart.
* The chart must be installed by a cluster administrator. See [Pre-install steps](#pre-install-steps).
* Release names cannot be longer than 20 characters, should be lower case characters.

## Documentation

Find out more about IBM Watson Language Translator for Cloud Pak for Data by reading the [product documentation](https://cloud.ibm.com/docs/services/language-translator-data).

## Configuration

### Global parameters

| Parameter | Description | Default |
|-----------|-------------|---------|
| `global.icpDockerRepo` | When installing via the CLI, this value must be set to `<cluster_CA_domain>:8500/<namespace>/` | `""` |
| `global.image.repository` | When installing via the CLI, this value must be set to `<cluster_CA_domain>:8500/<namespace>/` | `""`|

### Add-on parameters

| Parameter | Description | Default |
|-----------|-------------|---------|
|`gateway.addonService.zenNamespace` | Namespace where the add-on is installed | `zen`|

### API parameters

| Parameter | Description | Default |
|-----------|-------------|---------|
| `api.replicas` | number of replicas | `1` |
| `api.resources.cpuRequestMillis` | requested milli CPUs per pod | `200` |
| `api.resources.cpuLimitMillis` | not guaranteed maximum milli CPU limit per pod | `1000` |
| `api.resources.memoryRequestMB` | requested memory in MB per pod | `256` |
| `api.resources.memoryLimitMB` | not guaranteed maximum memory in MB per pod | `512` |
| `api.config.rootLogLevel` | log level, allowed values: `trace`, `debug`, `info`, `warn`, `error` | `error` |
| `api.config.request_throttling` | allowed requests/s per pod | `500` |

### Documents parameters

| Parameter | Description | Default |
|-----------|-------------|---------|
| `documents.replicas` | number of replicas | `1` |
| `documents.resources.cpuRequestMillis` | requested milli CPUs per pod | `200` |
| `documents.resources.cpuLimitMillis` | not guaranteed maximum milli CPU limit per pod | `1000` |
| `documents.resources.memoryRequestMB` | requested memory in MB per pod | `500` |
| `documents.resources.memoryLimitMB` | not guaranteed maximum memory in MB per pod | `1000` |
| `documents.config.rootLogLevel` | log level, allowed values: `trace`, `debug`, `info`, `warn`, `error` | `warn` |

### Language ID (LID) parameters

| Parameter | Description | Default |
|-----------|-------------|---------|
| `lid.replicas` | number of replicas | `1` |
| `lid.resources.cpuRequestMillis` | requested milli CPUs per pod | `250` |
| `lid.resources.cpuLimitMillis` | not guaranteed maximum milli CPU limit per pod | `750` |
| `lid.resources.memoryRequestMB` | requested memory in MB per pod | `2000` |
| `lid.resources.memoryLimitMB` | not guaranteed maximum memory in MB per pod | `2000` |

### Segmenter parameters (Sentence Splitting)

| Parameter | Description | Default |
|-----------|-------------|---------|
| `segmenter.replicas` | number of replicas | `1` |
| `segmenter.resources.cpuRequestMillis` | requested milli CPUs per pod | `250` |
| `segmenter.resources.cpuLimitMillis` | not guaranteed maximum milli CPU limit per pod | `750` |
| `segmenter.resources.memoryRequestMB` | requested memory in MB per pod | `2500` |
| `segmenter.resources.memoryLimitMB` | not guaranteed maximum memory in MB per pod | `2500` |

### Translation backend parameters (applied to all configured language pairs)

| Parameter | Description | Default |
|-----------|-------------|---------|
| `translation.replicas` | number of replicas | `1` |
| `translation.resources.cpuRequestMillis` | requested milli CPUs per pod | `1000` |
| `translation.resources.cpuLimitMillis` | not guaranteed maximum milli CPU limit per pod | `5000` |
| `translation.resources.memoryRequestMB` | requested memory in MB per pod | `3500` |
| `translation.resources.memoryLimitMB` | not guaranteed maximum memory in MB per pod | `5000` |

### MinIO parameters (S3 compatible object storage)

| Parameter | Description | Default |
|-----------|-------------|---------|
| `s3.replicas` | Number of replicas of the pod, must be even number. | `4` |
| `s3.persistence.enabled` | Use persistent volume to store data | `false` |
| `s3.persistence.size` | Size of the persistent volume created. | `10Gi` |
| `s3.persistence.storageClass` |  Storage class name for minio. `portworx-sc`, `vsphere-volume` and `glusterfs` are supported. | `""` |
| `s3.persistence.useDynamicProvisioning` | True to allow the cluster to automatically provision new storage resource and create PersistentVolume objects. | `true` |

### PostgreSQL parameters

| Parameter | Description | Default |
|-----------|-------------|---------|
| `postgres.keeper.replicas` | Number of keeper pods in statefulset | `3` |
| `postgres.persistence.enabled` | Use persistent volume to store data | `false` |
| `postgres.persistence.size` | Size of the persistent volume created. | `10Gi` |
| `postgres.persistence.storageClassName` | Storage class name for PostgreSQL: `portworx-sc` and `vsphere-volume` are supported. | `""` |
| `postgres.persistence.useDynamicProvisioning` | True to allow the cluster to automatically provision new storage resource and create PersistentVolume objects. | `true` |

### Language Support Configuration

#### Language Pak1

| Parameter | Description | Default |
|-----------|-------------|---------|
| `translationModels.ar-en.enabled` | Toggle Arabic to English translation | `false` |
| `translationModels.de-en.enabled` | Toggle German to English translation | `false` |
| `translationModels.en-ar.enabled` | Toggle English to Arabic translation | `false` |
| `translationModels.en-de.enabled` | Toggle English to German translation | `false` |
| `translationModels.en-es.enabled` | Toggle English to Spanish translation | `false` |
| `translationModels.en-fr.enabled` | Toggle English to French translation | `false` |
| `translationModels.en-he.enabled` | Toggle English to Hebrew translation | `false` |
| `translationModels.en-it.enabled` | Toggle English to Italian translation | `false` |
| `translationModels.en-pt.enabled` | Toggle English to Portuguese (Brazilian) translation | `false` |
| `translationModels.en-ru.enabled` | Toggle English to Russian translation | `false` |
| `translationModels.en-tr.enabled` | Toggle English to Turkish translation | `false` |
| `translationModels.en-zh.enabled` | Toggle English to Chinese (Simplified) translation | `false` |
| `translationModels.en-zh-TW.enabled` | Toggle English to Chinese (Traditional) translation | `false` |
| `translationModels.es-en.enabled` | Toggle Spanish to English translation | `false` |
| `translationModels.fr-en.enabled` | Toggle French to English translation | `false` |
| `translationModels.he-en.enabled` | Toggle Hebrew to English translation | `false` |
| `translationModels.it-en.enabled` | Toggle Italian to English translation | `false` |
| `translationModels.pt-en.enabled` | Toggle Portuguese (Brazilian) to English translation | `false` |
| `translationModels.ru-en.enabled` | Toggle Russian to English translation | `false` |
| `translationModels.tr-en.enabled` | Toggle Turkish to English translation | `false` |
| `translationModels.zh-en.enabled` | Toggle Chinese (Simplified) to English translation | `false` |
| `translationModels.zh-TW-en.enabled` | Toggle Chinese (Traditional) to English translation | `false` |

#### Language Pak2

| Parameter | Description | Default |
|-----------|-------------|---------|
| `translationModels.en-hi.enabled` | Toggle English to Hindi translation | `false` |
| `translationModels.en-id.enabled` | Toggle English to Indonesian translation | `false` |
| `translationModels.en-ja.enabled` | Toggle English to Japanese translation | `false` |
| `translationModels.en-ko.enabled` | Toggle English to Korean translation | `false` |
| `translationModels.en-ms.enabled` | Toggle English to Malay translation | `false` |
| `translationModels.en-th.enabled` | Toggle English to Thai translation | `false` |
| `translationModels.hi-en.enabled` | Toggle Hindi to English translation | `false` |
| `translationModels.id-en.enabled` | Toggle Indonesian to English translation | `false` |
| `translationModels.ja-en.enabled` | Toggle Japanese to English translation | `false` |
| `translationModels.ko-en.enabled` | Toggle Korean to English translation | `false` |
| `translationModels.ms-en.enabled` | Toggle Malay to English translation | `false` |
| `translationModels.th-en.enabled` | Toggle Thai to English translation | `false` |

#### Language Pak3

| Parameter | Description | Default |
|-----------|-------------|---------|
| `translationModels.bg-en.enabled` | Toggle Bulgarian to English translation | `false` |
| `translationModels.ca-es.enabled` | Toggle Catalan to Spanish translation | `false` |
| `translationModels.cs-en.enabled` | Toggle Czech to English translation | `false` |
| `translationModels.da-en.enabled` | Toggle Danish to English translation | `false` |
| `translationModels.de-fr.enabled` | Toggle German to French translation | `false` |
| `translationModels.de-it.enabled` | Toggle German to Italian translation | `false` |
| `translationModels.el-en.enabled` | Toggle Greek to English translation | `false` |
| `translationModels.en-bg.enabled` | Toggle English to Bulgarian translation | `false` |
| `translationModels.en-cs.enabled` | Toggle English to Czech translation | `false` |
| `translationModels.en-da.enabled` | Toggle English to Danish translation | `false` |
| `translationModels.en-el.enabled` | Toggle English to Greek translation | `false` |
| `translationModels.en-et.enabled` | Toggle English to Estonian translation | `false` |
| `translationModels.en-fi.enabled` | Toggle English to Finnish translation | `false` |
| `translationModels.en-ga.enabled` | Toggle English to Irish translation | `false` |
| `translationModels.en-hr.enabled` | Toggle English to Croatian translation | `false` |
| `translationModels.en-hu.enabled` | Toggle English to Hungarian translation | `false` |
| `translationModels.en-lt.enabled` | Toggle English to Lithuanian translation | `false` |
| `translationModels.en-nb.enabled` | Toggle English to Norwegian Bokmål translation | `false` |
| `translationModels.en-nl.enabled` | Toggle English to Dutch translation | `false` |
| `translationModels.en-pl.enabled` | Toggle English to Polish translation | `false` |
| `translationModels.en-ro.enabled` | Toggle English to Romanian translation | `false` |
| `translationModels.en-sk.enabled` | Toggle English to Slovak translation | `false` |
| `translationModels.en-sl.enabled` | Toggle English to Slovenian translation | `false` |
| `translationModels.en-sv.enabled` | Toggle English to Swedish translation | `false` |
| `translationModels.es-ca.enabled` | Toggle Spanish to Catalan translation | `false` |
| `translationModels.es-fr.enabled` | Toggle Spanish to French translation | `false` |
| `translationModels.et-en.enabled` | Toggle Estonian to English translation | `false` |
| `translationModels.fi-en.enabled` | Toggle Finnish to English translation | `false` |
| `translationModels.fr-es.enabled` | Toggle French to Spanish translation | `false` |
| `translationModels.ga-en.enabled` | Toggle Irish to English translation | `false` |
| `translationModels.hr-en.enabled` | Toggle Croatian to English translation | `false` |
| `translationModels.hu-en.enabled` | Toggle Hungarian to English translation | `false` |
| `translationModels.it-de.enabled` | Toggle Italian to German translation | `false` |
| `translationModels.lt-en.enabled` | Toggle Lithuanian to English translation | `false` |
| `translationModels.nb-en.enabled` | Toggle Norwegian Bokmål to English translation | `false` |
| `translationModels.pl-en.enabled` | Toggle Polish to English translation | `false` |
| `translationModels.ro-en.enabled` | Toggle Romanian to English translation | `false` |
| `translationModels.sk-en.enabled` | Toggle Slovak to English translation | `false` |
| `translationModels.sl-en.enabled` | Toggle Slovenian to English translation | `false` |
| `translationModels.sv-en.enabled` | Toggle Swedish to English translation | `false` |