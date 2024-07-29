### Setting up a GitHub Actions Pipeline

Assuming you have a GitHub repository with a `main` branch, you can set up a pipeline to run tests on every push to the `main` branch, on pull requests to `main`, or manually. This is done by creating a `.github/workflows` directory in the root of your repository and adding a `.yml` file in it. Follow the steps below to set up your pipeline.

#### Step 1: Create the Workflow Directory

1. **Navigate to Your Project Directory**:
   Open your terminal and navigate to your project's root directory.

2. **Create the Workflow Directory**:
   Create the necessary directory for GitHub workflows:
   ```bash
   mkdir -p .github/workflows
   ```

#### Step 2: Add the Workflow File

1. **Create a Workflow File**:
   Inside the `.github/workflows` directory, create a new file named `workflow.yml`.

2. **Add the Following Configuration to the File**:
   ```yaml
   name: CI
   on: # Trigger the workflow on push to main, pull request to main, or manual trigger
     push:
       branches:
         - main
     pull_request:
       branches:
         - main
     workflow_dispatch:

   jobs:
     reserve-ci-instance: # Reserve an instance
     runs-on: ubuntu-latest
     outputs:
         mpkit-url: ${{ steps.reserve.outputs.mpkit-url }}
         report-path: ${{ steps.reserve.outputs.report-path }}
     steps:
       - name: Get ci-instance-url
         id: reserve
         uses: Platform-OS/ci-repository-reserve-instance-url@0.0.9 # Use our custom action to reserve an instance
         with:
           repository-url: https://ci-repository.staging.oregon.platform-os.com
           method: reserve
           pos-ci-repo-token: ${{ secrets.POS_CI_REPO_ACCESS_TOKEN }}

     deploy:
       needs: ["reserve-ci-instance"] # Don't run this job until the reserve-ci-instance job is done
       runs-on: ubuntu-latest
       env:
         MPKIT_EMAIL: ${{ secrets.MPKIT_EMAIL }}
         MPKIT_TOKEN: ${{ secrets.MPKIT_TOKEN }}
         MPKIT_URL: ${{ needs.reserve-ci-instance.outputs.mpkit-url }}
         POS_CI_REPO_ACCESS_TOKEN: ${{ secrets.POS_CI_REPO_ACCESS_TOKEN }}
         UPLOAD_HOST: ${{ secrets.UPLOAD_HOST }}
         CI: true
       steps:
         - name: Checkout your repository
           uses: actions/checkout@v4

         - name: Build your project
           shell: sh
           run: |
             set -eu
             npm install
             npm run build

         - name: Deploy your project to the reserved instance
           shell: sh
           run: |
             set -eu
             ./seed.sh

     tests:
       needs: ["reserve-ci-instance", "deploy"] # Don't run this job until both the reserve-ci-instance and deploy jobs are done
       runs-on: ubuntu-latest
       env:
         MPKIT_EMAIL: ${{ secrets.MPKIT_EMAIL }}
         MPKIT_TOKEN: ${{ secrets.MPKIT_TOKEN }}
         MPKIT_URL: ${{ needs.reserve-ci-instance.outputs.mpkit-url }}
         REPORT_PATH: ${{ needs.reserve-ci-instance.outputs.report-path }}
         UPLOAD_HOST: ${{ secrets.UPLOAD_HOST }}
         CI: true
       steps:
         - name: Run unit tests
           shell: sh
           run: |
             set -eu
             npm run ci:test:unit

         - name: Clean up after unit tests and prepare environment for E2E tests
           shell: sh
           run: |
             set -e
             ./seed.sh

         - name: Run E2E tests
           if: success() || failure() # Run E2E tests even if unit tests fail, this will allow you to get feedback on the E2E tests
           with:
             test-name: e2e-as-defined-in-package-json # This is the name of the script that runs your E2E tests

     cleanup:
       if: ${{ always() }}
       needs: ["reserve-ci-instance", "deploy", "tests"]
       runs-on: ubuntu-latest
       steps:
         - name: release ci-instance-url
           uses: Platform-OS/ci-repository-reserve-instance-url@0.0.9 # Use our custom action to release the instance
           with:
             method: release
             repository-url: https://ci-repository.staging.oregon.platform-os.com
             pos-ci-repo-token: ${{ secrets.POS_CI_REPO_ACCESS_TOKEN }}

     deploy-qa: # Deploy to QA on push to master and after tests pass
       if: ${{ github.ref_name == 'master' }}
       needs: ["tests"] # Don't run this job until the tests job is done and successful
       runs-on: ubuntu-latest
       env:
         MPKIT_EMAIL: ${{ secrets.MPKIT_EMAIL }}
         MPKIT_TOKEN: ${{ secrets.MPKIT_TOKEN }}
         MPKIT_URL: ${{ QA-HOST-URL }}
         POS_CI_REPO_ACCESS_TOKEN: ${{ secrets.POS_CI_REPO_ACCESS_TOKEN }}
         CI: true
       steps:
         - name: Checkout repository
           uses: actions/checkout@v4

         - name: Build
           shell: sh
           run: |
             set -eu
             npm ci
             npm run build

         - name: Deploy
           shell: sh
           run: |
             set -eu
             pos-cli deploy
   ```

#### Step 3: Setting Up Secrets

To securely store sensitive information, use GitHub Secrets:

1. **Navigate to Your Repository Settings**:
   - Go to your repository on GitHub.
   - Click on **Settings**.

2. **Add Secrets**:
   - Click on **Secrets and variables** in the left sidebar, then **Actions**.
   - Click **New repository secret** and add the following secrets:
     - `POS_CI_REPO_ACCESS_TOKEN`
     - `MPKIT_EMAIL`
     - `MPKIT_TOKEN`
     - `UPLOAD_HOST`
     - `QA_HOST_URL`

#### Step 4: Committing the Workflow File

1. **Add the Workflow File**:
   - Add, commit, and push the `workflow.yml` file to your repository:
     ```bash
     git add .github/workflows/workflow.yml
     git commit -m "Add GitHub Actions CI workflow"
     git push origin main
     ```

#### Step 5: Running the Workflow

1. **Trigger the Workflow**:
   - The workflow will automatically run on pushes to the `main` branch or pull requests to the `main` branch.
   - You can also manually trigger the workflow from the Actions tab in your GitHub repository.

2. **Monitor the Workflow**:
   - Go to the **Actions** tab in your GitHub repository to monitor the progress of your workflow.
   - Check the logs for each job to ensure everything is running smoothly.