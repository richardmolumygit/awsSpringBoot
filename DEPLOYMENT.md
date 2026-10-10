# GitHub Actions deployment to Amazon EKS

The workflow at `.github/workflows/deploy-eks.yml` builds the Docker image, pushes it to Amazon ECR, and deploys it with Helm. A push to `main` deploys to `dev`; use **Actions > Build and deploy to EKS > Run workflow** to choose `dev`, `qa`, or `prod` manually.

The workflow first runs `./mvnw -B package`; the Dockerfile then copies that JAR into the Java runtime image. To build the image locally, run:

```bash
./mvnw -B package
docker build -t springwebpage:local .
```

## GitHub setup

Create GitHub Environments named `dev`, `qa`, and `prod` in the repository settings. Add these **environment variables** to each environment:

| Variable | Value |
| --- | --- |
| `AWS_REGION` | AWS region containing the ECR repository and EKS cluster |
| `AWS_ROLE_ARN` | ARN of the IAM role GitHub Actions assumes using OIDC |
| `EKS_CLUSTER_NAME` | EKS cluster name for this environment |
| `ECR_REPOSITORY` | Existing ECR repository name (not its full URI) |

The values can differ per environment. Configure required reviewers for `prod` in its GitHub Environment settings if production deployments need approval.

## AWS setup

Configure the AWS IAM OIDC provider for `https://token.actions.githubusercontent.com` with audience `sts.amazonaws.com`. Each environment's `AWS_ROLE_ARN` must trust this GitHub repository's environment subject, for example `repo:OWNER/REPOSITORY:environment:dev` (and corresponding `qa` and `prod` subjects). Replace `OWNER/REPOSITORY` with the GitHub repository path.

The assumed role needs permission to authenticate to ECR and push to the configured repository, and permission to describe the configured EKS cluster. The role must also be granted Kubernetes access to the target namespaces through an EKS access entry or the cluster's existing access configuration; AWS IAM permissions alone do not grant Helm access to the cluster. Limit that Kubernetes access to the deployment namespaces where possible.

Create the ECR repository before running the workflow. The workflow tags each image with the full Git commit SHA and deploys that exact tag. The chart's `values-dev.yaml`, `values-qa.yaml`, and `values-prod.yaml` files are intentionally empty override points.

## Production credential warning

`src/main/resources/application.properties` currently contains the literal Spring Security credentials `admin` and `password`. Replace these with secret-backed configuration before deploying to a production environment.