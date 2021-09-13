# Operator Certification Pipeline<br/>Alpha Instructions

## Table of Contents
* [Before you start](#before-you-start)
* [Installation](#installation)
  * [Step 1 - Install OpenShift Pipelines Operator](#step1) 
  * [Step 2 - Configure the OpenShift CLI tool](#step2)
  * [Step 3 - Add the Kubeconfig secret](#step3)
  * [Step 4 - Import Red Hat Catalogs](#step4)
  * [Step 5 - Install the Certification Pipeline and dependencies into the cluster](#step5)
* [Optional configuration](#optional-config)
  * [Optional Step - If using digest pinning (recommended)](#digest-pinning-config)
  * [Optional Step - If submitting your certification results to Red Hat](#submit-results)
  * [Optional Step - If using a private container registry](#private-registry)
* [Pipeline Execution (Development/Iteration)](#execute-pipeline)
  * [Minimal Pipeline Run](#minimal-pipeline-run)
  * [Pipeline Run with Image Digest Pinning](#img-digest-pipeline-run)
  * [Pipeline Run with a Private Container Registry](#private-registry-pipeline-run)
* [Submit Results to Red Hat](#submit-result)
  * [Submit results from Minimal Pipeline Run](#submit-result-minimal)
  * [Submit results with Image Digest Pinning](#submit-result-img-digest)
  * [Submit results with a private container registry](#submit-result-private-registy)
  * [Submit results with Image Digest Pinning and a private container registry](#submit-result-registy-and-pinning)


## <a id="before-you-start"></a>What you'll need before you start:
1. An OpenShift Cluster *(recommended version 4.8 or above)*
2. Kubeconfig file for a user with cluster admin privileges 
3. A Git repo that contains the contents of your Operator Bundle
4. [Install](https://docs.openshift.com/container-platform/4.8/cli_reference/openshift_cli/getting-started-cli.html#installing-openshift-cli) `oc`, the OpenShift CLI tool
5. [Install](https://tekton.dev/docs/cli/) `tkn`, the Tekton CLI tool

## <a id="installation"></a>Installation

### <a id="step1"></a>Step 1 - Install OpenShift Pipelines Operator
* Log into your cluster's OpenShift Console with cluster admin privileges
* Use the left hand menu to navigate to *Operators*
* In the *Operators* submenu click on *OperatorHub*
* Use the Filter/Search box to filter on *OpenShift Pipelines*
* Click the *Red Hat OpenShift Pipelines* tile
* In the fly out mneu to the right click the *Install* button near the top
* On the next screen "Install Operator" scroll to the bottom of the page and click *Install*

### <a id="step2"></a>Step 2 - Configure the OpenShift CLI tool (oc)
* Open a terminal window
* Set the `KUBECONFIG` environment variable
```bash
export KUBECONFIG=/path/to/your/cluster/kubeconfig
```

### <a id="step3"></a>Step 3 - Add the Kubeconfig secret
```bash
  oc create secret generic kubeconfig --from-file=kubeconfig=$KUBECONFIG
```
> *This kubeconfig will be used by to deploy the Operator under test and run the certificaiton checks.*

### <a id="step4"></a>Step 4 - Import Red Hat Catalogs
```bash
oc import-image certified-operator-index \
  --from=registry.redhat.io/redhat/certified-operator-index \
  --reference-policy local \
  --scheduled \
  --confirm \
  --all
```

```bash
oc import-image redhat-marketplace-index \
  --from=registry.redhat.io/redhat/redhat-marketplace-index \
  --reference-policy local \
  --scheduled \
  --confirm \
  --all
```

### <a id="step5"></a>Step 5 - Install the Certification Pipeline and dependencies into the cluster
```bash
git clone git@github.com:redhat-openshift-ecosystem/operator-pipelines.git
cd operator-pipelines
oc apply -R -f ansible/roles/operator-pipeline/templates/openshift/pipelines
oc apply -R -f ansible/roles/operator-pipeline/templates/openshift/tasks
oc apply -f  https://raw.githubusercontent.com/tektoncd/catalog/main/task/yaml-lint/0.1/yaml-lint.yaml
oc apply -f  https://raw.githubusercontent.com/tektoncd/catalog/main/task/git-clone/0.4/git-clone.yaml
```


# <a id="optional-config"></a>Optional Configuration

## <a id="digest-pinning-config"></a>Optional Step - If using digest pinning (recommended)
The pipeline offers functionality that will automatically replace all image tags in your Bundle with Image Digest SHAs.This allows the pipeline to ensure it is using a pinned version of all images. The pipeline commit the pinned version of your Bundle to your GitHub repo as a new branch.  In order to do this a private key with access to GitHub needs to be added to your cluster as a secret. 

### <a id="private-key"></a>Add private key to access GitHub repo with Operator Bundle source
Base64 encode a private key with access to GitHub repo containing the Bundle
```bash
base64 /path/to/private/key
```

Create a secrete that contains the base64 encoded private key
```bash
cat << EOF > ssh-secret.yml
kind: Secret
apiVersion: v1
metadata:
  name: github-ssh-credentials
data:
  id_rsa: |
        <base64 encoded private key>
EOF
```

Add the secret to the cluster
```bash
oc create -f ssh-secret.yml
```

## <a id="submit-results"></a>Optional Step - If submitting your certification results to Red Hat

### <a id="github-api-token"></a>Add a GitHub API Token for the repo where the PR will be created

Upon completion the Pipeline can automatically open a Pull Request to submit your Operator to Red Hat. To enable this functionally, add a GitHub API Token and use `--param submit=true` when running the pipeline. 

```bash
oc create secret generic github-api-token --from-literal GITHUB_TOKEN=<github token>
```

### <a id="container-api-key"></a>Add Red Hat Container API access key
You can retrieve a Red Hat Container API Key by logging into connect.redhat.com as a technology partner. Once logged in, navigate to `Product Certification` > `Manage container API keys`
```bash
oc create secret generic pyxis-api-secret --from-literal PYXIS_API_KEY.txt=< API KEY >
```

## <a id="private-registry"></a>Optional Step - If using a private container registry
By default the Pipeline will use the OpenShift Container Registry running in cluster.  If you would like to use a private registry you will need to add credentials for that registry.

### <a id="container-registry-creds"></a>Add credentials for the Container Registry
The Pipeline with automatically build your Operator Bundle Image as well as a Bundle Image Index for testing and verfication.  

By default these images will be create in container registry on cluster but if you would like to leverage an external private registry you can provide the credentials by adding a secret to the cluster. 

```bash
oc create secret docker-registry registry-dockerconfig-secret \
    --docker-server=quay.io \
    --docker-username=<registry username> \
    --docker-password=<registry password> \
    --docker-email=<registry email>
```





# <a id="execute-pipeline"></a>Execute the Pipeline (Development Iterations)
There are multiple ways to execute the Pipeline.  Below are several examples but paramaters and workspaces can be removed or added per your requirements. 

## <a id="minimal-pipeline-run"></a>Minimal Pipeline Run

```bash
GIT_REPO_URL=<enter the URL to the git repo containing the Operaotr bundle>
BUNDLE_PATH=<path to the bundle in the Git Repo>
```

```bash
tkn pipeline start operator-ci-pipeline-serial \
  --param git_repo_url=$GIT_REPO_URL \
  --param git_branch=main \
  --param bundle_path=$BUNDLE_PATH \
  --param env=production \
  --workspace name=pipeline,volumeClaimTemplateFile=templates/workspace-template.yml \
  --workspace name=kubeconfig,secret=kubeconfig \
  --showlog \
  --pod-template templates/crc-pod-template.yml
```
After running this command you will be prompted for several additional parameters. Accept all the defaults. 

> #### Troubleshooting TIp
>
> If you see a Permission Denied error try the GITHUB `HTTPS URL` instead of the `SSH URL`. 

## <a id="img-digest-pipeline-run"></a>Pipeline Run with Image Digest Pinning
* Execute the [Configuration Steps for Digest Pinning](#digest-pinning-config)

```bash
GIT_REPO_URL=<enter the URL to the git repo containing the Operaotr bundle>
BUNDLE_PATH=<path to the bundle in the Git Repo>
GIT_USERNAME=<your github username>
GIT_EMAIL=<your github email address>
```

```bash
tkn pipeline start operator-ci-pipeline-serial \
  --param git_repo_url=$GIT_REPO_URL \
  --param git_branch=main \
  --param bundle_path=$BUNDLE_PATH \
  --param env=production \
  --param pin_digests=true \
  --param git_username=$GIT_USERNAME \
  --param git_email=$GIT_EMAIL \
  --workspace name=pipeline,volumeClaimTemplateFile=templates/workspace-template.yml \
  --workspace name=kubeconfig,secret=kubeconfig \
  --workspace name=ssh-dir,secret=github-ssh-credentials \
  --showlog \
  --pod-template templates/crc-pod-template.yml
```
> #### Troubleshooting TIp
>
> ##### Error: could not read Username for `https://github.com` 
> try using the SSH GitHub URL in `--param git_repo_url`

## <a id="private-registry-pipeline-run"></a>Pipeline Run with a Private Container Registry
* Execute the [Configuration Steps for Private Registries](#private-registry)
```bash
GIT_REPO_URL=<enter the URL to the git repo containing the Operaotr bundle>
BUNDLE_PATH=<path to the bundle in the Git Repo>
GIT_USERNAME=<your github username>
GIT_EMAIL=<your github email address>
REGISTRY=<your image registry.  ie: quay.io>
IMAGE_NAMESPACE=<namespace in the container registry>
```

```bash
tkn pipeline start operator-ci-pipeline-serial \
  --param git_repo_url=$GIT_REPO_URL \
  --param git_branch=main \
  --param bundle_path=$BUNDLE_PATH \
  --param env=production \
  --param pin_digests=true \
  --param git_username=$GIT_USERNAME \
  --param git_email=$GIT_EMAIL \
  --param registry=$REGISTRY \
  --param image_namespace=$IMAGE_NAMESPACE \
  --workspace name=pipeline,volumeClaimTemplateFile=templates/workspace-template.yml \
  --workspace name=kubeconfig,secret=kubeconfig \
  --workspace name=ssh-dir,secret=github-ssh-credentials \
  --workspace name=registry-credentials,secret=registry-dockerconfig-secret \
  --showlog \
  --pod-template templates/crc-pod-template.yml
```

# <a id="submit-result"></a>Submit Results to Red Hat
* Execute the [Configuration Steps for Submitting Results](#submit-results)

In order to submit results add the following `--param`'s and `--workspace` where `$UPSTREAM_REPO_NAME` is equal to the repo where the Pull Request will be submitted. Typically this is a Red Hat Certificaiton repo but you can use a repo of your own for testing.
```bash
--param upstream_repo_name=$UPSTREAM_REPO_NAME #Repo where Pull Request (PR) will be opened
```

```bash
--param submit=true
```

```bash
--workspace name=pyxis-api-key,secret=pyxis-api-secret
```

## <a id="submit-result-minimal"></a>Submit results from Minimal Pipeline Run
```bash
GIT_REPO_URL=<enter the URL to the git repo containing the Operaotr bundle>
BUNDLE_PATH=<path to the bundle in the Git Repo>
UPSTREAM_REPO_NAME=<upstream repo where submission Pull Request is opened>
```

```bash
tkn pipeline start operator-ci-pipeline-serial \
  --param git_repo_url=$GIT_REPO_URL \
  --param git_branch=main \
  --param bundle_path=$BUNDLE_PATH \
  --param upstream_repo_name=$UPSTREAM_REPO_NAME \
  --param submit=true \
  --param env=production \
  --workspace name=pipeline,volumeClaimTemplateFile=templates/workspace-template.yml \
  --workspace name=kubeconfig,secret=kubeconfig \
  --workspace name=pyxis-api-key,secret=pyxis-api-secret \
  --showlog \
  --pod-template templates/crc-pod-template.yml
```

## <a id="submit-result-img-digest"></a>Submit results with Image Digest Pinning
* Execute the [Configuration Steps for Submitting Results](#submit-results)
* Execute the [Configuration Steps for Digest Pinning](#digest-pinning-config)

```bash
GIT_REPO_URL=<enter the URL to the git repo containing the Operaotr bundle>
BUNDLE_PATH=<path to the bundle in the Git Repo>
GIT_USERNAME=<your github username>
GIT_EMAIL=<your github email address>
UPSTREAM_REPO_NAME=<upstream repo where submission Pull Request is opened>
```

```bash
tkn pipeline start operator-ci-pipeline-serial \
  --param git_repo_url=$GIT_REPO_URL \
  --param git_branch=main \
  --param bundle_path=$BUNDLE_PATH \
  --param env=production \
  --param pin_digests=true \
  --param git_username=$GIT_USERNAME \
  --param git_email=$GIT_EMAIL \
  --param upstream_repo_name=$UPSTREAM_REPO_NAME \
  --param submit=true \
  --workspace name=pipeline,volumeClaimTemplateFile=templates/workspace-template.yml \
  --workspace name=kubeconfig,secret=kubeconfig \
  --workspace name=ssh-dir,secret=github-ssh-credentials \
  --workspace name=pyxis-api-key,secret=pyxis-api-secret \
  --showlog \
  --pod-template templates/crc-pod-template.yml
```
> #### Troubleshooting Tip
>
> ##### Error: could not read Username for `https://github.com` 
> try using the SSH GitHub URL in `--param git_repo_url`

## <a id="submit-result-private-registy"></a>Submit results with a private container registry
* Execute the [Configuration Steps for Submitting Results](#submit-results)
* Execute the [Configuration Steps for Private Registries](#private-registry)
```bash
GIT_REPO_URL=<enter the URL to the git repo containing the Operaotr bundle>
BUNDLE_PATH=<path to the bundle in the Git Repo>
GIT_USERNAME=<your github username>
GIT_EMAIL=<your github email address>
REGISTRY=<your image registry.  ie: quay.io>
IMAGE_NAMESPACE=<namespace in the container registry>
UPSTREAM_REPO_NAME=<upstream repo where submission Pull Request is opened>
```

```bash
tkn pipeline start operator-ci-pipeline-serial \
  --param git_repo_url=$GIT_REPO_URL \
  --param git_branch=main \
  --param bundle_path=$BUNDLE_PATH \
  --param env=production \
  --param pin_digests=true \
  --param git_username=$GIT_USERNAME \
  --param git_email=$GIT_EMAIL \
  --param registry=$REGISTRY \
  --param image_namespace=$IMAGE_NAMESPACE \
  --param upstream_repo_name=$UPSTREAM_REPO_NAME \
  --param submit=true \
  --workspace name=pipeline,volumeClaimTemplateFile=templates/workspace-template.yml \
  --workspace name=kubeconfig,secret=kubeconfig \
  --workspace name=ssh-dir,secret=github-ssh-credentials \
  --workspace name=registry-credentials,secret=registry-dockerconfig-secret \
  --workspace name=pyxis-api-key,secret=pyxis-api-secret \
  --showlog \
  --pod-template templates/crc-pod-template.yml
```

## <a id="submit-result-registy-and-pinning"></a>Submit results with Image Digest Pinning and a private container registry
* Execute the [Configuration Steps for Submitting Results](#submit-results)
* Execute the [Configuration Steps for Digest Pinning](#digest-pinning-config)
* Execute the [Configuration Steps for Private Registries](#private-registry)

```bash
GIT_REPO_URL=<enter the URL to the git repo containing the Operaotr bundle>
BUNDLE_PATH=<path to the bundle in the Git Repo>
GIT_USERNAME=<your github username>
GIT_EMAIL=<your github email address>
REGISTRY=<your image registry.  ie: quay.io>
IMAGE_NAMESPACE=<namespace in the container registry>
UPSTREAM_REPO_NAME=<upstream repo where submission Pull Request is opened>
```

```bash
tkn pipeline start operator-ci-pipeline-serial \
  --param git_repo_url=$GIT_REPO_URL \
  --param git_branch=main \
  --param bundle_path=$BUNDLE_PATH \
  --param env=production \
  --param pin_digests=true \
  --param git_username=$GIT_USERNAME \
  --param git_email=$GIT_EMAIL \
  --param upstream_repo_name=$UPSTREAM_REPO_NAME \
  --param registry=$REGISTRY \
  --param image_namespace=$IMAGE_NAMESPACE \
  --param submit=true \
  --workspace name=pipeline,volumeClaimTemplateFile=templates/workspace-template.yml \
  --workspace name=kubeconfig,secret=kubeconfig \
  --workspace name=ssh-dir,secret=github-ssh-credentials \
  --workspace name=registry-credentials,secret=registry-dockerconfig-secret \
  --workspace name=pyxis-api-key,secret=pyxis-api-secret \
  --showlog \
  --pod-template templates/crc-pod-template.yml
```
