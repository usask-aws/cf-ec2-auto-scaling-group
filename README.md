# cf-ec2-auto-scaling-group.yaml

Creates an EC2 Launch Template and Auto Scaling Group using a parameterized
VPC, subnets, security groups, and IAM instance profile. Instances launch
from the latest Amazon Linux 2023 AMI by default (via SSM parameter), with
IMDSv2 enforced and the root EBS volume explicitly encrypted.

## What this template does not do

- **No application bootstrap.** Unlike `Usask_EC2_Loadbalancer_cloudformation_template.yaml`
  and the `Usask_EC2_Loadbalancer_Test_Prod_*` template, this template's
  `LaunchTemplate` has no `UserData` — instances launch as bare Amazon Linux
  2023 with nothing installed. If you need `httpd`/PHP (or anything else)
  on the instances, add a `UserData` block to `LaunchTemplateData` before
  deploying, or bootstrap the instances some other way (e.g. the CodeBuild
  → SSM deploy flow in `cf-codebuild-ec2-deployment.yaml`,
  which pushes and runs a deploy script on already-running, tagged
  instances).
- **No load balancer / target group registration.** The `AutoScalingGroup`
  resource here doesn't register instances with any target group — this is
  a standalone ASG with no ALB in front of it. If you need instances behind
  an Application Load Balancer, use one of the `Usask_EC2_Loadbalancer_*`
  templates instead, or add `TargetGroupARNs` to the `AutoScalingGroup`
  resource pointing at a target group from one of those templates (or your
  own) before deploying.
- **No security groups created.** `SecurityGroupIds` is a required
  parameter — you must already have security group(s) allowing whatever
  inbound access your workload needs; this template doesn't create or
  manage them.

## Deploy from the AWS Console

1. Sign in to the AWS Console for the target account and switch to the
   correct region (top-right region selector).
2. Go to the **CloudFormation** service.
3. Click **Create stack** → **With new resources (standard)**.
4. Under **Prerequisite - Prepare template**, select **Choose an existing
   template**.
5. Under **Specify template**, select **Upload a template file**, click
   **Choose file**, and select your (possibly `UserData`-edited copy of)
   `cf-ec2-auto-scaling-group.yaml`. Click **Next**.
6. On **Specify stack details**:
   - Enter a **Stack name** (e.g. `app-asg-stack`).
   - Fill in the parameters:
     | Parameter | Notes |
     |---|---|
     | `AmiId` | Leave default to use the latest Amazon Linux 2023 AMI via SSM, or override |
     | `InstanceType` | Default `t2.micro` |
     | `SubnetIds` | One or more subnets the instances launch into |
     | `SecurityGroupIds` | One or more existing security groups (see **What this template does not do** above) |
     | `InstanceProfileName` | Name of an existing IAM instance profile (defaults to `DefaultEC2InstanceProfile`) |
     | `MinSize` / `MaxSize` / `DesiredCapacity` | Auto Scaling Group sizing (defaults `1`/`1`/`1`) |
     | `KeyPairName` | Optional, leave blank to disable SSH access |
     | `LaunchTemplateName` | Name of the launch template (default `EC2-Launch-Template`) |
     | `AutoScalingGroupName` | Name of the ASG (default `EC2-ASG`) |
     | `EC2Name` | Sets the `Name` tag propagated to launched instances (default `EC2-Instance`) |
   - Click **Next**.
7. On **Configure stack options**, leave defaults (add tags/permissions
   boundary only if your account requires them) and click **Next**.
8. On **Review**, scroll down and confirm the stack details, then click
   **Submit**.
9. Wait for the stack **Status** to reach `CREATE_COMPLETE`.

## Verify the deployment in the console

1. **CloudFormation** → open the stack → **Outputs** tab → note
   `LaunchTemplateId`, `AutoScalingGroupName`.
2. **EC2 → Auto Scaling Groups** → open `AutoScalingGroupName` → **Instance
   management** tab → confirm the expected number of instances (per
   `DesiredCapacity`) are running and show **Status check: 2/2 checks
   passed**.
3. **EC2 → Instances** → select one of the launched instances → **Storage**
   tab → confirm the root EBS volume shows **Encrypted: Yes**.
4. Still on the instance, confirm the **IAM Role** matches
   `InstanceProfileName` and **Instance metadata service (IMDS)** shows
   **IMDSv2: Required**.

## Updating or deleting the stack

- **Update**: select the stack in CloudFormation → **Update** → **Replace
  current template** (re-upload the YAML if it changed) or **Use current
  template** (to change parameters only) → adjust parameters → **Next** →
  **Submit**. Changes to `LaunchTemplateData` (AMI, instance type, security
  groups, etc.) create a new launch template version; the ASG picks it up
  via `Version: !GetAtt LaunchTemplate.LatestVersionNumber` on the next
  instance replacement, not retroactively on already-running instances.
- **Delete**: select the stack → **Delete** → confirm. This terminates all
  instances in the Auto Scaling Group and deletes the launch template.
  There is no retained state (no data volumes, no S3 bucket) — anything on
  an instance's root volume is lost once it's deleted.
