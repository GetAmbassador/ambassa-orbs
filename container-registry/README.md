## container-registry
The container registry circleci orb is a reusable flow for building and pushing images to a private Google Container Registry bucket. It has several steps built into a job that can be used in the circleci config.yml of any of our source code repos that require containerization.

#### Required Environment Variables

In order to use the `build_and_push` job from the `container-registry` orb, the following env vars are required:

`GCP_KEY_FILE` - A base64 encoded string from a Google Cloud Platform service account key. This service account key is generally created in the IAM preferences of GCP and is downloaded as a `.json` file. The content of this file can be copied into the env var.

`GOOGLE_PROJECT_ID` - The id of the project within GCP where the container registry we're pushing to lives.

`GOOGLE_COMPUTE_ZONE` - The region where the container registry bucket lives i.e. `us-central1-a`.

#### Internal Environment Variables

This orb uses two internal environment variables to set values according to convention, where necessary.

`CIRCLE_PROJECT_REPONAME` - The name of the repo for which code has been committed to. Used to name the image when being built/pushed.

`CIRCLE_BRANCH` - The name of the branch on which the code as been committed. Used in conjunction with the shortened sha of the git commit to create the `CONTAINER_TAG` value.

`CONTAINER_TAG` - The shortened sha of the git commit appended with a dash to the `CIRCLE_BRANCH`. This env var is then used to tag the image while building **and** pushing to the container registry.


#### Flow

The container-registry orb was implemented with the principlesof GitOps in mind. As such, a dev does not need to know when or how their code is built / pushed to a container registry. Instead, that portion of work is taken out of their hands by the CI/CD process. The only thing a dev should worry about is committing their code via git. At some point, an image of that committed version of their code will be built / pushed to the container registry. This is implemented with the following:

**Job name:** `build_and_push`

**Steps:**

**1) setup_remote_docker**

This step is specific to circleci. It enables a docker daemon for use when we go to actually build and image. More documentation on [circleci remote docker](https://circleci.com/docs/2.0/building-docker-images/#overview) and [circleci docker_layer_caching](https://circleci.com/docs/2.0/docker-layer-caching/).

**2) checkout**

This step will checkout the source code of the source repo for the project where the orb's job is implemented.

**3) Generate container tag**

This step uses git to get the shortened sha of the git commit and set it as the `CONTAINER_TAG` env variable in the build environment.

**4) gcp-gcr/gcr-auth**

This step will install the GCP CLI and attempt to authorize it using the official circleci `gcp-gcr` orb. The auth part of this step doesn't really matter because the command in the official orb is actually broken. Instead, the GCP CLI is being installed and the project/region are being set.

**5) gcp-gcr/build-image**

This step will again use the official circleci `gcp-gcr` orb to build an image of the committed code via docker. This particular step takes two parameters: `image` and `tag`. In this case, we use the  `CIRCLE_PROJECT_REPONAME` env var as the image name and the `CONTAINER_TAG` env var as the tag. The build image name will end up being formatted as follows:

`gcr.io/<GOOGLE_PROJECT_ID>/<CIRCLE_PROJECT_REPONAME>:<CONTAINER_TAG`


**6) push-image**

This step is pretty self explanatory by name, but has some caveats. As mentioned above, the auth piece of `gcp-gcr` orb is broken as of this writing. Instead, the `container-registry` orb implements a command that takes care of both authorizing the GCP CLI *and* and pushing the image that was built in the previous step. This step also takes two parameters `image` and `tag`. Again, we use the `CIRCLE_PROJECT_REPONAME` env var as the image name and the `CONTAINER_TAG` env var as the tag. This will push the image to the registry using the same format depicted in the previous step. This format also represents the structure of the registry once it is pushed. For example, assuming the following env var values:

```bash
GOOGLE_PROJECT_ID=myproject
CIRCLE_PROJECT_REPONAME=ambassa-add
CIRCLE_BRANCH=master
CONTAINER_TAG=1234
```

Within the Google Container Registry tab of the GCP project console for `myproject`, there will be a directory named `ambassa-add` and an image taged with `master-1234` within in it.
