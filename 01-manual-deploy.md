# Part 1 - Manual deploy 

## Prerequisites

### Git client

The git client needs to be installed in your development operating system. It comes as standard for Mac OS.

https://git-scm.com/

### IBM Cloud cli

The IBM Cloud cli is required for management of IBM Cloud Account and management of your managed IBM Kubernetes and Red Hat OpenShift clusters - https://cloud.ibm.com/docs/cli?topic=cloud-cli-getting-started

Don’t install just the IBM Cloud CLI, install the IBM Cloud CLI and Developer Tools

```
curl -sL https://ibm.biz/idt-installer | bash
```

Note: If you log in to the web UI using SSO, you’ll need to create an API key for logging into the CLI. (You can also use this API key for installing the Developer Environment.)

### OpenShift cli

The OpenShift cli is required for Red Hat OpenShift management and development

1. Download the appropriate client tar ball from the mirror site - https://mirror.openshift.com/pub/openshift-v4/clients/ocp/latest/

2. Unpack the tar ball

3. Copy the `oc` and `kubectl` from the unpacked folder into your terminal PATH (e.g. /usr/local/bin)

## Deploy app to Jenkins on OCP 4.3

For these exercises you will be asked for a `DEV_NAMESPACE`. That value should be the name 
you claimed in the box note prefixed to `-dev` (e.g. user01-dev)

### 1. Set an environment variable for your namespace

1. Export an environment variable. This will be used 

```
export DEV_NAMESPACE="userXX-dev"
```

### 2. Log into the cluster

1. Open the IBM Cloud console - https://cloud.ibm.com

2. Select Kedar's account from the account list

3. Open the `Resource List` from the menu and filter the list by setting the resource group filter to `faststart-one`

4. Find the `faststart-ap-cluster` in the list and click on it to open the overview page

5. Click on the `OpenShift web console` button to launch the OpenShift console

6. Within the console, click on the drop-down in the top-right that contains your user id and select `Copy Login Command`

7. Click the `Display Token` link and copy the login command shown. It should look something like the following:

    ```
    oc login --token=XXX --server=XXX
    ```

8. Run the command from your command-line to log in to the cluster

### 3. Create a new namespace

We will all be working in the same cluster but in different namespaces. You will need to create a namespace based on the name that you were assigned.

1. Run the following to create the namespace

```
oc create namespace "${DEV_NAMESPACE}"
```

### 4. Provision Jenkins ephemeral

Jenkins ephemeral provides a kubernetes native version of Jenkins that dynamically provisions build agents on-demand. It's
_ephemeral_ meaning it doesn't allocate any persistent storage in the cluster.

1. Run the following command to provision the Jenkins instance in your namespace

```
oc new-app jenkins-ephemeral -n "${DEV_NAMESPACE}"
```

2. Open the OpenShift console as described in the login steps above

3. Select `Workloads -> Pods` from the left-hand menu

4. At the top of the page select your project/namespace (e.g. user01-dev) from the drop-down list to see the Jenkins instance running

### 5. Create a project from the template in the FastStart git org

For the lab, we need a deployable application to run the pipeline. We will use a pre-built Code Pattern
to expedite things.

1. Open a browser to https://github.com/IBM/template-node-typescript
2. Click the green `Use this template` button
3. On the following page, select the `faststart-dev-lab` org, and give the repo a unique name.
5. Once you press `Create` it will fork the repo without the history into the new repo.

### 6. Copy the pull secrets into your namespace

Our pipeline will be using the IBM Cloud container registry to store the built images. In order for the cluster to read from the registry during deployment into the namespace, the secrets containing the credentials need to be added to the namespace. The following steps will set them up.

1. Download the shell script that sets up the pull secrets

```
curl -O https://raw.githubusercontent.com/ibm-garage-cloud/garage-terraform-modules/master/generic/cluster/namespaces/scripts/setup-namespace-pull-secrets.sh
```

2. Make the file executable

```
chmod +x setup-namespace-pull-secrets.sh
```

3. Run the script to copy the pull secrets into the namespace and add them to the default service account

```
./setup-namespace-pull-secrets.sh ${DEV_NAMESPACE}
```

### 7. Create the ConfigMap and Secret to describe the cluster

The pipeline needs certain information about the cluster to be able to push the image to the registry and 
deploy the image to the cluster. Instead of hard coding those values into the Jenkinsfile, we will store them
in a ConfigMap and Secret.

1. Log into the ibmcloud cli

```
ibmcloud login -r au-syd -g faststart-one [--sso]
```

where:
 - `resource-group` is your assigned resource group (e.g. `faststart-one`)
 - `--sso` may be needed if you are configured for SSO login

2. Get the cluster information

```
ibmcloud ks cluster get --cluster faststart-ap-cluster
```

3. Copy the following into a file named ibmcloud-config.yaml and update the values from the output of the previous command

```
apiVersion: v1
kind: ConfigMap
metadata:
  labels:
    app: ibmcloud-config
    group: catalyst-tools
  name: ibmcloud-config
data:
  APIURL: 'https://cloud.ibm.com'
  CLUSTER_NAME: faststart-ap-cluster
  CLUSTER_TYPE: openshift
  INGRESS_SUBDOMAIN: {INGRESS_SUBDOMAIN}
  REGION: au-syd
  REGISTRY_NAMESPACE: faststart-one
  REGISTRY_URL: au.icr.io
  RESOURCE_GROUP: {RESOURCE_GROUP}
  SERVER_URL: {MASTER_URL}
  TLS_SECRET_NAME: {INGRESS_SECRET_NAME}
---
apiVersion: v1
kind: Secret
metadata:
  labels:
    app: ibmcloud-config
    group: catalyst-tools
  name: ibmcloud-apikey
type: Opaque
stringData:
  APIKEY: {APIKEY}
  REGISTRY_USER: iamapikey
```

where:
 - `{APIKEY}` is the one provided in the box note
   
6. Create the resources in the cluster

```
kubectl create -n ${DEV_NAMESPACE} -f ibmcloud-config.yaml
```

where:
 - `${DEV_NAMESPACE}` should be the name you claimed in the box note prefixed to `-dev` (e.g. user01-dev)

### 8. Create git secret

In order for Jenkins have access to the git repository, particularly if it is a private repository, a Kubernetes secret 
needs to be added that contains the git credentials.

1. Create a personal access token (if you don't already have one) using the following instructions - https://cloudnativetoolkit.dev/getting-started/prereqs#configure-github-personal-access-token

2. Copy the following into a file called `gitsecret.yaml` and update the {Git-Username}, and {Git-PAT}

```
apiVersion: v1
kind: Secret
metadata:
  annotations:
    build.openshift.io/source-secret-match-uri-1: https://github.com/ibm-garage-cloud/*
  labels:
    jenkins.io/credentials-type: usernamePassword
  name: git-credentials
type: kubernetes.io/basic-auth
stringData:
  username: {Git-Username}
  password: {Git-PAT}
```

where:
 - `Git-Username` is the username that has access to the git repo
 - `Git-PAT` is the personal access token of the git user

2. Assuming you are still logged into the cluster, create the secret by running the following:

```
kubectl create -n ${DEV_NAMESPACE} -f gitsecret.yaml
```

where:
 - `${DEV_NAMESPACE}` should be the name you claimed in the box note prefixed to `-dev` (e.g. user01-dev)

### 9. Create the build config

On OpenShift 4.3, Jenkins is built into the OpenShift pipelines and the build pipelines can be managed using Kubernetes 
custom resources. We will create one by hand to create the build pipeline for our new application.

1. Copy the following into a file called `buildconfig.yaml` and update the {Name}, {Secret}, {Git-Repo-URL}, and {Namespace}

```
apiVersion: v1
kind: BuildConfig
metadata:
  name: {Name}
spec:
  triggers:
  - type: GitHub
    github:
      secret: my-secret-value
  source:
    git:
      uri: {Git-Repo-URL}
      ref: master
  strategy:
    jenkinsPipelineStrategy:
      jenkinsfilePath: Jenkinsfile
      env:
      - name: CLOUD_NAME
        value: openshift
      - name: NAMESPACE
        value: {Namespace}
```

where:
 - `Name` is in the name of your pipeline 
 - `Git-Repo-URL` is the https url to the git repository
 - `Namespace` is the dev namespace where the app will be deployed

2. Assuming you are still logged into the cluster, create the buildconfig resource in the cluster

```
oc create -n ${DEV_NAMESPACE} -f buildconfig.yaml
```

where:
 - `${DEV_NAMESPACE}` should be the name you claimed in the box note prefixed to `-dev` (e.g. user01-dev)

### 10. View the pipeline in the OpenShift console

1. Open the OpenShift console for the cluster
2. Select your project/namespace (i.e. `${DEV_NAMESPACE}`)
3. Select Builds -} Pipelines
4. The build pipeline that was created in the previous step should appear
5. Manually trigger the pipeline by pressing the `Build` button

### 11. Create the webhook

1. Run the following to get the webhook details from the build config 

```
oc describe bc {Name} -n ${DEV_NAMESPACE}
```

where:
 - `{Name}` is the name used in the previous step for the build config
 - `${DEV_NAMESPACE}` should be the name you claimed in the box note prefixed to `-dev` (e.g. user01-dev)

The webhook url will have a structure similar to:

`http://{openshift_api_host:port}/oapi/v1/namespaces/{namespace}/buildconfigs/{name}/webhooks/{secret}/generic`

In our case `{secret}` will be `my-secret-value`

2. Open a browser to the GitHub repo deployed in the previous step in the build config

3. Select `Settings` then `Webhooks`. Press `Add webhook`

4. Paste the webhook url from the previous step into the `Payload url`. Leave the rest of the values as the defaults and press `Add webhook`

5. Press the button to test the webhook to ensure that everything was done properly

6. Go back to your project code and push a change to one of the files
   
7. Go to the Build pipeline page in the OpenShift console to see that the build was triggered
