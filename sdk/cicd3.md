
## VM as Runner:

To run these commands on your own Ubuntu VM, you can configure the VM as a **self-hosted GitLab Runner**. This will let your GitLab pipeline jobs run directly on your VM, so any commands (like `kubectl`) will execute in that environment.

### Steps to Set Up Your Ubuntu VM as a GitLab Runner

1. **Install GitLab Runner on Your VM**:
   - SSH into your Ubuntu VM.
   - Follow the official [GitLab Runner installation guide for Ubuntu](https://docs.gitlab.com/runner/install/linux-repository.html):
     ```bash
     curl -L --output /usr/local/bin/gitlab-runner https://gitlab-runner-downloads.s3.amazonaws.com/latest/binaries/gitlab-runner-linux-amd64
     sudo chmod +x /usr/local/bin/gitlab-runner
     ```

2. **Register the Runner with Your GitLab Project**:
   - After installation, register your GitLab Runner to link it to your GitLab project. Use the following command on your VM:
     ```bash
     sudo gitlab-runner register
     ```
   - During registration, you’ll need:
     - The **GitLab instance URL** (e.g., `https://gitlab.com`).
     - A **registration token**, found in **Settings > CI / CD > Runners** in your GitLab project.
     - **Executor type**: Choose `shell` if you want jobs to run directly on the VM without containerization.
   - This setup allows jobs to use the environment of your VM directly.

3. **Update Your .gitlab-ci.yml File**:
   - Now that your runner is registered, you can use the same `.gitlab-ci.yml` file as before. It will run directly on your VM and use any tools installed there, such as `kubectl` or `gcloud`.
   - You don’t need to install `kubectl` in every pipeline job if it’s already installed on your VM; you can remove the installation lines from the YAML file.
  
---

Since you’ll be using your own Ubuntu VM as the GitLab Runner, the pipeline can be simplified. We can skip installing `kubectl` and `gcloud` in each job since they are already set up on your VM.

Here’s a streamlined `.gitlab-ci.yml` file:

```yaml
stages:
  - deploy

before_script:
  # Authenticate with GCP
  - echo "$GCP_SERVICE_ACCOUNT_KEY" > /tmp/gcp-key.json
  - gcloud auth activate-service-account --key-file=/tmp/gcp-key.json
  - gcloud config set project <YOUR_PROJECT_ID>
  - gcloud config set compute/zone <YOUR_COMPUTE_ZONE>

deploy:
  stage: deploy
  script:
    # Configure kubectl to access the cluster
    - gcloud container clusters get-credentials <YOUR_CLUSTER_NAME> --zone <YOUR_COMPUTE_ZONE> --project <YOUR_PROJECT_ID>
    # Run any kubectl commands here
    - kubectl get pods
```

### Explanation

1. **before_script**:
   - Sets up Google Cloud authentication using the service account key stored as a GitLab CI/CD variable.
   - Configures the GCP project and zone for `gcloud` commands.

2. **deploy Job**:
   - Fetches the credentials for your Kubernetes cluster using `gcloud container clusters get-credentials`, enabling `kubectl` to communicate with the cluster.
   - Runs `kubectl get pods` to list the pods in your cluster. You can replace this with any other `kubectl` commands as needed.

### GitLab CI/CD Variables

- In your GitLab project’s settings, go to **Settings > CI / CD > Variables**.
- Add `GCP_SERVICE_ACCOUNT_KEY` as a variable, with the content of your Google Cloud service account JSON key.

### Notes

- Make sure to replace `<YOUR_PROJECT_ID>`, `<YOUR_COMPUTE_ZONE>`, and `<YOUR_CLUSTER_NAME>` with your actual Google Cloud project and Kubernetes cluster details.
- Since this runner is set up on your VM, each command will run in the VM environment directly, allowing seamless access to `gcloud` and `kubectl`.

4. **Run the Pipeline**:
   - Commit and push your `.gitlab-ci.yml` file to trigger the pipeline.
   - The registered runner (your VM) will pick up the job and execute the commands directly on the VM.

With this setup, `kubectl` and `gcloud` will run on your Ubuntu VM whenever a job is triggered, allowing you to interact with your Kubernetes cluster as you would locally.
