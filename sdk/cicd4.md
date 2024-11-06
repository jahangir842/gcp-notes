To address the `gcloud: command not found` error in your pipeline, you can make some adjustments. It seems like the installation of `gcloud` via `curl` is not completing before the command is invoked, as it relies on the `$SHELL` environment to reload. Here are two solutions to help streamline this setup:

### Solution 1: Use `google/cloud-sdk` Image

To avoid installing `gcloud` manually, you can base your pipeline on an image that already includes the Google Cloud SDK, such as `google/cloud-sdk`. Here’s how to modify your pipeline:

```yaml
image: google/cloud-sdk:latest

stages:
  - setup
  - deploy

variables:
  GOOGLE_APPLICATION_CREDENTIALS: /tmp/gcp-key.json

before_script:
  - echo "$GCP_SERVICE_ACCOUNT_KEY" > $GOOGLE_APPLICATION_CREDENTIALS
  - gcloud auth activate-service-account --key-file=$GOOGLE_APPLICATION_CREDENTIALS
  - gcloud config set project <YOUR_PROJECT_ID>
  - gcloud config set compute/zone <YOUR_COMPUTE_ZONE>

setup_kubectl:
  stage: setup
  script:
    # Install kubectl if it's not available
    - if ! command -v kubectl &> /dev/null; then
        curl -LO "https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl";
        chmod +x ./kubectl;
        mv ./kubectl /usr/local/bin/kubectl;
      fi
    # Authenticate and get cluster credentials
    - gcloud container clusters get-credentials <YOUR_CLUSTER_NAME> --zone <YOUR_COMPUTE_ZONE> --project <YOUR_PROJECT_ID>

deploy:
  stage: deploy
  script:
    - kubectl get pods
```

### Solution 2: Adjust the Installation Steps for `gcloud` and `kubectl`

If you want to continue using your current image, you can adjust the installation steps to ensure the shell reloads after installing `gcloud`. This avoids needing to restart the shell mid-script.

```yaml
before_script:
  - echo "$GCP_SERVICE_ACCOUNT_KEY" > $GOOGLE_APPLICATION_CREDENTIALS
  - |
    if ! command -v gcloud &> /dev/null; then
      curl -sSL https://sdk.cloud.google.com | bash -s -- --install-dir=/usr/local/google-cloud-sdk --disable-prompts;
      export PATH="/usr/local/google-cloud-sdk/bin:$PATH";
      source /usr/local/google-cloud-sdk/path.bash.inc;
      gcloud components install kubectl -q;
    fi
  - gcloud auth activate-service-account --key-file=$GOOGLE_APPLICATION_CREDENTIALS
  - gcloud config set project <YOUR_PROJECT_ID>
  - gcloud config set compute/zone <YOUR_COMPUTE_ZONE>
```

These adjustments should ensure `gcloud` and `kubectl` are available for the pipeline stages that follow.
