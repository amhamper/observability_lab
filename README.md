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
# Specify the AWS provider and region
provider "aws" {
  region = "us-west-2"
}

# Create an S3 bucket to store Terraform state
resource "aws_s3_bucket" "terraform_state" {
  bucket = "your-terraform-state-bucket"
  versioning {
    enabled = true
  }
}

# Create a DynamoDB table for state locking to prevent concurrent modifications
resource "aws_dynamodb_table" "terraform_locks" {
  name         = "your-terraform-lock-table"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }
}

# Configure Terraform backend to use S3 and DynamoDB
terraform {
  backend "s3" {
    bucket         = "your-terraform-state-bucket"
    key            = "path/to/terraform.tfstate"
    region         = "us-west-2"
    dynamodb_table = "your-terraform-lock-table"
    encrypt        = true
  }
}

# Create an IAM role for EC2 instances
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

# Attach the necessary policy to the IAM role
resource "aws_iam_role_policy_attachment" "ec2_attach" {
  role       = aws_iam_role.ec2_role.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonEC2FullAccess"
}

# Create an instance profile for the IAM role
resource "aws_iam_instance_profile" "ec2_instance_profile" {
  name = "ec2_instance_profile"
  role = aws_iam_role.ec2_role.name
}

# Create a VPC
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
}

# Create a subnet within the VPC
resource "aws_subnet" "subnet" {
  vpc_id     = aws_vpc.main.id
  cidr_block = "10.0.1.0/24"
}

# Create an internet gateway for the VPC
resource "aws_internet_gateway" "gw" {
  vpc_id = aws_vpc.main.id
}

# Create a route table for the VPC
resource "aws_route_table" "routetable" {
  vpc_id = aws_vpc.main.id
}

# Create a route in the route table
resource "aws_route" "route" {
  route_table_id         = aws_route_table.routetable.id
  destination_cidr_block = "0.0.0.0/0"
  gateway_id             = aws_internet_gateway.gw.id
}

# Associate the route table with the subnet
resource "aws_route_table_association" "a" {
  subnet_id      = aws_subnet.subnet.id
  route_table_id = aws_route_table.routetable.id
}

# Create a security group to allow SSH access
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
# Create an EC2 instance for Jenkins
resource "aws_instance" "jenkins" {
  ami           = "ami-0c55b159cbfafe1f0" # Amazon Linux 2 AMI
  instance_type = "t2.micro"
  subnet_id     = aws_subnet.subnet.id
  iam_instance_profile = aws_iam_instance_profile.ec2_instance_profile.name
  security_groups = [aws_security_group.allow_ssh.name]

  # Use spot instances to reduce costs
  instance_market_options {
    market_type = "spot"
  }

  # User data to install Jenkins
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
# Create an EC2 instance for Grafana
resource "aws_instance" "grafana" {
  ami           = "ami-0c55b159cbfafe1f0" # Amazon Linux 2 AMI
  instance_type = "t2.micro"
  subnet_id     = aws_subnet.subnet.id
  iam_instance_profile = aws_iam_instance_profile.ec2_instance_profile.name
  security_groups = [aws_security_group.allow_ssh.name]

  # Use spot instances to reduce costs
  instance_market_options {
    market_type = "spot"
  }

  # User data to install Grafana
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
# Create an EC2 instance for Prometheus
resource "aws_instance" "prometheus" {
  ami           = "ami-0c55b159cbfafe1f0" # Amazon Linux 2 AMI
  instance_type = "t2.micro"
  subnet_id     = aws_subnet.subnet.id
  iam_instance_profile = aws_iam_instance_profile.ec2_instance_profile.name
  security_groups = [aws_security_group.allow_ssh.name]

  # Use spot instances to reduce costs
  instance_market_options {
    market_type = "spot"
  }

  # User data to install Prometheus
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
              mkdir /var/lib

/prometheus
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

  tags = {
    Name = "prometheus-instance"
  }
}
```

### Kibana Instance

Add the following to `main.tf`:

```hcl
# Create an EC2 instance for Kibana
resource "aws_instance" "kibana" {
  ami           = "ami-0c55b159cbfafe1f0" # Amazon Linux 2 AMI
  instance_type = "t2.micro"
  subnet_id     = aws_subnet.subnet.id
  iam_instance_profile = aws_iam_instance_profile.ec2_instance_profile.name
  security_groups = [aws_security_group.allow_ssh.name]

  # Use spot instances to reduce costs
  instance_market_options {
    market_type = "spot"
  }

  # User data to install Kibana
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

### Apply the Terraform Configuration AGAIN ugh

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
  - job_name: 'node_exporter'
    ec2_sd_configs:
      - region: us-west-2
        access_key: '<AWS_ACCESS_KEY>'
        secret_key: '<AWS_SECRET_KEY>'
    relabel_configs:
      - source_labels: [__meta_ec2_tag_Name]
        regex: node-exporter-instance-.*
        action: keep
  - job_name: 'cloudwatch_exporter'
    static_configs:
      - targets: ['<cloudwatch_exporter_host>:9106']
  - job_name: 'jenkins'
    static_configs:
      - targets: ['<jenkins_host>:<jenkins_metrics_port>']
```

### Create Grafana Dashboard for Cost Tracking

1. **Add Prometheus as a Data Source**:
   - Go to Grafana > Configuration > Data Sources > Add data source.
   - Select Prometheus and configure it with your Prometheus server URL.

2. **Create Dashboards and Panels**:

#### Infrastructure Metrics Dashboard

- **Panel 1**: CPU Usage
  - Query: `avg(rate(node_cpu_seconds_total{mode!="idle"}[5m])) by (instance)`
  - Visualization: Line Chart

- **Panel 2**: Memory Usage
  - Query: `node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes * 100`
  - Visualization: Line Chart

- **Panel 3**: Disk I/O
  - Query: `rate(node_disk_io_time_seconds_total[5m])`
  - Visualization: Bar Chart

- **Panel 4**: Network Traffic
  - Query: `rate(node_network_receive_bytes_total[5m])` and `rate(node_network_transmit_bytes_total[5m])`
  - Visualization: Line Chart

#### Application Metrics Dashboard

- **Panel 1**: Application Response Time
  - Query: `histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))`
  - Visualization: Heatmap

- **Panel 2**: Error Rates
  - Query: `sum(rate(http_requests_total{status=~"5.."}[5m])) by (instance)`
  - Visualization: Bar Chart

- **Panel 3**: Request Rates
  - Query: `sum(rate(http_requests_total[5m])) by (instance)`
  - Visualization: Line Chart

#### CI/CD Metrics Dashboard

- **Panel 1**: Job Duration
  - Query: `avg(jenkins_job_duration_seconds) by (job)`
  - Visualization: Line Chart

- **Panel 2**: Job Failure Rates
  - Query: `sum(jenkins_job_last_build_result == 2) by (job)`
  - Visualization: Bar Chart

- **Panel 3**: Time to Resolve Failed Jobs
  - Query: `avg(time() - jenkins_job_last_build_timestamp_seconds) by (job)`
  - Visualization: Heatmap

- **Panel 4**: Deployment Frequency
  - Query: `count(jenkins_job_last_build_result == 0) by (job)`
  - Visualization: Line Chart

#### Cost Metrics Dashboard

- **Panel 1**: Total AWS Cost
  - Query: `aws_billing_estimatedcharges_maximum{currency="USD"}`
  - Visualization: Single Stat

- **Panel 2**: Cost by Service
  - Query: `aws_billing_estimatedcharges_maximum{currency="USD"}`
  - Visualization: Bar Chart
  - Group by: Service

#### Logs Dashboard

- **Panel 1**: System Logs
  - Query: Logs from `node_exporter`
  - Visualization: Logs

- **Panel 2**: Application Logs
  - Query: Logs from your application
  - Visualization: Logs

- **Panel 3**: CI/CD Pipeline Logs
  - Query: Jenkins pipeline logs
  - Visualization: Logs

By following these steps, you can set up comprehensive monitoring and logging for your infrastructure. This setup will provide valuable insights into the performance and health of your systems and applications, helping you identify and resolve issues more efficiently
