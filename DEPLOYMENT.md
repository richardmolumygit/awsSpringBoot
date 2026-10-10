# GitHub Actions deployment to Amazon EKS

The workflow at `.github/workflows/build-deploy-eks.yml` builds and then deploys on every run:

- A push to `main`, `develop`, or `feature/**` builds the image and deploys it to `dev`.
- Use **Actions > Build and deploy to EKS > Run workflow** and select `build-and-deploy` to build the selected branch's current commit, then deploy it to `dev`, `qa`, or `prod`.
- Select `deploy-only` to deploy an existing image without rebuilding. Enter its full 40-character commit SHA in `image_tag`.
- Images are tagged with the full commit SHA. Re-running a build skips publishing if that SHA already exists.

The `dev` Service uses an internet-facing AWS load balancer. After deployment, wait for its hostname with `kubectl get svc springwebpage -n dev`; open `http://<EXTERNAL-IP>` in a browser. This requires AWS load-balancer provisioning and public subnets in the EKS cluster. `qa` and `prod` remain `ClusterIP` unless overridden.

The build job runs `./mvnw -B package`; the Dockerfile copies that JAR into the Java runtime image. To build the image locally, run:

```bash
./mvnw -B package
docker build -t springwebpage:local .
```

## GitHub setup

Create GitHub Environments named `build`, `dev`, `qa`, and `prod` in the repository settings.

Add these **repository variables**. `IMAGE_REPOSITORY` is the full ECR repository URI without a tag, for example `123456789012.dkr.ecr.us-east-1.amazonaws.com/springwebpage`. The same ECR repository must be used for every deployment environment.

| Repository variable | Value |
| --- | --- |
| `IMAGE_REPOSITORY` | Shared ECR repository URI, without an image tag |
| `IMAGE_AWS_REGION` | AWS region containing the shared ECR repository |

Add `AWS_ROLE_ARN` as an **environment variable** in the `build` environment and in each of `dev`, `qa`, and `prod`. The build environment role is used only to push images; each deployment environment role is used only to access its EKS cluster. Add these additional environment variables to `dev`, `qa`, and `prod`:

| Environment variable | Value |
| --- | --- |
| `AWS_REGION` | AWS region containing this environment's EKS cluster |
| `EKS_CLUSTER_NAME` | EKS cluster name for this environment |

The EKS region and cluster can differ per environment. Configure required reviewers for `prod` in its GitHub Environment settings if production deployments need approval.

## AWS setup

Configure the AWS IAM OIDC provider for `https://token.actions.githubusercontent.com` with audience `sts.amazonaws.com`. The build role must trust this GitHub repository's environment subject `repo:OWNER/REPOSITORY:environment:build`; each deployment role must trust the corresponding `dev`, `qa`, or `prod` environment subject. Replace `OWNER/REPOSITORY` with the GitHub repository path.

The build role needs permission to authenticate to ECR, push images, and call `ecr:DescribeImages` on the shared repository. Each deployment role needs permission to describe its EKS cluster. The role must also be granted Kubernetes access to the target namespaces through an EKS access entry or the cluster's existing access configuration; AWS IAM permissions alone do not grant Helm access to the cluster. Ensure the EKS worker nodes can pull images from the shared ECR repository. Limit Kubernetes deployment access to the target namespaces where possible.

Create the ECR repository before running the build workflow and enable tag immutability. The chart's `values-dev.yaml`, `values-qa.yaml`, and `values-prod.yaml` files are intentionally empty override points.

## Production credential warning

`src/main/resources/application.properties` currently contains the literal Spring Security credentials `admin` and `password`. Replace these with secret-backed configuration before exposing the application publicly or deploying to a production environment.