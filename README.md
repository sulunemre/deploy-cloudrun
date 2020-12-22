<!--
Copyright 2020 Google LLC

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
-->
# deploy-cloudrun

This action deploys your container image to [Cloud Run][cloud-run] and makes the URL
available to later build steps via outputs.

## Prerequisites

This action requires:

- Google Cloud credentials that are authorized to deploy a
Cloud Run service. See the Authorization section below for more information.

- [Enable the Cloud Run API](http://console.cloud.google.com/apis/library/run.googleapis.com?_ga=2.267842766.1374248275.1591025444-475066991.1589991158)

## Usage

```yaml
steps:
- id: deploy
  uses: google-github-actions/deploy-cloudrun@main
  with:
    image: gcr.io/cloudrun/hello
    service: hello-cloud-run
    credentials: ${{ secrets.gcp_credentials }}

# Example of using the output
- id: test
  run: curl "${{ steps.deploy.outputs.url }}"
```

## Inputs

- `image`: Name of the container image to deploy (e.g. gcr.io/cloudrun/hello:latest).
  Required if not using a service YAML.

- `service`: ID of the service or fully qualified identifier for the service.
  Required if not using a service YAML.

- `region`: Region in which the resource can be found.

- `credentials`: Service account key to use for authentication. This should be
  the JSON formatted private key which can be exported from the Cloud Console. The
  value can be raw or base64-encoded. Required if not using a the
  `setup-gcloud` action with exported credentials.

- `env_vars`: List of key-value pairs to set as environment variables in the format:
  KEY1=VALUE1,KEY2=VALUE2. **All existing environment variables will be retained**.

- `metadata`: YAML service description for the Cloud Run service. See
  [Metadata customizations](#metadata-customizations) for more information.
  **Existing configuration will be retained besides container entrypoint and arguments**.

- `project_id`: (Optional) ID of the Google Cloud project. If provided, this
  will override the project configured by gcloud.

### Metadata customizations

You can store your service specification in a YAML file. This will allow for
further service configuration, such as [memory limits](https://cloud.google.com/run/docs/configuring/memory-limits),
[CPU allocation](https://cloud.google.com/run/docs/configuring/cpu),
[max instances](https://cloud.google.com/run/docs/configuring/max-instances),
and [more.](https://cloud.google.com/sdk/gcloud/reference/run/deploy#OPTIONAL-FLAGS)

- See [Deploying a new service](https://cloud.google.com/run/docs/deploying#yaml)
to create a new YAML service definition, for example:

```YAML
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: SERVICE
spec:
  template:
    spec:
      containers:
      - image: IMAGE
```

- See [Deploy a new revision of an existing service](https://cloud.google.com/run/docs/deploying#yaml_1)
to generated a YAML service specification from an existing service:

```
gcloud run services describe SERVICE --format yaml > service.yaml
```
## Allow unauthenticated requests

A Cloud Run product recommendation is that CI/CD systems not set or change
settings for allowing unauthenticated invocations. New deployments are
automatically private services, while deploying a revision of a public
(unauthenticated) service will preserve the IAM setting of public
(unauthenticated). For more information, see [Controlling access on an individual service](https://cloud.google.com/run/docs/securing/managing-access).

## Outputs

- `url`: The URL of your Cloud Run service.

## Authorization

There are a few ways to authenticate this action. A service account will be needed
with the following roles:

- Cloud Run Admin (`roles/run.admin`):
  - Can create, update, and delete services.
  - Can get and set IAM policies.

This service account needs to a member of the `Compute Engine default service account`,
`(PROJECT_NUMBER-compute@developer.gserviceaccount.com)`, with role
`Service Account User`. To grant a user permissions for a service account, use
one of the methods found in [Configuring Ownership and access to a service account](https://cloud.google.com/iam/docs/granting-roles-to-service-accounts#granting_access_to_a_user_for_a_service_account).

### Used with `setup-gcloud`

You can provide credentials using the [setup-gcloud][setup-gcloud] action:

```yaml
- uses: google-github-actions/setup-gcloud@master
  with:
    version: '290.0.1'
    service_account_key: ${{ secrets.GCP_SA_KEY }}
    export_default_credentials: true
- id: Deploy
  uses: google-github-actions/deploy-cloudrun@main
  with:
    image: gcr.io/cloudrun/hello
    service: hello-cloud-run
```

### Via Credentials

You can provide [Google Cloud Service Account JSON][sa] directly to the action
by specifying the `credentials` input. First, create a [GitHub
Secret][gh-secret] that contains the JSON content, then import it into the
action:

```yaml
- id: Deploy
  uses: google-github-actions/deploy-cloudrun@main
  with:
    credentials: ${{ secrets.GCP_SA_KEY }}
    image: gcr.io/cloudrun/hello
    service: hello-cloud-run
```

### Via Application Default Credentials

If you are hosting your own runners, **and** those runners are on Google Cloud,
you can leverage the Application Default Credentials of the instance. This will
authenticate requests as the service account attached to the instance. **This
only works using a custom runner hosted on GCP.**

```yaml
- id: Deploy
  uses: google-github-actions/deploy-cloudrun@main
  with:
    image: gcr.io/cloudrun/hello
    service: hello-cloud-run
```

The action will automatically detect and use the Application Default
Credentials.

## Example Workflows

* [Deploy a prebuilt container](#deploy-a-prebuilt-container)

* [Build and deploy a container](#build-and-deploy-a-container)

### Setup

1.  Create a new Google Cloud Project (or select an existing project).

1. [Enable the Cloud Run API](https://console.cloud.google.com/flows/enableapi?apiid=run.googleapis.com).

1.  [Create a Google Cloud service account][sa] or select an existing one.

1.  Add the the following [Cloud IAM roles][roles] to your service account:

    - `Cloud Run Admin` - allows for the creation of new Cloud Run services

    - `Service Account User` -  required to deploy to Cloud Run as service account

    - `Storage Admin` - allow push to Google Container Registry (this grants project level access, but recommend reducing this scope to [bucket level permissions](https://cloud.google.com/container-registry/docs/access-control#grant).)

1.  [Download a JSON service account key][create-key] for the service account.

1.  Add the following [secrets to your repository's secrets][gh-secret]:

    - `GCP_PROJECT`: Google Cloud project ID

    - `GCP_SA_KEY`: the downloaded service account key

### Deploy a prebuilt container

To run this [workflow](.github/workflows/example-workflow-quickstart.yaml), push to the branch named `example-deploy`:

```sh
git push YOUR-FORK main:example-deploy
```

### Build and deploy a container

To run this [workflow](.github/workflows/example-workflow.yaml), push to the branch named `example-build-deploy`:

```sh
git push YOUR-FORK main:example-build-deploy
```

**Reminder: If this is your first deployment of a service, it will reject all unauthenticated requests. Learn more at [allowing unauthenticated requests](#Allow-unauthenticated-requests)**

## Migrating from `setup-gcloud`

Example using `setup-gcloud`:

```YAML
- name: Setup Cloud SDK
  uses: google-github-actions/setup-gcloud@v0.2.0
  with:
    project_id: ${{ env.PROJECT_ID }}
    service_account_key: ${{ secrets.GCP_SA_KEY }}

- name: Deploy to Cloud Run
  run: |-
    gcloud run deploy $SERVICE \
      --region $REGION \
      --image gcr.io/$PROJECT_ID/$SERVICE \
      --platform managed \
      --set-env-vars NAME="Hello World"
```

Migrated to `deploy-cloudrun`:

```YAML
- name: Deploy to Cloud Run
  uses: google-github-actions/deploy-cloudrun@v0.2.0
  with:
    service: ${{ env.SERVICE }}
    image: gcr.io/${{ env.PROJECT_ID }}/${{ env.SERVICE }}
    region: ${{ env.REGION }}
    credentials: ${{ secrets.GCP_SA_KEY }}
    env_vars: NAME="Hello World"
```
Note: The action is for the "managed" platform and will not set access privileges such as [allowing unauthenticated requests](#Allow-unauthenticated-requests).


[cloud-run]: https://cloud.google.com/run
[sa]: https://cloud.google.com/iam/docs/creating-managing-service-accounts
[create-key]: https://cloud.google.com/iam/docs/creating-managing-service-account-keys
[gh-runners]: https://help.github.com/en/actions/hosting-your-own-runners/about-self-hosted-runners
[gh-secret]: https://help.github.com/en/actions/configuring-and-managing-workflows/creating-and-storing-encrypted-secrets
[setup-gcloud]: ./setup-gcloud
