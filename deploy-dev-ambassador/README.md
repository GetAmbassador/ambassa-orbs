## deploy-dev-ambassador

The deploy-dev-ambassador circleci orb is a reusable flow for deploying containerized applications to the development environment for the ambassador platform. It has several steps built into a job that can be used in the circleci config.yml of any repo that falls within the ambassador platform.

### Required Environment Variables

In order to use the `deploy` job from the `deploy-dev-ambassador` orb, the following env vars are required:

`GCP_KEY_FILE` - A base64 encoded string from a Google Cloud Platform service account key. This service account key is generally created in the IAM preferences of GCP and is downloaded as a `.json` file. The contents of this file is what is needed for this env var.

`GOOGLE_PROJECT_ID` - The id of the project where the kubernetes cluster we'll be interacting with lives.

`GOOGLE_COMPUTE_ZONE` - The region of where the kubernetes cluster we'll be interacting with lives i.e. `us-central1-a`.

`CONFIG_REPO` - The Github ssh url to use to pull the kubernetes config files for the source application. If these files reside within a private repo, you'll need to grant access to circleci using a Github machine user as described in the [circleci/Github docs](https://circleci.com/docs/2.0/gh-bb-integration/#creating-a-machine-user)

### Environment Variables Provided

`CIRCLE_BRANCH` - The name of the branch on which the code as been committed. Used in conjunction with the shortened sha of the git commit to create the `CONTAINER_TAG` value.

`CONTAINER_TAG` - The shortened sha of the git commit appended with a dash to the `CIRCLE_BRANCH` env var. The intention of this env var is that it will be merged into the kubernetes config files pulled from the `CONFIG_REPO`. This particular logic assumes that a container image using this branch and commit sha was built and pushed to the Google Container Registry bucket for this application, with the same tag, in a previous job in the workflow **or** a manual build/push.

Example of how this would be used in the kubernetes yaml config repo:

```yaml
# some container / deployment spec
...
image: gcr.io/some-bucket/some-app:{CONTAINER_TAG}
...
```

The container tag would then be merged into the output config file in step 8.


### Flow
The deploy-dev-ambassador orb has a very specific flow that was designed with the principles of GitOps in mind in that, a dev should not know or care how an app is deployed. The only thing a dev should worry about is committing their code via git and a version of that code will be deployed in an observable and testable environment for them. This is implemented with the following:

**Job name:** `deploy`

**Steps:**

**Note:** This job uses the circleci `machine` as an executor. This generally is a slower approach because a VM needs to be spun up. However, one of the steps requires mounting a volume into a docker container, and as such, requires the use of the VM. Using `setup_remote_docker` does not work for this use case.

**1) gcp-cli/install**

This step uses an official orb from circleci to install the Google Cloud Platform CLI.

**2) kubernetes/install**

This step uses an official orb from circleci to install kubernetes aka the kubectl CLI.

**3) Configure gcp-cli and kubectl**

This step sets up the auth for the Google Cloud Platform CLI which then allows us to configure the kubectl context to point at the `ambassador` cluster within the `development-environment` project of our Google Cloud Platform account.

**4) checkout**

This step will checkout the source code of the source repo for the project where the orb's job is implemented.

**5) Generate container tag**

This step uses git to get the shortened sha of the git commit and append it with a dash to the `CIRCLE_BRANCH` env var. It's then set as the `CONTAINER_TAG` env variable in the build environment.

**6) Export env**

This step copies all env vars within the current build env to a file that will be passed to the docker container that merges any tags into the kubernetes config yaml files. This means that any env vars that are available in the current build, will be available to merge into the kubernetes config yaml files.

**7) Pull kubernetes config**

This step pulls in the config files to be applied to the the kubernetes cluster, using the `CONFIG_REPO` env var.

**8) Merge tags and apply config**

This step has two parts. The first part uses a dockerized python application along with the exported env variables to merge any env vars into the previously pulled kubernetes config files. The output of this is piped to the `kubectl apply command` which applies the config to the `ambassador` cluster within the `development-environment` of our Google Cloud Platform account.
