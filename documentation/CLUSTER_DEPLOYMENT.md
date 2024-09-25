# Déploiement des clusters

## Clusters managés

Sur demo.redhat.com, je commande l'asset **Red Hat Trusted Application Pipeline** avec les paramètres :

- Developer Hub : stable
- Auto-stop: no auto-stop
- Auto-destroy: +10 jours

## Environnement AWS

Sur demo.redhat.com, je commande l'asset **AWS Blank Environment**.

Je configure la cli AWS :

```sh
export AWS_CONFIG_FILE=$HOME/tmp/summit-connect-2024/aws/.aws/config
export AWS_SHARED_CREDENTIALS_FILE=$HOME/tmp/summit-connect-2024/aws/.aws/credentials
mkdir -p $HOME/tmp/summit-connect-2024/aws/.aws/
aws configure
```

## Déploiement d'un nouveau cluster pour OpenShift AI dans la région de Francfort

```
$ mkdir -p ~/tmp/summit-connect-2024/dev-clusters/llm
$ cd ~/tmp/summit-connect-2024/dev-clusters/llm
```

Fichier **install-config.yaml** :

```yaml
additionalTrustBundlePolicy: Proxyonly
apiVersion: v1
baseDomain: sandbox2463.opentlc.com
compute:
- architecture: amd64
  hyperthreading: Enabled
  name: worker
  platform:
    aws:
      type: m5a.2xlarge
      zones:
      - eu-central-1a
      - eu-central-1b
      - eu-central-1c
  replicas: 3
controlPlane:
  architecture: amd64
  hyperthreading: Enabled
  name: master
  platform:
    aws:
      rootVolume:
        iops: 4000
        size: 500
        type: io1 
      type: m5a.xlarge
      zones:
      - eu-central-1a
      - eu-central-1b
      - eu-central-1c
  replicas: 3
metadata:
  creationTimestamp: null
  name: llm
networking:
  clusterNetwork:
  - cidr: 10.128.0.0/14
    hostPrefix: 23
  machineNetwork:
  - cidr: 10.0.0.0/16
  networkType: OVNKubernetes
  serviceNetwork:
  - 172.30.0.0/16
platform:
  aws:
    region: eu-central-1
publish: External
pullSecret: 'REDACTED'
sshKey: |
  ssh-ed25519 REDACTED
```

Lancer l'installation du cluster.

```sh
openshift-install create cluster --dir . --log-level=info
```

Résultat :

```
INFO Credentials loaded from the "default" profile in file "/home/nmasse/.aws/credentials" 
INFO Consuming Install Config from target directory 
INFO Creating infrastructure resources...         
INFO Waiting up to 20m0s (until 2:52PM CEST) for the Kubernetes API at https://api.llm.sandbox2463.opentlc.com:6443... 
INFO API v1.28.6+6216ea1 up                       
INFO Waiting up to 30m0s (until 3:05PM CEST) for bootstrapping to complete... 
INFO Destroying the bootstrap resources...        
INFO Waiting up to 40m0s (until 3:30PM CEST) for the cluster at https://api.llm.sandbox2463.opentlc.com:6443 to initialize... 
INFO Waiting up to 30m0s (until 3:33PM CEST) to ensure each cluster operator has finished progressing... 
INFO All cluster operators have completed progressing 
INFO Checking to see if there is a route at openshift-console/console... 
INFO Install complete!                            
INFO To access the cluster as the system:admin user when using 'oc', run 'export KUBECONFIG=/home/nmasse/tmp/summit-connect-2024/dev-clusters/llm/auth/kubeconfig' 
INFO Access the OpenShift web-console here: https://console-openshift-console.apps.llm.sandbox2463.opentlc.com 
INFO Login to the console with user: "kubeadmin", and password: "REDACTED" 
INFO Time elapsed: 41m17s                         
```

## Auto-start / auto-stop des clusters OpenShift

Module commun (instancié une fois) :

```terraform
resource "aws_iam_policy" "scheduled_start_stop" {
  name   = "Scheduled-Start-Stop"
  policy = jsonencode({
    Version   = "2012-10-17"
    Statement = [
      {
        Effect   = "Allow"
        Action   = [
          "logs:CreateLogGroup",
          "logs:CreateLogStream",
          "logs:PutLogEvents"
        ]
        Resource = "arn:aws:logs:*:*:*"
      },
      {
        Effect   = "Allow"
        Action   = [
          "ec2:DescribeInstances",
          "ec2:StopInstances",
          "ec2:StartInstances"
        ]
        Resource = "*"
      }
    ]
  })
}

resource "aws_iam_role" "lambda_execution" {
  name = "Scheduled-Start-Stop"

  assume_role_policy = jsonencode({
    Version   = "2012-10-17"
    Statement = [{
      Action    = "sts:AssumeRole"
      Effect    = "Allow"
      Principal = { Service = "lambda.amazonaws.com" }
    }]
  })
}
```

Module à instancier par région :

```terraform
variable "region" {
  default = "eu-central-1"
}

provider "aws" {
  region = var.region
}

data "aws_iam_policy" "scheduled_start_stop" {
  name   = "Scheduled-Start-Stop"
}

data "aws_iam_role" "lambda_execution" {
  name = "Scheduled-Start-Stop"
}

resource "aws_iam_role_policy_attachment" "lambda_attach" {
  role       = data.aws_iam_role.lambda_execution.name
  policy_arn = data.aws_iam_policy.scheduled_start_stop.arn
}

data "archive_file" "stop_ec2_instances_zip" {
  type        = "zip"
  output_path = "${path.module}/stop.zip"
  source_content_filename = "lambda_function.py"
  source_content = <<-EOF
import boto3

region = '${var.region}'
ec2 = boto3.client('ec2', region_name=region)

def lambda_handler(event, context):
    filters = [
        {'Name': 'instance-state-name', 'Values': ['running']}
    ]
    response = ec2.describe_instances(Filters=filters)
    instances = [instance['InstanceId'] for reservation in response['Reservations'] for instance in reservation['Instances']]
    if instances:
        ec2.stop_instances(InstanceIds=instances)
        print('Stopped your instances: ' + str(instances))
    else:
        print('No instances found matching the criteria.')
  EOF
}

data "archive_file" "start_ec2_instances_zip" {
  type        = "zip"
  output_path = "${path.module}/start.zip"
  source_content_filename = "lambda_function.py"
  source_content = <<-EOF
import boto3

region = '${var.region}'
ec2 = boto3.client('ec2', region_name=region)

def lambda_handler(event, context):
    filters = [
        {'Name': 'instance-state-name', 'Values': ['stopped']}
    ]
    response = ec2.describe_instances(Filters=filters)
    instances = [instance['InstanceId'] for reservation in response['Reservations'] for instance in reservation['Instances']]
    if instances:
        ec2.start_instances(InstanceIds=instances)
        print('Started your instances: ' + str(instances))
    else:
        print('No instances found matching the criteria.')
  EOF
}

resource "aws_lambda_function" "stop_ec2_instances" {
  function_name    = "StopEC2Instances"
  handler          = "lambda_function.lambda_handler"
  role             = data.aws_iam_role.lambda_execution.arn
  runtime          = "python3.8"
  timeout          = 10
  filename         = data.archive_file.stop_ec2_instances_zip.output_path
  source_code_hash = data.archive_file.stop_ec2_instances_zip.output_base64sha256
}

resource "aws_lambda_function" "start_ec2_instances" {
  function_name    = "StartEC2Instances"
  handler          = "lambda_function.lambda_handler"
  role             = data.aws_iam_role.lambda_execution.arn
  runtime          = "python3.8"
  timeout          = 10
  filename         = data.archive_file.start_ec2_instances_zip.output_path
  source_code_hash = data.archive_file.start_ec2_instances_zip.output_base64sha256
}

resource "aws_cloudwatch_event_rule" "stop_ec2_instances_schedule" {
  name                = "StopEC2Instances"
  schedule_expression = "cron(30 16 * * ? *)" // UTC
  
}

resource "aws_cloudwatch_event_target" "stop_ec2_instances_target" {
  rule      = aws_cloudwatch_event_rule.stop_ec2_instances_schedule.name
  arn       = aws_lambda_function.stop_ec2_instances.arn
}

resource "aws_cloudwatch_event_rule" "start_ec2_instances_schedule" {
  name                = "StartEC2Instances"
  schedule_expression = "cron(30 6 ? * MON,TUE,WED,THU,FRI *)" // UTC
}

resource "aws_cloudwatch_event_target" "start_ec2_instances_target" {
  rule      = aws_cloudwatch_event_rule.start_ec2_instances_schedule.name
  arn       = aws_lambda_function.start_ec2_instances.arn
}

resource "aws_lambda_permission" "stop_ec2_instances_invoke" {
  statement_id  = "AllowExecutionFromCloudWatch"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.stop_ec2_instances.function_name
  principal     = "events.amazonaws.com"
  source_arn    = aws_cloudwatch_event_rule.stop_ec2_instances_schedule.arn
}

resource "aws_lambda_permission" "start_ec2_instances_invoke" {
  statement_id  = "AllowExecutionFromCloudWatch"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.start_ec2_instances.function_name
  principal     = "events.amazonaws.com"
  source_arn    = aws_cloudwatch_event_rule.start_ec2_instances_schedule.arn
}
```

Déploiement.

```sh
terraform init
terraform apply
```

Au final, le cluster tourne 50 heures par semaine, c'est à dire 30% du temps.

