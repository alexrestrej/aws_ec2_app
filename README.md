# AWS Windows infrastructure deployment

This repository contains AWS CloudFormation templates and an Azure Pipelines YAML
pipeline for deploying Windows EC2 infrastructure and its supporting resources to
development, QA, and production environments.

The deployment is a generic starting point. Replace every `YOUR_*` and
`#{...}#` marker with values for your own AWS accounts, application, network,
and organization before running it. Do not commit credentials or other secrets.

## Repository layout

```text
.
├── azure-pipeline.yaml
├── cloudformation/
│   ├── template-ec2.yaml
│   ├── template-kms.yaml
│   ├── template-reporting.yaml
│   ├── template-s3.yaml
│   ├── template-sg.yaml
│   └── template-sm.yaml
└── config/
    ├── params-ec2.json
    ├── params-kms.json
    ├── params-reporting.json
    ├── params-s3.json
    ├── params-sg.json
    └── params-sm.json
```

Each `template-*.yaml` file defines a CloudFormation stack. Its corresponding
`params-*.json` file provides the deployment parameters. `template-kms.yaml`
does not currently declare input parameters; its parameter file is empty.

## Infrastructure stacks

The pipeline deploys these stacks in order:

| Stack | Main purpose |
| --- | --- |
| `kms` | Creates a KMS key, alias, and SSM parameters containing the key ID and ARN. |
| `s3` | Creates private, encrypted S3 buckets for application content and optional restore data, plus related policy/SSM resources. |
| `sg` | Creates security groups for application and reporting instances, including environment-specific ingress rules. |
| `sm` | Creates Secrets Manager secrets for three database profiles and restricts access with resource policies. |
| `reporting` | Provisions the Windows reporting EC2 instance, storage, instance role, and certificate-related resources. |
| `ec2` | Provisions the Windows application EC2 instance, instance role/profile, storage, certificates, and bootstrap configuration. |

The order is intentional: later stacks refer to resources or parameters created by
earlier stacks. Confirm the dependencies and account/region-specific references
in the templates before changing the order.

## Deployment pipeline

[`azure-pipeline.yaml`](./azure-pipeline.yaml) triggers on pushes to `trunk` and
contains three sequential stages:

1. `deploy_dev`
2. `deploy_qa`, after `deploy_dev`
3. `deploy_prod`, after `deploy_qa`

Each stage loops through the six stacks above and runs `aws cloudformation
deploy` using the matching template and parameter file. The stack name is formed
from `appCode`, `projectName`, the environment name, and the stack suffix, for
example `<appCode>-<projectName>-qa-ec2`.

The pipeline currently runs on an Azure Pipelines `ubuntu-latest` hosted agent.
It expects Python 3 and AWS CLI to be available on that agent. It does not install
or configure either tool. It passes the AWS region and credentials to each
deployment job using these pipeline variables:

| Environment | AWS credential variables | Region variable |
| --- | --- | --- |
| dev | `AWS_ACCESS_KEY_ID_DEV`, `AWS_SECRET_ACCESS_KEY_DEV`, optional `AWS_SESSION_TOKEN_DEV` | `AWS_REGION_DEV` |
| qa | `AWS_ACCESS_KEY_ID_QA`, `AWS_SECRET_ACCESS_KEY_QA`, optional `AWS_SESSION_TOKEN_QA` | `AWS_REGION_QA` |
| prod | `AWS_ACCESS_KEY_ID_PROD`, `AWS_SECRET_ACCESS_KEY_PROD`, optional `AWS_SESSION_TOKEN_PROD` | `AWS_REGION_PROD` |

Create these as secret pipeline variables or use an approved secure credential
integration. The deployment identity needs permission to create, update, and
inspect the resources in the templates, pass required IAM roles, and create
CloudFormation stacks. The pipeline uses
`CAPABILITY_IAM` and `CAPABILITY_NAMED_IAM`.

## Configuration before deployment

1. Set `appCode` and `projectName` in `azure-pipeline.yaml`.
2. Configure the AWS credentials and region variables for all three environments.
3. Populate the parameter files under `config/` with values appropriate for the
   AWS account and environment being deployed. The files contain generic
   placeholders; the pipeline passes their values directly to CloudFormation and
   does not expand `#{...}#` tokens or select environment-specific parameter
   files.
4. Replace remaining `YOUR_*` markers in the templates, including application
   paths/names, deployment role details, business tags, and VPN SSM parameter
   names.
5. Review network rules, IAM policies, resource names, KMS key policies, and
   region-specific service references. In particular, check that the network
   ranges and the referenced VPC, subnets, security groups, roles, certificates,
   buckets, and SSM parameters exist in each target environment.
6. Confirm that the application installer, bootstrap scripts, monitoring
   configuration, and other files referenced by EC2 user data are available at
   the configured S3 locations.

The same `config/params-*.json` files are used for dev, QA, and production. If
each environment needs different settings, provide environment-specific
parameter files or update the pipeline to select separate configuration files
before deploying.

### Protecting sensitive values

Do not store passwords, access keys, tokens, private keys, or production network
details in source control. Prefer injecting secrets through an approved secret
store or secure pipeline integration. The current pipeline reads parameter
values from checked-in JSON files and passes them as command-line overrides;
avoid putting plaintext credentials in those files. CloudFormation `NoEcho`
helps hide a parameter in some CloudFormation views, but it does not make it
safe to commit or pass as an ordinary command-line value.

## Running the pipeline

1. Review all placeholders and environment configuration.
2. Ensure the target accounts and regions are ready, and that required shared
   resources exist.
3. Run the pipeline from `trunk`.
4. Monitor each stage and the CloudFormation stack events in AWS. Later
   environments run only after the previous deployment stage succeeds.

To deploy a single stack manually, use AWS CLI with the correct account, region,
template, stack name, and parameter overrides. For example:

```sh
aws cloudformation deploy \
  --template-file cloudformation/template-kms.yaml \
  --stack-name <stack-name> \
  --capabilities CAPABILITY_IAM CAPABILITY_NAMED_IAM
```

For parameterized stacks, provide the values from the matching
`config/params-<stack>.json` file with `--parameter-overrides`. Do not use
placeholder values for a real deployment.

## Operational notes

- The pipeline has no validation or security-scan stage; it deploys the
  templates directly. Review changes and deployment permissions before running
  it, especially for production.
- `--no-fail-on-empty-changeset` makes a deployment succeed when CloudFormation
  reports no changes.
- Some templates create IAM roles or policies, EC2 instances, S3 buckets,
  KMS keys, and secrets. Review retention/deletion behavior and possible costs
  before deployment.
- The KMS policy includes service endpoints for `us-east-1`. If deploying to a
  different region, verify and update the regional service references.
- The security group template expects configured CIDR values and SSM parameter
  names. Review the allowed ports and source ranges against your organization's
  network and security requirements.
- Stack names, SSM parameter names, resource names, and some references are
  derived from the application and project identifiers. Keep those values
  consistent across the pipeline and parameter files.

## Troubleshooting

- **Missing CloudFormation parameter:** compare the parameter keys in
  `config/params-<stack>.json` with the `Parameters` section in the matching
  template.
- **Unresolved placeholder:** check that no `YOUR_*` or `#{...}#` values remain
  in the files used by the deployment.
- **Access denied:** verify the pipeline identity's AWS account, region,
  CloudFormation permissions, IAM `PassRole` permissions, and access to
  referenced S3, KMS, Secrets Manager, and SSM resources.
- **Resource not found:** verify that referenced VPCs, subnets, SSM parameters,
  certificates, buckets, and IAM roles exist in the target account and region.
- **EC2 bootstrap failure:** inspect EC2 system/user-data logs and confirm the
  instance role can retrieve secrets and download the expected S3 objects.
