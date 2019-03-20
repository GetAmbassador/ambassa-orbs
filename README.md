## ambassa-orbs

This repo houses the source code for all internally developed [circleci orbs](https://circleci.com/orbs/) used within our CI/CD pipeline. Docs for each can be found in the orbs respective directories.

**Contents**

- [container-registry](https://github.com/GetAmbassador/ambassa-orbs/tree/master/container-registry)
- [deploy-dev-ambassador](https://github.com/GetAmbassador/ambassa-orbs/tree/master/deploy-dev-ambassador)

### Env Vars

The orbs housed within this repo both use an env var named `GCP_KEY_FILE`. This particular var is generated from a Google Cloud Platform service account 's crednetial. Within GCP, service accounts are a way for
application managers to implement fine grained access control for "non-human" users who need to have access to the cluster (or other GCP products). As of this writing, there is only a single service account
within the development environment project and it has access to manipulate the state of the `ambassador` cluster. The email address associated with it is `circleci-kube-operator@staging-environment-232215.iam.gserviceaccount.com`.
It can be found in the console navigator `IAM & admin` tab under `Service Accounts`.
