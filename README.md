# Setting Up a Complete CI/CD and Observability Lab Environment on AWS Using Terraform

This guide walks you through setting up a complete CI/CD and observability lab environment from scratch using AWS, Terraform, Jenkins, Prometheus, Grafana, and Kibana. The focus is on automation, cost-saving measures, and detailed monitoring.

## Prerequisites

- An AWS account
- AWS CLI installed and configured
- Terraform installed
- Jenkins installed
- Basic knowledge of AWS, Terraform, and CI/CD principles

## Step 1: Create an AWS Account

If you don't already have an AWS account, sign up at [AWS](https://aws.amazon.com/).

## Step 2: Install AWS CLI and Terraform

### Install AWS CLI

Follow the [AWS CLI installation guide](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html).

### Configure AWS CLI

```sh
aws configure
```

### Install Terraform

Download and install Terraform from the [Terraform website](https://www.terraform.io/downloads.html).

## Step 3: Terraform Configuration

### Create a Terraform Configuration File

Create a file named `main.tf` and add the following configuration:

```hcl
provider "aws" {
  region = "us-west-2"
}

# S3 Bucket for Terraform State
resource "aws_s3_bucket" "terraform_state" {
  bucket = "your-terraform-state-bucket"
  versioning {
    enabled = true
  }
}

# DynamoDB Table for State Locking
resource "aws_dynamodb_table" "terraform_locks" {
  name         = "your-terraform-lock-table"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }
}

terraform {
  backend "s3" {
    bucket         = "your-terraform-state-bucket"
    key            = "path/to/terraform.tfstate"
    region         = "us-west-2"
    dynamodb_table = "your-terraform-lock-table"
    encrypt        = true
  }
}

# IAM Role for EC2
resource "aws_iam_role" "ec2_role" {
  name = "ec2-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "ec2.amazonaws.com"
        }
      },
    ]
  })
}

resource "aws_iam_role_policy_attachment" "ec2_attach" {
  role       = aws_iam_role.ec2_role.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonEC2FullAccess"
}

resource "aws_iam_instance_profile" "ec2_instance_profile" {
  name = "ec2_instance_profile"
  role = aws_iam_role.ec2_role.name
}

# VPC and Networking Components
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
}

resource "aws_subnet" "subnet" {
  vpc_id     = aws_vpc.main.id
  cidr_block = "10.0.1.0/24"
}

resource "aws_internet_gateway" "gw" {
  vpc_id = aws_vpc.main.id
}

resource "aws_route_table" "routetable" {
  vpc_id = aws_vpc.main.id
}

resource "aws_route" "route" {
  route_table_id         = aws_route_table.routetable.id
  destination_cidr_block = "0.0.0.0/0"
  gateway_id             = aws_internet_gateway.gw.id
}

resource "aws_route_table_association" "a" {
  subnet_id      = aws_subnet.subnet.id
  route_table_id = aws_route_table.routetable.id
}

resource "aws_security_group" "allow_ssh" {
  vpc_id = aws_vpc.main.id

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

### Initialize Terraform

```sh
terraform init
```

### Apply the Terraform Configuration

```sh
terraform apply -auto-approve
```

## Step 4: Provision EC2 Instances

### Jenkins Instance

Add the following to `main.tf`:

```hcl
resource "aws_instance" "jenkins" {
  ami           = "ami-0c55b159cbfafe1f0" # Amazon Linux 2 AMI
  instance_type = "t2.micro"
  subnet_id     = aws_subnet.subnet.id
  iam_instance_profile = aws_iam_instance_profile.ec2_instance_profile.name
  security_groups = [aws_security_group.allow_ssh.name]

  instance_market_options {
    market_type = "spot"
  }

  user_data = <<-EOF
              #!/bin/bash
              yum update -y
              wget -O /etc/yum.repos.d/jenkins.repo \
                  https://pkg.jenkins.io/redhat-stable/jenkins.repo
              rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io.key
              yum install -y jenkins java-1.8.0-openjdk-devel
              systemctl start jenkins
              systemctl enable jenkins
              EOF

  tags = {
    Name = "jenkins-instance"
  }
}
```

### Grafana Instance

Add the following to `main.tf`:

```hcl
resource "aws_instance" "grafana" {
  ami           = "ami-0c55b159cbfafe1f0" # Amazon Linux 2 AMI
  instance_type = "t2.micro"
  subnet_id     = aws_subnet.subnet.id
  iam_instance_profile = aws_iam_instance_profile.ec2_instance_profile.name
  security_groups = [aws_security_group.allow_ssh.name]

  instance_market_options {
    market_type = "spot"
  }

  user_data = <<-EOF
              #!/bin/bash
              yum update -y
              yum install -y amazon-linux-extras
              amazon-linux-extras install epel -y
              amazon-linux-extras install nginx1.12 -y
              systemctl start nginx
              systemctl enable nginx

              # Install Grafana
              wget https://dl.grafana.com/oss/release/grafana-7.5.4-1.x86_64.rpm
              yum install -y grafana-7.5.4-1.x86_64.rpm
              systemctl start grafana-server
              systemctl enable grafana-server
              EOF

  tags = {
    Name = "grafana-instance"
  }
}
```

### Prometheus Instance

Add the following to `main.tf`:

```hcl
resource "aws_instance" "prometheus" {
  ami           = "ami-0c55b159cbfafe1f0" # Amazon Linux 2 AMI
  instance_type = "t2.micro"
  subnet_id     = aws_subnet.subnet.id
  iam_instance_profile = aws_iam_instance_profile.ec2_instance_profile.name
  security_groups = [aws_security_group.allow_ssh.name]

  instance_market_options {
    market_type = "spot"
  }

  user_data = <<-EOF
              #!/bin/bash
              yum update -y
              wget https://github.com/prometheus/prometheus/releases/download/v2.27.1/prometheus-2.27.1.linux-amd64.tar.gz
              tar xvf prometheus-2.27.1.linux-amd64.tar.gz
              cp prometheus-2.27.1.linux-amd64/prometheus /usr/local/bin/
              cp prometheus-2.27.1.linux-amd64/promtool /usr/local/bin/
              cp -r prometheus-2.27.1.linux-amd64/consoles /etc/prometheus/
              cp -r prometheus-2.27.1.linux-amd64/console_libraries /etc/prometheus/
              cp prometheus-2.27.1.linux-amd64/prometheus.yml /etc/prometheus/prometheus.yml
              useradd --no-create-home --shell /bin/false prometheus
              mkdir /var/lib/prometheus
              chown prometheus:prometheus /var/lib/prometheus
              cat <<EOT > /etc/systemd/system/prometheus.service
              [Unit]
              Description=Prometheus
              Wants=network-online.target
              After=network-online.target

              [Service]
              User=prometheus
              Group=prometheus
              Type=simple
              ExecStart=/usr/local/bin/prometheus \\
                --config.file /etc/prometheus/prometheus.yml \\
                --storage.tsdb.path /var/lib/prometheus/ \\
                --web.console.templates=/etc/prometheus/consoles \\
                --web.console.libraries=/etc/prometheus/console_libraries

              [Install]
              WantedBy=multi-user.target
              EOT
              systemctl daemon-reload
              systemctl start prometheus
              systemctl enable prometheus
              EOF

  tags

 = {
    Name = "prometheus-instance"
  }
}
```

### Kibana Instance

Add the following to `main.tf`:

```hcl
resource "aws_instance" "kibana" {
  ami           = "ami-0c55b159cbfafe1f0" # Amazon Linux 2 AMI
  instance_type = "t2.micro"
  subnet_id     = aws_subnet.subnet.id
  iam_instance_profile = aws_iam_instance_profile.ec2_instance_profile.name
  security_groups = [aws_security_group.allow_ssh.name]

  instance_market_options {
    market_type = "spot"
  }

  user_data = <<-EOF
              #!/bin/bash
              yum update -y
              rpm --import https://artifacts.elastic.co/GPG-KEY-elasticsearch
              cat <<EOT > /etc/yum.repos.d/kibana.repo
              [kibana-7.x]
              name=Kibana repository for 7.x packages
              baseurl=https://artifacts.elastic.co/packages/7.x/yum
              gpgcheck=1
              gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
              enabled=1
              autorefresh=1
              type=rpm-md
              EOT
              yum install -y kibana
              systemctl start kibana
              systemctl enable kibana
              EOF

  tags = {
    Name = "kibana-instance"
  }
}
```

### Apply the Terraform Configuration Part 2

```sh
terraform apply -auto-approve
```

## Step 5: Configure Jenkins Pipeline

### Create a Jenkinsfile

Create a file named `Jenkinsfile` in your repository:

```groovy
pipeline {
    agent {
        label 'your-agent-label'
    }

    environment {
        AWS_REGION = 'us-west-2'
        AWS_ACCESS_KEY_ID = credentials('aws-access-key')
        AWS_SECRET_ACCESS_KEY = credentials('aws-secret-key')
    }

    stages {
        stage('Checkout') {
            steps {
                git 'https://your-repo-url.git'
            }
        }
        stage('Retrieve Secrets') {
            steps {
                script {
                    def secrets = sh(script: '''
                        aws secretsmanager get-secret-value --secret-id observability-stack-secrets --region $AWS_REGION --query SecretString --output text
                    ''', returnStdout: true).trim()
                    def parsedSecrets = readJSON text: secrets
                    env.GRAFANA_ADMIN_PASSWORD = parsedSecrets.grafana_admin_password
                    env.PROMETHEUS_DB_PASSWORD = parsedSecrets.prometheus_db_password
                    env.KIBANA_SECRET_KEY = parsedSecrets.kibana_secret_key
                }
            }
        }
        stage('Terraform Init') {
            steps {
                sh 'terraform init'
            }
        }
        stage('Terraform Plan') {
            steps {
                sh 'terraform plan'
            }
        }
        stage('Terraform Apply') {
            steps {
                sh 'terraform apply -auto-approve'
            }
        }
        stage('Test') {
            steps {
                // Add your tests here
            }
        }
    }

    post {
        always {
            script {
                def jobDuration = currentBuild.durationString.replace(' and counting', '')
                def buildStatus = currentBuild.result ?: 'SUCCESS'
                def metrics = [
                    "job_duration_seconds{job=\"${env.JOB_NAME}\",status=\"${buildStatus}\"} ${currentBuild.duration / 1000}",
                    "job_failure_rate{job=\"${env.JOB_NAME}\"} ${buildStatus == 'SUCCESS' ? 0 : 1}"
                ]
                writeFile file: 'metrics.prom', text: metrics.join('\n')
                sh 'curl --data-binary @metrics.prom http://<prometheus-pushgateway-url>/metrics/job/${env.JOB_NAME}'
            }
        }
    }
}
```

### Configure Jenkins Pipeline Job

1. **Access Jenkins**:
   - Open a browser and navigate to the public IP address of the Jenkins EC2 instance, port 8080 (e.g., `http://<jenkins-public-ip>:8080`).

2. **Install Required Plugins**:
   - Go to Jenkins Dashboard > Manage Jenkins > Manage Plugins.
   - Install the "Git" plugin to clone your repository.
   - Install the "Pipeline" plugin to support Jenkinsfile pipelines.

3. **Create a New Pipeline Job**:
   - Go to Jenkins Dashboard.
   - Click on "New Item" and select "Pipeline".
   - Name your job and configure it.
   - In the "Pipeline" section, set the "Definition" to "Pipeline script from SCM".
   - Set "SCM" to "Git" and provide your repository URL.

4. **Add AWS Credentials**:
   - Go to "Manage Jenkins" > "Manage Credentials".
   - Add your AWS access key ID and secret access key as a new "Secret Text" credential.
   - Use the ID `aws-access-key` and `aws-secret-key` in your Jenkinsfile.

## Step 6: Monitoring and Cost Management

### Set Up CloudWatch Exporter

Deploy the CloudWatch Exporter to pull cost metrics from AWS CloudWatch.

### Example CloudWatch Exporter Configuration (`cloudwatch_exporter_config.yml`)

```yaml
region: us-west-2
metrics:
  - aws_namespace: AWS/Billing
    aws_metric_name: EstimatedCharges
    aws_dimensions: [Currency]
    aws_statistics: [Maximum]
    period_seconds: 86400
    range_seconds: 2592000
    delay_seconds: 86400
    aws_dimension_select:
      Currency: [USD]
```

### Deploy CloudWatch Exporter

Deploy the CloudWatch Exporter on an EC2 instance or as a Docker container.

### Prometheus Configuration to Scrape CloudWatch Exporter

Update Prometheus configuration (`prometheus.yml`):

```yaml
scrape_configs:
  - job_name: 'cloudwatch_exporter'
    static_configs:
      - targets: ['<cloudwatch_exporter_host>:9106']
```

### Create Grafana Dashboard for Cost Tracking

1. **Add Prometheus as a Data Source**:
   - Configure Grafana to use Prometheus as a data source.

2. **Create Dashboard Panels**:
   - **Panel 1**: Total AWS Cost
     - Query: `aws_billing_estimatedcharges_maximum{currency="USD"}`
     - Visualization: Single Stat

   - **Panel 2**: Cost by Service
     - Query: `aws_billing_estimatedcharges_maximum{currency="USD"}`
     - Visualization: Bar Chart
     - Group by: Service

### Monitoring CI/CD Pipeline Metrics

1. **Jenkins Metrics**:
   - Use the Prometheus plugin for Jenkins to expose CI/CD metrics.
   - Add Prometheus scrape configuration for Jenkins metrics.

### Visualizing CI/CD Metrics in Grafana

1. **Add Prometheus as a Data Source**:
   - Configure Grafana to use Prometheus as a data source.

2. **Create Dashboards for CI/CD Metrics**:
   - Create dashboards to track job duration, failure rates, and other relevant metrics.

### Example Dashboard Panels

- **Panel 1**: Job Duration
  - Query: `job_duration_seconds`
  - Visualization: Line Chart

- **Panel 2**: Job Failure Rate
  - Query: `job_failure_rate`
  - Visualization: Bar Chart

By following these steps, you can set up a comprehensive CI/CD and observability lab environment on AWS. This guide ensures that you automate as much as possible using Terraform and the command line, and track costs and performance metrics using Prometheus and Grafana.
