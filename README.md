# efs_backed_conda

This repository demonstrates how to create a custom Amazon SageMaker Studio kernel image which uses an EFS backed conda environment.

## Pre-Requisites

In order to build a custom kernel your Amazon SageMaker Studio Execution role will require additional permissions to:

1. Run a AWS CodeBuild project
2. Build and register a docker container with the Amazon Elastic Container Registry (ECR)
3. Update your SageMaker domain to attach the kernel image.

### Trust Relationship with AWS CodeBuild

The first is that the Sagemaker Execution Policy should have a trust policy with CodeBuild. So that it can execute the image build using CodeBuild.

Go to IAM and find your Sagemaker Execution Role. Then edit to the Trust Relationships to include `codebuild.amazonaws.com`.

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": [
          "sagemaker.amazonaws.com",
          "codebuild.amazonaws.com"
        ]
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

### AWS CodeBuild Permissions

You also need to make sure the appropriate permissions are included in your role to run the build in CodeBuild, create a repository in Amazon ECR, and push images to that repository.

Go to IAM and find your Sagemaker Execution Role, and add a new inline policy with the following JSON definition, replacing `<<SageMakerExecutionRoleArn>>` with your the ARN of your SageMaker Execution Role:

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "codebuild:DeleteProject",
                "codebuild:CreateProject",
                "codebuild:BatchGetBuilds",
                "codebuild:StartBuild"
            ],
            "Resource": "arn:aws:codebuild:*:*:project/sagemaker-studio*"
        },
        {
            "Effect": "Allow",
            "Action": "logs:CreateLogStream",
            "Resource": "arn:aws:logs:*:*:log-group:/aws/codebuild/sagemaker-studio*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "logs:GetLogEvents",
                "logs:PutLogEvents"
            ],
            "Resource": "arn:aws:logs:*:*:log-group:/aws/codebuild/sagemaker-studio*:log-stream:*"
        },
        {
            "Effect": "Allow",
            "Action": "logs:CreateLogGroup",
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "ecr:CreateRepository",
                "ecr:BatchGetImage",
                "ecr:CompleteLayerUpload",
                "ecr:DescribeImages",
                "ecr:DescribeRepositories",
                "ecr:UploadLayerPart",
                "ecr:ListImages",
                "ecr:InitiateLayerUpload",
                "ecr:BatchCheckLayerAvailability",
                "ecr:PutImage"
            ],
            "Resource": "arn:aws:ecr:*:*:repository/sagemaker-studio*"
        },
        {
            "Effect": "Allow",
            "Action": "ecr:GetAuthorizationToken",
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": [
              "s3:GetObject",
              "s3:DeleteObject",
              "s3:PutObject"
              ],
            "Resource": "arn:aws:s3:::sagemaker-*/*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "s3:CreateBucket"
            ],
            "Resource": "arn:aws:s3:::sagemaker*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "iam:GetRole",
                "iam:ListRoles"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": "iam:PassRole",
            "Resource": "<<SageMakerExecutionRoleArn>>",
            "Condition": {
                "StringLikeIfExists": {
                    "iam:PassedToService": "codebuild.amazonaws.com"
                }
            }
        }
    ]
}
```

### Permissions to Modify Amazon Sagemaker Studio Domain

Finally you will need to add a policy to your Role that will allow you to modify the Studio domain. This is the final step where you will make your custom kernel available within Sagemaker Studio to run a Notebook.

Go to IAM and find your Sagemaker Execution Role, and add a new inline policy with the following JSON definition:

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "sagemaker:CreateApp",
                "sagemaker:CreateAppImageConfig",
                "sagemaker:CreateDomain",
                "sagemaker:CreateImage",
                "sagemaker:CreateImageVersion",
                "sagemaker:UpdateDomain"
            ],
            "Resource": "*"
        }
    ]
}
```

## Installation steps

1. Build the [docker image](https://docs.aws.amazon.com/sagemaker/latest/dg/studio-byoi-sdk-add-container-image.html) and push to ECR:

```bash
pip install sagemaker-studio-image-build
sm-docker build . --repository sagemaker-studio-conda-efs:1.0
```

Make a note of the `Image URI` in the output.

2. Attach the docker image to the SageMaker Studio Domain. You can use the AWS Console or the command line as described below:

```bash
# Set environment variables
export IMAGE_URI=<<Image URI>>
export SAGEMAKER_EXECUTION_ROLE=<<SageMakerExecutionRoleArn>>
export DOMAIN_ID=<<SageMakerDomainID>>

# Create image, app, and update domain
aws sagemaker create-image --image-name conda-efs --role-arn $SAGEMAKER_EXECUTION_ROLE --display-name "Conda with EFS backed env"
aws sagemaker create-image-version --base-image $IMAGE_URI --image-name conda-efs
aws sagemaker create-app-image-config --cli-input-json file://app-image-config-input.json
aws sagemaker update-domain --domain-id $DOMAIN_ID --cli-input-json file://update-domain.json
```

`NOTE`: If you are updating an existing app-image, use `update-app-image` command.


3. From Studio use the Image terminal of the **datascience** first-party kernel image and create the conda environment in the following way:

```bash
mkdir -p ~/.conda/envs
conda create -p ~/.conda/envs/custom
conda activate custom
conda install scikit-learn numpy
```

Also create a **.condarc** file on the EFS volume with the following content:

```
envs_dirs:
  - ~/.conda/envs
```

3. Create a notebook with the custom EFS backed conda image.

You can list the current conda environment by running:

```
conda env list
```

4. For installing packages permanently on EFS, use the image terminal of the custom image. You can install packages like in the example below:

```bash
conda activate custom
conda install seaborn pyarrow
```



