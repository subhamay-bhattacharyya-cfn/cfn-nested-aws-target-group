# CloudFormation Nested Stac Template : Target Group

<!-- Row 1: Status - Most Important -->
[![Release](https://img.shields.io/github/v/release/subhamay-bhattacharyya-cfn/cfn-nested-aws-target-group?label=Release)](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-target-group/releases)&nbsp;[![Release Workflow](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-target-group/actions/workflows/release.yaml/badge.svg)](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-target-group/actions/workflows/release.yaml)&nbsp;[![Issues](https://img.shields.io/github/issues/subhamay-bhattacharyya-cfn/cfn-nested-aws-target-group)](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-target-group/issues)&nbsp;[![Last Commit](https://img.shields.io/github/last-commit/subhamay-bhattacharyya-cfn/cfn-nested-aws-target-group)](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-target-group/commits)

<!-- Row 2: Code Quality -->
[![Top Language](https://img.shields.io/badge/Languages-Python%20%7C%20YAML-blue)](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-target-group)&nbsp;[![Commits](https://img.shields.io/github/commit-activity/t/subhamay-bhattacharyya-cfn/cfn-nested-aws-target-group)](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-target-group/commits)

<!-- Row 3: Tech Stack -->
[![CloudFormation](https://img.shields.io/badge/CloudFormation-IaC-orange?logo=amazonaws&logoColor=white)](https://aws.amazon.com/cloudformation/)&nbsp;[![Built with Claude Code](https://img.shields.io/badge/Built_with-Claude_Code-D97757?logo=anthropic&logoColor=white)](https://claude.ai/)

<!-- Row 4: Repository Info -->
[![Files](https://img.shields.io/github/directory-file-count/subhamay-bhattacharyya-cfn/cfn-nested-aws-target-group)](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-target-group)&nbsp;[![Repo Size](https://img.shields.io/github/repo-size/subhamay-bhattacharyya-cfn/cfn-nested-aws-target-group)](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-target-group)&nbsp;[![Release Date](https://img.shields.io/github/release-date/subhamay-bhattacharyya-cfn/cfn-nested-aws-target-group)](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-target-group/releases)

<!-- Row 5: Custom Metrics -->
[![Custom Endpoint](https://img.shields.io/endpoint?url=https://gist.githubusercontent.com/bsubhamay/88973aebd3457ddfae0c12fe9b6971a1/raw/cfn-nested-aws-target-group.json)](https://gist.github.com/subhamay-bhattacharyya/88973aebd3457ddfae0c12fe9b6971a1)

A nested CloudFormation template that creates an IP-type Application Load Balancer target group for either the public **Web** tier or the internal **App** tier.

## Overview

The template is designed to be invoked from a parent/root CloudFormation stack. A single template covers both tiers; the `TierType` parameter selects which tier's defaults apply.

- **Template:** [cloudformation/template.yaml](cloudformation/template.yaml)
- **Parameter file:** [cloudformation/parameters.json](cloudformation/parameters.json)
- **Stack config:** [cloudformation/stack-config.json](cloudformation/stack-config.json)

## Tier Defaults

| Setting | Web tier (public) | App tier (internal) |
| --- | --- | --- |
| `TierType` | `web` | `app` |
| Service name | `web` | `app` |
| Target group port | `80` | `3001` |
| Health check path | `/` | `/api/books` |
| Export name | `{ProjectName}-{Environment}-web-tg-arn` | `{ProjectName}-{Environment}-app-tg-arn` |

`Port` and `HealthCheckPath` can be set explicitly to override these defaults. Leave them empty to use the tier defaults.

## Parameters

| Parameter | Type | Required | Default | Description |
| --- | --- | --- | --- | --- |
| `ProjectName` | String | No | `cfn-nested` | Project prefix. Lowercase letters, digits, and hyphens, up to 12 characters |
| `Environment` | String | No | `dev` | One of `dev`, `staging`, `prod` |
| `TierType` | String | No | `web` | One of `web`, `app` |
| `Port` | String | No | `""` | Port override (0–5 digits). Empty uses the tier default |
| `HealthCheckPath` | String | No | `""` | Health check path override. Empty uses the tier default |
| `VpcId` | `AWS::EC2::VPC::Id` | Yes | — | VPC the target group is created in |
| `CiSuffix` | String | No | `""` | Optional suffix for CI deployments. Lowercase letters, digits, and hyphens |

## Naming

The target group name and the `Name` tag follow this pattern:

```bash
{ProjectName}-{Environment}-{ServiceName}-tg
```

When `CiSuffix` is set, the suffix is inserted before `-tg`:

```bash
{ProjectName}-{Environment}-{ServiceName}-{CiSuffix}-tg
```

Examples:

- `cfn-nested-prod-web-tg`
- `cfn-nested-dev-app-tg-42`

Target group names are limited to 32 characters. Long project, environment, or service names combined with a CI suffix can exceed this limit and fail at deploy time.

## Implementation Details

### Mappings and Conditions

- **`TierSettings` mapping** holds the service name, port, and health check path for each tier. The resources read values from it with `!FindInMap` using `!Ref TierType`.
- **`HasCiSuffix`** is true when `CiSuffix` is not empty. It chooses between the name with and without the suffix.
- **`HasPortOverride`** and **`HasHealthCheckPathOverride`** are true when the corresponding parameter is not empty. They choose between the parameter value and the tier default.

### Resources

A single `AWS::ElasticLoadBalancingV2::TargetGroup` named `TargetGroup` is created with these fixed settings:

| Property | Value |
| --- | --- |
| `Protocol` | `HTTP` |
| `TargetType` | `ip` |
| `HealthCheckProtocol` | `HTTP` |
| `HealthCheckPort` | `traffic-port` |
| `HealthCheckIntervalSeconds` | `30` |
| `HealthCheckTimeoutSeconds` | `5` |
| `HealthyThresholdCount` | `3` |
| `UnhealthyThresholdCount` | `3` |
| `Matcher.HttpCode` | `200-299` |

Tags:

| Key | Value |
| --- | --- |
| `Name` | Target group name (see [Naming](#naming)) |
| `Project` | `ProjectName` |
| `Environment` | `Environment` |
| `Tier` | `TierType` |
| `Service` | Service name from the tier mapping |

### Outputs

| Output | Value | Export |
| --- | --- | --- |
| `TargetGroupArn` | Target group ARN (`!Ref TargetGroup`) | `{ProjectName}-{Environment}-{ServiceName}-tg-arn` |
| `TargetGroupName` | Full target group name (`!GetAtt TargetGroup.TargetGroupName`) | — |

The `TargetGroupArn` export is what the ALB stack's listener rules should import.

## Usage

### 1. Reference from a Parent Stack

```yaml
WebTargetGroupStack:
  Type: AWS::CloudFormation::Stack
  Properties:
    TemplateURL: https://s3.amazonaws.com/your-cfn-bucket/templates/template.yaml
    Parameters:
      ProjectName: !Ref ProjectName
      Environment: !Ref Environment
      TierType: web
      VpcId: !Ref VpcId
      CiSuffix: !Ref CiSuffix
    Tags:
      - Key: Environment
        Value: !Ref Environment

AppTargetGroupStack:
  Type: AWS::CloudFormation::Stack
  Properties:
    TemplateURL: https://s3.amazonaws.com/your-cfn-bucket/templates/template.yaml
    Parameters:
      ProjectName: !Ref ProjectName
      Environment: !Ref Environment
      TierType: app
      VpcId: !Ref VpcId
      CiSuffix: !Ref CiSuffix
    Tags:
      - Key: Environment
        Value: !Ref Environment
```

### 2. Upload the Template to S3

```bash
aws s3 cp cloudformation/template.yaml s3://your-cfn-bucket/templates/template.yaml
```

### 3. Deploy Directly with the AWS CLI

Create a web tier target group:

```bash
aws cloudformation deploy \
  --template-file cloudformation/template.yaml \
  --stack-name cfn-nested-web-tg-dev \
  --parameter-overrides \
    ProjectName=cfn-nested \
    Environment=dev \
    TierType=web \
    VpcId=vpc-0123456789abcdef0
```

Create an app tier target group with a port override:

```bash
aws cloudformation deploy \
  --template-file cloudformation/template.yaml \
  --stack-name cfn-nested-app-tg-dev \
  --parameter-overrides \
    ProjectName=cfn-nested \
    Environment=dev \
    TierType=app \
    Port=3002 \
    VpcId=vpc-0123456789abcdef0
```

### 4. Validate the Template

```bash
aws cloudformation validate-template --template-body file://cloudformation/template.yaml
cfn-lint cloudformation/template.yaml
```

## Best Practices Implemented

- ✅ One template for both web and app tiers, driven by `TierType`
- ✅ IP target type for ECS/EKS and other IP-addressed backends
- ✅ Health checks on the traffic port with explicit success codes
- ✅ Consistent names, tags, and exports across tiers and environments
- ✅ Optional CI suffix for unique deployments
- ✅ Export values for cross-stack references

## License

MIT
