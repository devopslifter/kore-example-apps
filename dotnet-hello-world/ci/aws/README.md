## CI with AWS services and Helm

This repository demonstrates the use of AWS services and Helm to achieve a simple continuous integration pipeline.

The CI workflow is triggered when a developer creates or merges an AWS CodeCommit pull request. This ensures that the code changes run through security and quality gates to provide a peer reviewer with the confidence and assurance to merge a pull request.

For demonstration purposes, code changes in a `pull request` or code changes merged into the `master` branch are eventually deployed in the same environment. However this can be customised according to your team's/organisation's branching and deployment strategy to extend to either perform a continuous delivery or continuous deployment to a Production environment.

## Prerequisites

The following scripts must be executed by an **AWS administrator** to ensure that a developer is granted least privileged permissions to complete the setup.

```bash
./scripts/grant-aws-access.sh -u <USER_NAME>
```

Once an **AWS administrator** has successfully executed the above scripts, you must update your local AWS configuration to obtain the required permissions.

Update `~/.aws/credentials` with an IAM user part of the `appvia-workshop-admin` IAM group.
```
[appvia-workshop-user]
aws_access_key_id = <AWS_ACCESS_KEY_ID>
aws_secret_access_key = <AWS_SECRET_ACCESS_KEY>
```

Update `~/.aws/config` to reference the above source profile and the `appvia-workshop-admin-role` IAM role to assume.
```
[profile appvia-workshop-user]
role_arn=arn:aws:iam::<AWS_ACCOUNT_ID>:role/appvia-workshop-admin-role
source_profile=appvia-workshop-user
region=eu-west-2
```

## Getting started

### Create and connect to CodeCommit repository

1. Create a new CodeCommit repository.
```bash
aws codecommit create-repository --repository-name kore-example-apps
```

1. Install the [git-remote-codecommit](https://pypi.org/project/git-remote-codecommit/) utility on your local computer to provide a simple method for pushing and pulling code from CodeCommit repositories. It extends Git by enabling the use of AWS temporary credentials.
```bash
pip install git-remote-codecommit
```

1. Clone the existing `kore-example-apps` GitHub repository and push to the new CodeCommit repository.
```bash
git clone https://github.com/appvia/kore-example-apps.git
cd kore-apps && rm -rf .github .git
git init && git remote add origin codecommit://appvia-workshop-user@kore-example-apps && \
git add . && git commit -m "initial commit" && git push -u origin master
```

#### Create CodeBuild build specifications

1. Create a build specification to instruct CodeBuild how to build, scan for vulnerabilities, package and deploy the .net core web application - see [buildspec.yml](https://github.com/appvia/kore-example-apps/blob/main/dotnet-hello-world/ci/aws/config/buildspec.yml)

1. Create another build specification to instruct Codebuild how to perform static code analysis and push the report to SonarCloud - see [buildspec-sonarcloud.yml](https://github.com/appvia/kore-example-apps/blob/main/dotnet-hello-world/ci/aws/config/buildspec-sonarcloud.yml)

#### Create CodeBuild projects

1. Create an IAM role that enables CodeBuild to interact with dependent AWS services on behalf of the AWS account.  
```bash
aws iam create-role --profile appvia-workshop-user --role-name CodeBuildServiceRole --assume-role-policy-document "{\"Version\":\"2012-10-17\",\"Statement\":[{\"Effect\":\"Allow\",\"Principal\":{\"Service\":\"codebuild.amazonaws.com\"},\"Action\": \"sts:AssumeRole\"}]}"
aws iam put-role-policy --profile appvia-workshop-user --role-name CodeBuildServiceRole --policy-name CodeBuildServiceRolePolicy --policy-document file://config/iam-codebuild-role-policy.json
```

1. Create a build project that references the CodeCommit repository and the [buildspec.yml](https://github.com/appvia/kore-example-apps/blob/main/dotnet-hello-world/ci/aws/config/buildspec.yml).
```bash
AWS_ACCOUNT_ID=$(aws sts get-caller-identity --profile appvia-workshop-user --query "Account" --output text)
CODEBUILD_PROJECT="dotnet-hello-world-build-deploy"
aws codebuild create-project \
 --profile appvia-workshop-user \
 --name ${CODEBUILD_PROJECT} \
 --source "{\"type\": \"CODECOMMIT\",\"location\": \"https://git-codecommit.eu-west-2.amazonaws.com/v1/repos/kore-example-apps\", \"buildspec\": \"dotnet-hello-world/ci/aws/config/buildspec.yml\"}" \
 --environment "{\"type\": \"LINUX_CONTAINER\",\"image\": \"aws/codebuild/amazonlinux2-x86_64-standard:3.0\",\"computeType\": \"BUILD_GENERAL1_SMALL\"}" \
 --service-role "arn:aws:iam::${AWS_ACCOUNT_ID}:role/CodeBuildServiceRole" \
 --artifacts "{\"type\": \"NO_ARTIFACTS\"}" \
 --source-version "refs/heads/master"
```

1. Create a build project that references the CodeCommit repository and the [buildspec-sonarcloud.yml](https://github.com/appvia/kore-example-apps/blob/main/dotnet-hello-world/ci/aws/config/buildspec-sonarcloud.yml).
```bash
AWS_ACCOUNT_ID=$(aws sts get-caller-identity --profile appvia-workshop-user --query "Account" --output text)
CODEBUILD_PROJECT="dotnet-hello-world-code-coverage"
aws codebuild create-project \
 --profile appvia-workshop-user \
 --name ${CODEBUILD_PROJECT} \
 --source "{\"type\": \"CODECOMMIT\",\"location\": \"https://git-codecommit.eu-west-2.amazonaws.com/v1/repos/kore-example-apps\", \"buildspec\": \"dotnet-hello-world/ci/aws/config/buildspec-sonarcloud.yml\"}" \
 --environment "{\"type\": \"LINUX_CONTAINER\",\"image\": \"aws/codebuild/amazonlinux2-x86_64-standard:3.0\",\"computeType\": \"BUILD_GENERAL1_SMALL\"}" \
 --service-role "arn:aws:iam::${AWS_ACCOUNT_ID}:role/CodeBuildServiceRole" \
 --artifacts "{\"type\": \"NO_ARTIFACTS\"}" \
 --source-version "refs/heads/master"
```

#### Create CloudWatch rule for CI

1. Create an IAM role that enables CloudWatch to start builds for the CodeBuild projects.
```bash
aws iam create-role --profile appvia-workshop-user --role-name CloudWatchServiceRole --assume-role-policy-document "{\"Version\":\"2012-10-17\",\"Statement\":[{\"Effect\":\"Allow\",\"Principal\":{\"Service\":\"events.amazonaws.com\"},\"Action\": \"sts:AssumeRole\"}]}"
aws iam put-role-policy --profile appvia-workshop-user --role-name CloudWatchServiceRole --policy-name CloudWatchServiceRolePolicy --policy-document file://config/iam-cloudwatch-role-policy.json
```

1. Create a CloudWatch rule that triggers a continuous integration workflow when a CodeCommit pull request is created and/or updated.
```bash
AWS_ACCOUNT_ID=$(aws sts get-caller-identity --profile appvia-workshop-user --query "Account" --output text)
AWS_REGION="eu-west-2"
CODECOMMIT_REPOSITORY="kore-example-apps"
CLOUDWATCH_RULE="trigger-ci-workflow-on-pr"
```
```bash
aws events put-rule \
 --profile appvia-workshop-user \
 --name ${CLOUDWATCH_RULE} \
 --description "Continuous integration workflow triggered by CodeCommit Pull Request" \
 --event-pattern "{\"source\":[\"aws.codecommit\"],\"detail-type\":[\"CodeCommit Pull Request State Change\"],\"resources\":[\"arn:aws:codecommit:${AWS_REGION}:${AWS_ACCOUNT_ID}:${CODECOMMIT_REPOSITORY}\"],\"detail\":{\"destinationReference\":[\"refs/heads/master\"],\"event\":[\"pullRequestCreated\",\"pullRequestSourceBranchUpdated\"]}}"
```
```bash
aws events put-targets \
 --profile appvia-workshop-user \
 --rule ${CLOUDWATCH_RULE} \
 --targets '[{"Id": "1", "Arn": "arn:aws:codebuild:'${AWS_REGION}':'${AWS_ACCOUNT_ID}':project/dotnet-hello-world-build-deploy", "RoleArn": "arn:aws:iam::'${AWS_ACCOUNT_ID}':role/CloudWatchServiceRole", "InputTransformer": {"InputPathsMap": {"sourceVersion": "$.detail.sourceCommit"}, "InputTemplate": "{\"sourceVersion\": <sourceVersion>}"}}, {"Id": "2", "Arn": "arn:aws:codebuild:'${AWS_REGION}':'${AWS_ACCOUNT_ID}':project/dotnet-hello-world-code-coverage", "RoleArn": "arn:aws:iam::'${AWS_ACCOUNT_ID}':role/CloudWatchServiceRole", "InputTransformer": {"InputPathsMap": {"sourceVersion": "$.detail.sourceCommit"}, "InputTemplate": "{\"sourceVersion\": <sourceVersion>}"}}]'
```

1. Create a CloudWatch rule that triggers a continuous integration workflow when a CodeCommit pull request is merged into the `master` branch.
```bash
AWS_ACCOUNT_ID=$(aws sts get-caller-identity --profile appvia-workshop-user --query "Account" --output text)
AWS_REGION="eu-west-2"
CODECOMMIT_REPOSITORY="kore-example-apps"
CLOUDWATCH_RULE="trigger-ci-workflow-on-pr-merged"
CODEBUILD_PROJECT="dotnet-hello-world-code-coverage"
```
```bash
aws events put-rule \
 --profile appvia-workshop-user \
 --name ${CLOUDWATCH_RULE} \
 --description "Continuous integration workflow triggered by CodeCommit Pull Request merged" \
 --event-pattern "{\"source\":[\"aws.codecommit\"],\"detail-type\":[\"CodeCommit Pull Request State Change\"],\"resources\":[\"arn:aws:codecommit:${AWS_REGION}:${AWS_ACCOUNT_ID}:${CODECOMMIT_REPOSITORY}\"],\"detail\":{\"destinationReference\":[\"refs/heads/master\"],\"event\":[\"pullRequestMergeStatusUpdated\"]}}"
```
```bash
aws events put-targets \
 --profile appvia-workshop-user \
 --rule ${CLOUDWATCH_RULE} \
 --targets '[{"Id": "1", "Arn": "arn:aws:codebuild:'${AWS_REGION}':'${AWS_ACCOUNT_ID}':project/dotnet-hello-world-build-deploy", "RoleArn": "arn:aws:iam::'${AWS_ACCOUNT_ID}':role/CloudWatchServiceRole"}, {"Id": "2", "Arn": "arn:aws:codebuild:'${AWS_REGION}':'${AWS_ACCOUNT_ID}':project/dotnet-hello-world-code-coverage", "RoleArn": "arn:aws:iam::'${AWS_ACCOUNT_ID}':role/CloudWatchServiceRole"}]'
```

#### Create Kubernetes service account

This section assumes that you have used **Kore Operate** to self serve Kubernetes cluster for your team. If not, then you can create one with a tool of your choice.

1. Create a Kubernetes namespace with the `kore` CLI.
```bash
kore -t <TEAM> create namespace <NAMESPACE>
```

1. Log in to the Kore-managed Kubernetes clusters and set the Kubernetes configuration.
```bash
kore profile configure <KORE_API_URL>
kore login
kore kubeconfig -t <TEAM>
```

1. Create a Kubernetes service account with administrator privileges scoped to a Kubernetes namespace.
```bash
kubectl -n <NAMESPACE> create serviceaccount <SERVICE_ACCOUNT>
kubectl -n <NAMESPACE> create rolebinding <ROLE_BINDING> --clusterrole=kore-nsadmin --serviceaccount=<NAMESPACE>:<SERVICE_ACCOUNT>
```

1. Get the Service account token
```bash
kubectl get secret -n <NAMESPACE> $(kubectl -n <NAMESPACE> get serviceaccount <SERVICE_ACCOUNT> -o jsonpath='{.secrets[0].name}') -o jsonpath='{.data.token}' | base64 --decode
```

#### Create SonarCloud account
Create an account on [SonarCloud](https://sonarcloud.io) and [generate a token](https://sonarcloud.io/account/security/) to access a SonarCloud project.

#### Create Parameters in Systems Manager Parameter store

Add the following as parameters of type `SecureString` in Systems Manager Parameter Store.
```
HELM_KUBEAPISERVER # The Kubernetes API Server URL
HELM_KUBETOKEN     # The Kubernetes Service Account token
HELM_NAMESPACE     # The Kubernetes namespace
EKS_CLUSTER_CA     # The Kubernetes cluster certificate authority (Only applies to AWS EKS)
SONAR_TOKEN        # The SonarCloud token
```

## Triggering the CI pipeline
![AWS CodePipeline CI Pipeline](../../images/aws-codebuild-ci.png)