# Coinbase

## APP:

Using python with flask framwork we create a webservice that returns a json object containing spot price data from Coinbase (https://api.coinbase.com/v2/prices/spot?currency=USD). The endpoint support: EUR, GBP, USD and JPY. and also have `/health` endpoint that returns 200 if the application is running.

## Folder structure:

```sh
_ /
 |_  src/ # the application code
 |_ terraform # Terraform files
 |_ helm-chart # Helm chart for the application
 |_ stop-app.sh # Script to stop and remove the app
 |_ Dockerfile # Docker file for the application
 |_ requirement.txt # requirement libraries for the app
 |_ README.md # documentation file
```

## Requirement:

- Python
- Docker
- helm
- terraform
- aws
- ek

## Github secret to define:

We used github secrets to store the sensitive data and credentials, here is the list of required secret for the CI/CD pipeline:

![github secrect](./img/github-secrets.png)

- ACTION_MONITORING_SLACK: Slack webhook url for the channel
- AWS_ACCESS_KEY_ID and AWS_SECRET_ACCESS_KEY: are the IAM account credentials (ID and secret access key)

## Run test

To ensure the best code intergreation the CI/CD pipeline runs a test that connects to the mock service and it fails if the json object cannot be parsed, or if the currency does not exist.

```yml
test:
  runs-on: ubuntu-latest
  steps:
    - uses: actions/checkout@v3
    - name: Set up Python 3.9
      uses: actions/setup-python@v3
      with:
        python-version: 3.9
    - name: Install dependencies
      run: |
        python -m pip install --upgrade pip
        pip install pytest
        pip install -r requirements.txt
    - name: Test with pytest
      run: |
        pytest
```

## AWS ECR:

If the test is OK, the next step in our CI/CD is to build and push our application image to ECR.
In this step we need to provide IAM credentails to access the ECR account.

```yml
build:
  runs-on: ubuntu-latest
  needs: [test]
  steps:
    - uses: actions/checkout@v3
    - name: Build the Docker image 📦
      run: docker build --tag python-docker .
    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v1
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: us-east-2
    - name: Login to Amazon ECR
      id: login-ecr
      uses: aws-actions/amazon-ecr-login@v1
    - name: Build, tag, and push image to Amazon ECR 📦
      env:
        ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
        ECR_REPOSITORY: python-docker
        IMAGE_TAG: latest
      run: |
        docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG .
        docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG
```

## Slack Notification:

We add intergration between github action and slack in order to push a message about the build status to a specific channel.  
All we need is to create slack Web hook url and add the following github action to our pipeline:

```yml
- name: Report Status
  if: always()
  uses: ravsamhq/notify-slack-action@v1
  with:
    status: ${{ job.status }}
  env:
    SLACK_WEBHOOK_URL: ${{ secrets.ACTION_MONITORING_SLACK }}
```

## HELM and terraform:

The last step is to apply the terraform playbook to deploy our application to the created kubernetes cluster in EKS, terraform will use the helm chart to deploy the app. refere to [deploy.tf file](./terraform/deploy.tf).

```yml
terraform:
  runs-on: ubuntu-latest
  needs: [build]
  steps:
    - name: Check out code
      uses: actions/checkout@v2
    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v1
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: us-east-2
    - name: Install and configure kubectl
      run: |
        VERSION=$(curl --silent https://storage.googleapis.com/kubernetes-release/release/stable.txt)
        curl https://storage.googleapis.com/kubernetes-release/release/$VERSION/bin/linux/amd64/kubectl \
            --progress-bar \
            --location \
            --remote-name
        chmod +x kubectl
        sudo mv kubectl /usr/local/bin/
        aws eks --region us-east-2 update-kubeconfig --name k8s-cluster-2
    - uses: hashicorp/setup-terraform@v2
      with:
        terraform_version: 1.1.2
    - name: Terraform init
      run: terraform -chdir="terraform" init

    - name: Terraform apply
      run: terraform -chdir="terraform" apply -auto-approve
```
