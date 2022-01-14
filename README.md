# Configure AWS Credentials with GitHub Actions

Demonstration repository how to configure AWS and GitHub Actions to use GitHub's OIDC Token endpoint.

## Requirements

 - AWS CLI v2
 - AWS Account with Administration rights

## Setup

### Setup GitHub OIDC provider

Recommended way of connecting AWS to GitHub is setup AWS Identity Provider. 

Create stack with GitHub Idenity Provider from the [template](cloudformation/GitHubIdentityProvider.cfn.yaml:

```bash
STACK_NAME="GitHubIdentityProvider"

# Create Stack
aws cloudformation create-stack \
    --stack-name $STACK_NAME \
    --template-body file://cloudformation/GitHubIdentityProvider.cfn.yaml

# Update Stack
aws cloudformation update-stack \
    --stack-name $STACK_NAME \
    --template-body file://cloudformation/GitHubIdentityProvider.cfn.yaml

# Wait till finish
aws cloudformation wait stack-create-complete --stack-name $STACK_NAME

# Get GithubOidcId Value
OIDC_PROVIDER=`aws cloudformation describe-stacks \
    --stack-name $STACK_NAME \
    --query "Stacks[].Outputs[?OutputKey=='GithubOidcId'].OutputValue" \
    --output text`
echo $OIDC_PROVIDER
```

As a result you should get Identity Provider ARN, like:
```
arn:aws:iam::1234567890:oidc-provider/token.actions.githubusercontent.com
```

### Setup IAM role 

Setup IAM role that allow write-access to AWS ECR.

Specify with repositories are able to use the role:
```bash
# Organization level limit
GITHUB_ACCESS_SCOPE="octo-org/*:ref:*"
# Repository level limit
GITHUB_ACCESS_SCOPE="octo-org/octo-repo:ref:*"
# Branch level
GITHUB_ACCESS_SCOPE="octo-org/octo-repo:ref:refs/heads/octo-branch"
```

Create stack with new role from the [template](cloudformation/GitHubECRPublishRole.yaml):

```bash
GITHUB_ACCESS_SCOPE="octo-org/octo-repo:ref:refs/heads/octo-branch"
ROLE_NAME="octo-org"
STACK_NAME="GitHubOctoOrgPushECRRole"

# Create Stack
aws cloudformation create-stack \
    --stack-name $STACK_NAME \
    --template-body file://cloudformation/GitHubECRPublishRole.yaml \
    --capabilities CAPABILITY_NAMED_IAM \
    --parameters \
        ParameterKey=OIDCProviderArn,ParameterValue=$OIDC_PROVIDER \
        ParameterKey=RoleName,ParameterValue=$ROLE_NAME \
        ParameterKey=GitHubAccessScope,ParameterValue=$GITHUB_ACCESS_SCOPE

# Wait till finish
aws cloudformation wait stack-create-complete --stack-name $STACK_NAME

# Get created role ARN
aws cloudformation describe-stacks \
    --stack-name $STACK_NAME \
    --query "Stacks[].Outputs[?OutputKey=='RoleArn'].OutputValue" \
    --output text
```

As a result you should see IAM ROLE ARN, like:
```
arn:aws:iam::1234567890:role/github-actions-octo-org-ecr
```

### Usage

#### Test pushing new image to ECR

1. Create new [GitHub repository](https://github.com/new) and clone on your local
2. Create testing `Dockerfile` file:
   ```
   FROM busybox

   CMD ["echo", "Hello GitHub Actions!"]
   ```
3. Create `.github/workflows/ci-docker.yml` file:
    ```yaml
    on:
      push:

    env:
      AWS_REGION: "us-east-1"
      AWS_ROLE: "arn:aws:iam::1234567890:role/github-actions-octo-org-ecr" # TODO: FILL ME!
      ECR_REGISTRY: "1234567890.dkr.ecr.us-east-1.amazonaws.com"           # TODO: FILL ME!
      ECR_REPOSITORY: "github/test-repository"

    permissions:
      id-token: write
      contents: read

    jobs:
      release:
        name: Build and push
        runs-on: ubuntu-latest
        steps:

          - name: Checkout
            uses: actions/checkout@v2

          -
            name: Docker meta
            id: meta
            uses: docker/metadata-action@v3
            with:
              images: "${{ env.ECR_REGISTRY }}/${{ env.ECR_REPOSITORY }}"
              labels: |
                maintainer=Piotr Plenik
                org.opencontainers.image.title=MyTestRepository
                org.opencontainers.image.description=Another description
                org.opencontainers.image.vendor=ACME company

          - name: Set up Docker Buildx
            id: buildx
            uses: docker/setup-buildx-action@v1

          - name: Configure AWS Credentials
            uses: aws-actions/configure-aws-credentials@v1
            with:
              role-to-assume: ${{ env.AWS_ROLE }}
              aws-region: ${{ env.AWS_REGION }}

          - name: Login to Amazon ECR
            id: login-ecr
            uses: docker/login-action@v1
            with:
              registry: ${{ env.ECR_REGISTRY }}

          - name: Build, tag, and push image to Amazon ECR
            uses: docker/build-push-action@v2
            with:
              builder: ${{ steps.buildx.outputs.name }}
              context: .
              platforms: linux/amd64,linux/arm64,linux/ppc64le
              push: true
              tags: ${{ steps.meta.outputs.tags }}
              labels: ${{ steps.meta.outputs.labels }}
     ```
3. Push al changes and see magic!

## References

 - [GitHub - Configuring OpenID Connect in Amazon Web Services](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services)
 - [GitHub security hardening with OpenID Connect](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/about-security-hardening-with-openid-connect)
 - [GitHub Action - Configure AWS Credentials](https://github.com/aws-actions/configure-aws-credentials)
 - [GitHub Action - Docker metadata](https://github.com/docker/metadata-action)
 - [GitHub Action - Docker Build Push](https://github.com/docker/build-push-action)