
# Dockerized Hello World App with GitHub Actions and AWS ECR Integration

This repository contains a simple Hello World application, Dockerized for easy deployment, and configured to automatically build and push Docker images to AWS Elastic Container Registry (ECR) using GitHub Actions. The workflow triggers on code pushes and pull requests, ensuring a seamless integration and deployment process.




## Pre-requisite to replicate the setup

1. AWS account to create ECR, access keys for pushing and pulling images
2. Docker installed on local


## Docker Image Creation
The Dockerfile in the repository is carefully crafted to build an optimized Docker image for the Hello World application. It prioritizes both size and build time efficiency.

Write Dockerfile that transfers your code to the image filesystem, installs required dependencies, and runs a Flask application. Please note that you should customize it according to the structure of your Flask application and any additional dependencies you may have.


``` 
FROM python:3.8-slim

WORKDIR /app
COPY /src .

RUN pip install -r requirements.txt

EXPOSE 5000

CMD [ "flask","run","--host","0.0.0.0"] 
```
Save this as name Dockerfile in your root directory of your project

The ```FROM python:3.8-slim``` line in a Dockerfile is an instruction that specifies the base image for your Docker image.


The `WORKDIR /app` is used to set the working directory for any subsequent RUN, CMD, ENTRYPOINT, COPY, and ADD instructions.

The ` COPY /src . `is used to copy files or directories from the host machine (outside the Docker image) to the image itself.

`RUN pip install -r requirements.txt` This line installs the Python dependencies specified in the requirements.txt file using the pip package manager. It ensures that the required Python packages are installed inside the image

You can find this requirements.txt file in /src folder.

`EXPOSE 5000` informs Docker that the container will listen on the specified network ports at runtime

`CMD [ "flask", "run", "--host", "0.0.0.0" ]` This runs the Flask development server, making the application accessible externally by binding it to 0.0.0.0.
## AWS ECR Setup
Open the AWS Console and search ECR.
Now click on the Create repository button to Create ECR repository

on visibility setting click on the private option, so that Access is managed by IAM and repository policy permissions only.


Provide a concise Repository name.

Enable tag immutability to prevent image tags from being overwritten by subsequent image pushes using the same tag

At last click on the button named Create repository to finish the process.

Now you have a ECR.

The Repository URI should look like this `Account_name.dkr.ecr.AWS_Reagion.amazonaws.com/Repository_Name`
## GitHub Actions Workflow

```
name: Build and Push to ECR

on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main

env:
  AWS_REGION: ap-south-1
  ECR_REPO: imagesync
  DOCKER_IMAGE_NAME: initial_image
  COMMIT_HASH: ${{ github.sha }}
jobs:
  build-and-push:
    runs-on: ubuntu-latest

    steps:
    - name: Checkout Repository
      uses: actions/checkout@v2

    - name: Configure AWS Credentials
      uses: aws-actions/configure-aws-credentials@v2
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: ${{ env.AWS_REGION }}

    - name: Login to Amazon ECR
      id: login-ecr
      run: |
        aws ecr get-login-password --region ap-south-1 | docker login --username AWS --password-stdin ${{ vars.AWS_ACCOUNT_ID }}.dkr.ecr.${{ env.AWS_REGION }}.amazonaws.com

    - name: Build and Push Docker Image
      run: |
        docker build -t ${{ env.DOCKER_IMAGE_NAME }} .
        docker tag ${{ env.DOCKER_IMAGE_NAME }} ${{ vars.AWS_ACCOUNT_ID }}.dkr.ecr.${{ env.AWS_REGION }}.amazonaws.com/${{ env.ECR_REPO }}:${{ env.COMMIT_HASH }}
        docker push ${{ vars.AWS_ACCOUNT_ID }}.dkr.ecr.${{ env.AWS_REGION }}.amazonaws.com/${{ env.ECR_REPO }}:${{ env.COMMIT_HASH }}

    - name: Run Trivy vulnerability scanner
      uses: aquasecurity/trivy-action@master
      with:
        image-ref: '${{ vars.AWS_ACCOUNT_ID }}.dkr.ecr.${{ env.AWS_REGION }}.amazonaws.com/${{ env.ECR_REPO }}:${{ env.COMMIT_HASH }}'
        format: 'template'
        template: '@/contrib/sarif.tpl'
        output: 'trivy-results.sarif'
        severity: 'CRITICAL,HIGH'
      env:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: ${{ env.AWS_REGION }}

    - name: Upload Trivy scan results to GitHub Security tab
      uses: github/codeql-action/upload-sarif@v2
      with:
        sarif_file: 'trivy-results.sarif'

    - name: Notify on Failure
      if: ${{ failure() }}
      run: |
          curl -X POST -H 'Content-type: application/json' --data '{
            "text": ":x: GitHub Actions Workflow Failed - ${{ github.repository }}, Commit Hash: ${{ env.COMMIT_HASH }}"
          }' ${{ secrets.SLACK_WEBHOOK_URL }}

    - name: Notify on Success
      if: ${{ success() }}
      run: |
          curl -X POST -H 'Content-type: application/json' --data '{
            "text": ":white_check_mark: GitHub Actions Workflow Succeeded - ${{ github.repository }}, Commit Hash: ${{ env.COMMIT_HASH }}"
          }' ${{ secrets.SLACK_WEBHOOK_URL }}
```


### Workflow Trigger Configuration

```
on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main
```
This configuration specifies when the workflow should be triggered. In this case, it triggers on both code pushes to the main branch and pull requests targeting the main branch.

### Environment Variables
```
env:
  AWS_REGION: ap-south-1
  ECR_REPO: imagesync
  DOCKER_IMAGE_NAME: initial_image
  COMMIT_HASH: ${{ github.sha }}

```

Defines environment variables that the workflow uses throughout its execution. It includes the AWS region, the name of the ECR repository, the initial Docker image name, and the commit hash.

### Job Configuration

```
jobs:
  build-and-push:
    runs-on: ubuntu-latest

```

### Job Steps
#### Step 1: Checkout Repository

```
- name: Checkout Repository
  uses: actions/checkout@v2
```
Uses the GitHub Actions checkout action to clone the repository into the runner's workspace.

#### Step 2: Configure AWS Credentials
```
- name: Configure AWS Credentials
  uses: aws-actions/configure-aws-credentials@v2
  with:
    aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
    aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
    aws-region: ${{ env.AWS_REGION }}
```

Configures AWS credentials using the aws-actions/configure-aws-credentials action. It pulls the AWS access key and secret key from GitHub secrets.

#### Step 3: Login to Amazon ECR

```
- name: Login to Amazon ECR
  id: login-ecr
  run: |
    aws ecr get-login-password --region ap-south-1 | docker login --username AWS --password-stdin ${{ vars.AWS_ACCOUNT_ID }}.dkr.ecr.${{ env.AWS_REGION }}.amazonaws.com
```
Logs in to Amazon ECR to authenticate Docker and enable pushing Docker images.

#### Step 4: Build and Push Docker Image
```
- name: Build and Push Docker Image
  run: |
    docker build -t ${{ env.DOCKER_IMAGE_NAME }} .
    docker tag ${{ env.DOCKER_IMAGE_NAME }} ${{ vars.AWS_ACCOUNT_ID }}.dkr.ecr.${{ env.AWS_REGION }}.amazonaws.com/${{ env.ECR_REPO }}:${{ env.COMMIT_HASH }}
    docker push ${{ vars.AWS_ACCOUNT_ID }}.dkr.ecr.${{ env.AWS_REGION }}.amazonaws.com/${{ env.ECR_REPO }}:${{ env.COMMIT_HASH }}

```

Builds the Docker image from the repository code, tags it with the commit hash, and pushes it to the specified ECR repository.

#### Step 5: Run Trivy Vulnerability Scanner
```
- name: Run Trivy vulnerability scanner
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: '${{ vars.AWS_ACCOUNT_ID }}.dkr.ecr.${{ env.AWS_REGION }}.amazonaws.com/${{ env.ECR_REPO }}:${{ env.COMMIT_HASH }}'
    format: 'template'
    template: '@/contrib/sarif.tpl'
    output: 'trivy-results.sarif'
    severity: 'CRITICAL,HIGH'
  env:
    aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
    aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
    aws-region: ${{ env.AWS_REGION }}

```

Uses the Trivy security scanner to identify vulnerabilities in the Docker image, generates results in SARIF format, and stores them in trivy-results.sarif.

#### Step 6: Upload Trivy Scan Results to GitHub Security Tab

```
- name: Upload Trivy scan results to GitHub Security tab
  uses: github/codeql-action/upload-sarif@v2
  with:
    sarif_file: 'trivy-results.sarif'

```

Uploads the Trivy scan results to the GitHub Security tab for better visibility into potential vulnerabilities.

#### Steps 7-8: Notifications on Failure and Success
```
- name: Notify on Failure
  if: ${{ failure() }}
  run: |
    curl -X POST -H 'Content-type: application/json' --data '{
      "text": ":x: GitHub Actions Workflow Failed - ${{ github.repository }}, Commit Hash: ${{ env.COMMIT_HASH }}"
    }' ${{ secrets.SLACK_WEBHOOK_URL }}

- name: Notify on Success
  if: ${{ success() }}
  run: |
    curl -X POST -H 'Content-type: application/json' --data '{
      "text": ":white_check_mark: GitHub Actions Workflow Succeeded - ${{ github.repository }}, Commit Hash: ${{ env.COMMIT_HASH }}"
    }' ${{ secrets.SLACK_WEBHOOK_URL }}
```
Send notifications to a Slack channel using a webhook URL in case of workflow failure or success, providing information about the repository name and commit hash

This comprehensive GitHub Actions workflow automates various tasks, including building and pushing Docker images, performing security scans, and providing notifications, contributing to a streamlined and secure development pipeline.

For the slack incoming notifications, change the Workflow permissions:
open the Repository setting , click on the Action -> General tab and change permission to Read and Write permissions and save it.
## Security and Best Practices

Sensitive credentials are stored as secrets in GitHub, ensuring secure management of access tokens and other sensitive information.


## Run project

Now that you have understand the project,
make a commit to the repository and open the github actions tab,
You will see a workflow run.
![workflow_run](https://github.com/Aman16102000/DockerImageSync/assets/57329099/5d1dec4e-33b5-42a4-848c-f04587e8f257)




Also you can check the ECR Repository where the images are uploaded
![ECR](https://github.com/Aman16102000/DockerImageSync/assets/57329099/0caeb7e8-4246-421e-af60-a9d7321a24aa)


Below are the Slack incoming messages
![slack](https://github.com/Aman16102000/DockerImageSync/assets/57329099/875463a9-be0e-4488-9a76-03d5e44714c2)


## How to access the hello world app from the public internet

Now if you are using AWS, make a EC2 instance with below security group configurations
![Security_group](https://github.com/Aman16102000/DockerImageSync/assets/57329099/ded2d524-c78a-4b47-aeea-075319b9de0c)


Open port 22 with CIDR block "0.0.0.0/0" to get incoming SSH request from IP's, if you want to increase security of this web server add IP address your ISP provided you.

Also open port for HTTP request.

Install docker on you web server and use ` aws configure ` command to add AWS access Keys.

Now pull docker image from your ECR using command
` docker pull Account_id.dkr.ecr.AWS_Region.amazonaws.com/Repo_name:Image_tag `

Please write the URI string of the image you want to pull. You can get this from AWS ECR

Now use this command to run docker container

` docker run -p 80:5000 Account_id.dkr.ecr.AWS_Region.amazonaws.com/Repo_name:Image_tag `

` -p 80:5000 ` this maps the port of host machine "80" to the port of docker container "5000"

That's all you need to do !

Now open the default Public IPv4 DNS of the AWS web server. you can find these details AWS console.

and make sure only use http when opening this Public IPv4 DNS.

![Hello_world_2](https://github.com/Aman16102000/DockerImageSync/assets/57329099/24da3f4a-f33e-41f8-b1ec-cbfdd977fe09)


## How to access the hello world app from the local

Local steps are alos quite similiar to above

buidld the docker image using build command 

` docker build -t imagesync . `
imagesync is the name of the image i want to give this.

` docker run -p 5000:5000 imagesync `

Now open the http://127.0.0.1:5000/ this url access the Hello World App.

Happy Coding !
