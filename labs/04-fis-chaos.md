# Lab 04: FIS Chaos

In this lab we will perform a Chaos Experiment using the AWS Fault Injection Simulator service.

- **NOTE:** For this Lab we created another cluster in a different region and configured DNS failover in order to make the application resilient.
  - For more info on Route53 DNS failover: https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/dns-failover-how-to.html

## Configure FIS Role

1. Navigate to the IAM console and create a new IAM policy. On the “Create Policy” page select the JSON tab
  ![fis-1](/images/fis-1.png)

2. Paste the following policy. Take the time to look at how broad these permissions are:
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AllowFISExperimentRoleReadOnly",
            "Effect": "Allow",
            "Action": [
                "ec2:DescribeInstances",
                "ecs:DescribeClusters",
                "ecs:ListContainerInstances",
                "eks:DescribeNodegroup",
                "iam:ListRoles",
                "rds:DescribeDBInstances",
                "rds:DescribeDbClusters",
                "ssm:ListCommands"
            ],
            "Resource": "*"
        },
        {
            "Sid": "AllowFISExperimentRoleEC2Actions",
            "Effect": "Allow",
            "Action": [
                "ec2:RebootInstances",
                "ec2:StopInstances",
                "ec2:StartInstances",
                "ec2:TerminateInstances"
            ],
            "Resource": "arn:aws:ec2:*:*:instance/*"
        },
        {
            "Sid": "AllowFISExperimentRoleECSActions",
            "Effect": "Allow",
            "Action": [
                "ecs:UpdateContainerInstancesState",
                "ecs:ListContainerInstances"
            ],
            "Resource": "arn:aws:ecs:*:*:container-instance/*"
        },
        {
            "Sid": "AllowFISExperimentRoleEKSActions",
            "Effect": "Allow",
            "Action": [
                "ec2:TerminateInstances"
            ],
            "Resource": "arn:aws:ec2:*:*:instance/*"
        },
        {
            "Sid": "AllowFISExperimentRoleFISActions",
            "Effect": "Allow",
            "Action": [
                "fis:InjectApiInternalError",
                "fis:InjectApiThrottleError",
                "fis:InjectApiUnavailableError"
            ],
            "Resource": "arn:*:fis:*:*:experiment/*"
        },
        {
            "Sid": "AllowFISExperimentRoleRDSReboot",
            "Effect": "Allow",
            "Action": [
                "rds:RebootDBInstance"
            ],
            "Resource": "arn:aws:rds:*:*:db:*"
        },
        {
            "Sid": "AllowFISExperimentRoleRDSFailOver",
            "Effect": "Allow",
            "Action": [
                "rds:FailoverDBCluster"
            ],
            "Resource": "arn:aws:rds:*:*:cluster:*"
        },
        {
            "Sid": "AllowFISExperimentRoleSSMSendCommand",
            "Effect": "Allow",
            "Action": [
                "ssm:SendCommand"
            ],
            "Resource": [
                "arn:aws:ec2:*:*:instance/*",
                "arn:aws:ssm:*:*:document/*"
            ]
        },
        {
            "Sid": "AllowFISExperimentRoleSSMCancelCommand",
            "Effect": "Allow",
            "Action": [
                "ssm:CancelCommand"
            ],
            "Resource": "*"
        }
    ]
}
```

3. Click on Next: Tags to move to the next screen, adding any Tags as you’d wish. In the Review Policy page, save this policy as `FisWorkshopServicePolicy` and add any description you would like. Complete the policy creation by clicking on Create Policy.

4. Navigate to the IAM console page and create a new Role.
  - On the “Select type of trusted entity” page AWS FIS does not exist as a trusted service yet. We shall add an account trust as a placeholder and replace this with AWS FIS later. Select “Another AWS Account” and add the current account number. You can find the AWS account number in the drop-down menu at the top right of the page as shown:
  ![fis-2](/images/fis-2.png)

5. Click on `Next: permissions`. On the “Attach permissions” page search for the FisWorkshopServicePolicy we just created and check the box beside it to attach it to the role.
  ![fis-3](/images/fis-3.png)

6. Click on Next: Tags and add any Tags you would like for this role.
7. Click on Next: Review and save the role name as FisWorkshopServiceRole. Add any description you would like for this role.
8. Complete the Role creation by clicking on Create role.
9. Back in the IAM Roles page, find and edit the FisWorkshopServiceRole. Select “Trust relationships” and the “Edit trust relationship” button.
  ![fis-4](/images/fis-4.png)

10. Replace the policy document with the following:
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Service": [
                  "fis.amazonaws.com"
                ]
            },
            "Action": "sts:AssumeRole",
            "Condition": {}
        }
    ]
}
```

11. Click on Update Trust Policy to complete updating the Role.


## Create Experiment Template

- Steady State:
  - The application should have at least 10 pods at all times
  - The application should always return Http 200 when a user browses to it's URL

- Hypothesis: 
  - If there's an outage in the region `eu-central-1` the `failover` mechanism will be activated and all traffic will be redirected to the application running on a cluster in `ap-southeast-1`

- Experiment:
  - In order to simulate a region outage we are going to terminate all instances of the EKS cluster.

1. Configure some variables for creating the FIS Experiment template configuration file
```
NODEGROUP_ARN=<NODEGROUP-ARN>
NODEGROUP_NAME=nodegroup
FIS_ROLE_ARN=<FIS-ROLE-ARN>
EXPERIMENT_TEMPLATE_NAME=sdp-chaos
```

2. Create Experiment template configuration file
```
tee -a ~/fis-experiment-template.json > /dev/null <<EOT
{
    "description": "Terminate a region by deleting all the nodes of the cluster located in eu-central-1",
    "targets": {
        "nodegroup": {
            "resourceType": "aws:eks:nodegroup",
            "resourceArns": [
                "${NODEGROUP_ARN}"
            ],
            "selectionMode": "ALL"
        }
    },
    "actions": {
        "delete-all-nodes": {
            "actionId": "aws:eks:terminate-nodegroup-instances",
            "description": "Deletes all the nodes of the cluster located in eu-central-1",
            "parameters": {
                "instanceTerminationPercentage": "100"
            },
            "targets": {
                "Nodegroups": "${NODEGROUP_NAME}"
            }
        }
    },
    "stopConditions": [
        {
            "source": "none"
        }
    ],
    "roleArn": "${FIS_ROLE_ARN}",
    "tags": {
        "Name": "${EXPERIMENT_TEMPLATE_NAME}"
    }
}
EOT
```

3. Create Experiment template
```
aws fis create-experiment-template --cli-input-json file://~/fis-experiment-template.json --region eu-central-1
```

4. Get Experiment template ID
```
aws fis list-experiment-templates --region eu-central-1
```

5. Run the experiment
```
aws fis start-experiment --region eu-central-1 --experiment-template-id EXPERIMENT_TEMPLATE_ID --tags Name=sdp-pacman-chaos-1 | jq '.experiment.id'
```

6. Using the returned id field you can check on the outcome of the experiment
```
aws fis get-experiment --region eu-central-1 --id YOUR_EXPERIMENT_ID_HERE
```