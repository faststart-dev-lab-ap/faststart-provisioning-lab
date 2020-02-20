# Part 2 - Terraform

## Provision dev tools with scripts

### 1. Look at the provided terraform modules

The automated scripts have been provided in two parts - 1) a set of reusable, pluggable modules and 2) a set of "stages"
that put those modules together to set up an environment.

1. Clone the terraform modules repo - https://github.com/ibm-garage-cloud/garage-terraform-modules
2. Take a look through the way the modules are organized
3. Look at the structure of one module, e.g. https://github.com/ibm-garage-cloud/garage-terraform-modules/tree/master/generic/cluster/namespaces

Each module has the same basic structure:
- Module input variables are defined in `variables.tf`
- Module output variables are defined in `outputs.tf`
- Module logic is defined in `main.tf`
  
### 2. Setup the terraform scripts

For the purposes of the lab and for the sake of time, a stripped down version of the `iteration-zero` repository has been
created that installs a subset of the basic components. It requires a little bit of setup before the scripts can be run
to allow the script to access the cluster.

1. Clone the faststart iteration-zero repo - https://github.com/ibm-garage-cloud/iteration-zero-faststart
2. Copy `credentials.template` into a file named `credentials.properties`
3. In `credentials.properties`, replace {IBMCLOUD_API_KEY} with the APIKEY from the top of the box note
4. Open `terraform/settings/environment.tfvars". Set the 'cluster_name' and the 'resource_group_name' to 
match the cluster you've been assigned

### 3. Run the terraform scripts

Once the scripts are configured, they can be executed. A docker image is provided that sets up the tools required for the
scripts to run.

1. Start the docker image to create the CLI environment by running `./launch.sh`. You should see an IBM Cloud banner
2. Kick off the process by running `./runTerraform.sh`. You will see a screen that confirms the settings that will be applied. Select Y
3. The script will prompt for the dev_namespace. Provide the value used above (e.g. user01-dev)
4. In about 5 minutes the process will complete (note: the tools may not be available yet)
5. Open the OpenShift console to look at the deployments and pods in your namespace

### 4. Set up the Artifactory instance

The oss version of Artifactory we are using doesn't allow CLI access so there are a couple of manual steps that 
need to be performed to make Artifactory available to the pipeline.

1. Follow the steps described here to complete the installation of Artifactory - https://ibm-garage-cloud.github.io/ibm-garage-developer-guide/admin/artifactory-setup
2. Once the steps are completed, open the OpenShift console, select your project/namespace (userXX-dev) and go to Build -> Pipelines
3. Select `Start Build` to kickoff your Build
4. Once the build has completed, within the OpenShift console go to Networking -> Routes and click on the Artifactory route
5. Log in with admin/password
6. Click the top icon on the left to open the registry list
7. Expand `generic-local` and you should see a folder that matches the resource group name (e.g. faststart-one)
8. Expand the resource group folder and you should see an index.yaml and helm chart tarball that were produced by the
build

### 5. Register Artifactory helm repository with ArgoCD

In order for ArgoCD to deploy helm charts from a helm repository, those helm repositories must first be registered.
Since the final steps to configure Artifactory were manual, there are a couple of manual steps required to 
register the Artifactory helm repository with ArgoCD.

1. Use the following steps to register the Artifactory helm repository with ArgoCD - https://ibm-garage-cloud.github.io/ibm-garage-developer-guide/admin/argocd-setup

### 6. Enable ArgoCD in the CI pipeline

The Jenkins pipeline from the sample application has a final stage that will trigger the CD pipeline. To do that, a 
GitOps repository first needs to be configured with the details of the application that will be deployed. After that 
the GitOps repository will need to be made available to the pipeline through a Secret.

1. Export an environment variable for your namespace

```
export TEST_NAMESPACE="userXX-test"
```

where:
- `userXX` is the user id assigned

2. Create a namespace that will be used as your "test" environment

```
oc create namespace ${TEST_NAMESPACE}
```

3. Follow the instructions found here to set up the ArgoCD pipeline - https://ibm-garage-cloud.github.io/ibm-garage-developer-guide/guides/continuous-delivery

**Note**
When configuring the application within ArgoCD, use the TEST_NAMESPACE for the destination namespace
