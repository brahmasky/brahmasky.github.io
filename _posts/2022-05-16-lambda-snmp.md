## Overview

In a recent project we deployed a few appliances into AWS workspaces with EC2 service. As per the organisation's monitoring requirment, the appliances need to be monitored to provide metrics from infrastructure level including memory utilisation and disk space usage which are not covered by AWS Cloudwatch out of box.

One option is to use [CloudWatch Agent](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/mon-scripts.html), unfortunatelly the vendor lock out the shell access and does not allow any root access to the OS without its approval. So we have to rely on the SNMP service provided by the appliances to perform the monitoring with Lambda function


## Environment 
To implement the SNMP monitoring with the least priveleges principal in mind, we need to have the proper environment setup with includes IAM roles and VPC configuration.

### IAM role
The snmp monitoring role has been granted with previlege to access specifi secrets in Secrets Manager, which is the credential to authenticate against the SNMP v3 service. Some cloudwatch permissions have been added as well so we can post the result to CloudWatch.
```yaml
          # to allow lambda function to access SNMP v3 credentials stored in secrets manager
          - secretsmanager:ListSecrets
          - secretsmanager:GetResourcePolicy
          - secretsmanager:GetSecretValue
          - secretsmanager:DescribeSecret
          - secretsmanager:ListSecretVersionIds
          # to allow lamda function to create and post metrics to cloud watch
          - cloudwatch:Describe*
          - cloudwatch:Get*
          - cloudwatch:List*
          - cloudwatch:PutMetricData
```

### VPC
As the appliances is deployed in private VPC and only open to internal network, Lambda function has to have VPC enabled in specific subnet with NSG rules to allow SNMP polling from Lambda to target appliance.
```yaml
      SecurityGroupIngress:
      - IpProtocol: udp
        FromPort: 161
        ToPort: 161
        SourceSecurityGroupId: !Ref mySg
        Description: Allow SNMP 161 from Lambda function located in mySg subnet
```

### Runtime and library
In this task Python is chosen to be the runtime for the lambda function and [PySNMP](https://pypi.org/project/pysnmp/) module are imported to perform the SNMP polling against the appliance.

