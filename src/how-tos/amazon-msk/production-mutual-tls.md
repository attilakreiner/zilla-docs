---
icon: aky-zilla-plus
description: Setup mutual authentication to your MSK cluster from anywhere on the internet.
---

# Production (Mutual TLS)

[Available in Zilla Plus<sup>+</sup>](https://www.aklivity.io/products/zilla-plus)
{.zilla-plus-badge .hint-container .info}

::: tip Estimated time to complete 20-30 minutes.
:::

## Overview

The [Zilla Plus (Public MSK Proxy)](https://aws.amazon.com/marketplace/pp/prodview-jshnzslazfm44) lets authorized Kafka clients connect, publish messages and subscribe to topics in your Amazon MSK cluster via the internet.

In this guide we will deploy the Zilla Plus (Public MSK Proxy) and showcase globally trusted public internet connectivity to an MSK cluster from a Kafka client, using the custom wildcard domain `*.example.aklivity.io`. Kafka clients will use TLS client certificates to verify trusted client identity.

### AWS services used

| Service                     | Required                                                                       | Usage        | Quota                                                                                        |
| --------------------------- | ------------------------------------------------------------------------------ | ------------ | -------------------------------------------------------------------------------------------- |
| Resource Groups and Tagging | Yes                                                                            | Startup only | [None](https://docs.aws.amazon.com/general/latest/gr/arg.html#arg-quotas)                    |
| Secrets Manager             | Yes                                                                            | Startup only | [Not reached](https://docs.aws.amazon.com/general/latest/gr/asm.html#limits_secrets-manager) |
| Certificate Manager         | No<br><br>Private key and certificate can be inline in Secrets Manager instead | Startup only | [Not reached](https://docs.aws.amazon.com/general/latest/gr/acm.html#limits_acm)             |
| Private Certificate Manager | No<br><br>Private key and certificate can be inline in Secrets Manager instead | Startup only | [Not reached](https://docs.aws.amazon.com/general/latest/gr/acm-pca.html#limits_acm-pca)     |

Default [AWS Service Quotas](https://docs.aws.amazon.com/general/latest/gr/aws_service_limits.html) are recommended.

## Prerequisites

Before setting up internet access to your MSK Cluster, you will need the following:

- an MSK Cluster configured for TLS encrypted client access and TLS client authentication
- an VPC security group for MSK Proxy instances
- an IAM security role for MSK Proxy instances
- subscription to Zilla Plus (Public MSK Proxy) via AWS Marketplace
- permission to modify global DNS records for a custom domain
- permission to generate client certificates signed by a private certificate authority

::: tip
Check out the [Troubleshooting](../../reference/troubleshooting/amazon-msk.md) guide if you run into any issues.
:::

### Create Certificate Authority (ACM) for mTLS

> This creates a new private certificate authority in ACM.

Follow the [Create Certificate Authority (ACM)](../../reference/amazon-msk/create-certificate-authority-acm.md) to create a private certificate authority to verify TLS client authentication.

- Distinguished Name
  ::: code-tabs

  @tab Common Name (CN)

  ```text:no-line-numbers
  Mutual Authentication CA
  ```

  :::

### Create the MSK Cluster

> This creates your MSK cluster in preparation for secure access via the internet.

An MSK cluster is needed for secure remote access via the internet. You can skip this step if you have already created an MSK cluster with equivalent configuration.

Follow the [Create MSK Cluster](../../reference/amazon-msk/create-msk-cluster.md) guide to setup the a new MSK cluster. We will use the bellow resource names to reference the AWS resources needed in this guide.

- Cluster Name: `my-msk-cluster`
- Access control methods: `TLS client certificates`
- AWS Private CAs: `Mutual Authentication CA`
- VPC: `my-msk-cluster-vpc`
- Subnet: `my-msk-cluster-subnet-*`
- Route tables: `my-msk-cluster-rtb-*`
- Internet gateway: `my-msk-cluster-igw`

### Create Client Certificate (ACM) for mTLS

This allows an authorized Kafka client to connect directly to your MSK cluster with Mutual TLS (mTLS). Follow the [Create Client Certificate (ACM)](../../reference/amazon-msk/create-client-certificate-acm.md) to create a private certificate authority.

You can create additional client certificates for each different authorized client identity that will connect via the internet to your MSK Public Proxy deployment.

[Update the security settings](https://docs.aws.amazon.com/msk/latest/developerguide/msk-update-security.html) of your MSK cluster. Update your Access control methods to include

Common Name: `client-1`\
Private Certificate Authority: `Mutual Authentication CA`

### Create the MSK Proxy security group

> This creates your Public MSK proxy security group to allow Kafka clients and SSH access.

A VPC security group is needed for the Public MSK Proxy instances when they are launched.

Follow the [Create Security Group](https://docs.aws.amazon.com/vpc/latest/userguide/security-groups.html#creating-security-groups) docs with the following parameters and defaults. This creates your MSK proxy security group to allow Kafka clients and SSH access.

- VPC: `my-msk-cluster-vpc`
- Name: `my-msk-proxy-sg`
- Description: `Kafka clients and SSH access`
- Add Inbound Rule
  - Type: `CUSTOM TCP`
  - Port Range: `9094`
  - Source type: `Anywhere-IPv4`
- Add Inbound Rule
  - Type: `SSH`
  - Source type: `My IP`

### Update the default security group rules

> This allows the MSK Proxy instances to communicate with your MSK cluster.

Navigate to the VPC Management Console [Security Groups](https://console.aws.amazon.com/vpc/home#securityGroups:) table.

::: note Check your selected region
Make sure you have selected the desired region, such as `US East (N. Virginia) us-east-1`.
:::

Filter the security groups by selecting a `VPC` and select the `default` security group.

- VPC: `my-msk-cluster-vpc`
- Security Group: `default`

#### Add a Custom TCP Rule

Add this Inbound Rule to allow the MSK Proxy instances to communicate with the MSK cluster.

- Type: `Custom TCP`
- Port Range: `9094`
- Source type: `Custom`
- Source: `my-msk-proxy-sg`

### Create the MSK Proxy IAM security role

> This creates an IAM security role to enable the required AWS services for the MSK Proxy instances.

Follow the [Create IAM Role](../../reference/amazon-msk/create-iam-role.md) guide to create an IAM security role with the following parameters:

::: code-tabs

@tab Name

```text:no-line-numbers
aklivity-public-msk-proxy
```

@tab Policies

```text:no-line-numbers
AWSMarketplaceMeteringFullAccess
AWSCertificateManagerReadOnly
AWSCertificateManagerPrivateCAReadOnly
ResourceGroupsandTagEditorReadOnlyAccess
```

:::

#### IAM role Inline Policies

::: code-tabs

@tab Name

```text:no-line-numbers
MSKProxySecretsManagerRead
```

@tab JSON Summary

```json:no-line-numbers
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "VisualEditor0",
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue",
        "secretsmanager:DescribeSecret"
      ],
      "Resource": [
        "arn:aws:secretsmanager:*:*:secret:wildcard.example.aklivity.io-*",
        "arn:aws:secretsmanager:*:*:secret:client-*"
      ]
    }
  ]
}
```

:::

::: note
This example pattern requires all trusted client certificate key secrets to be named `client-*`.
:::

::: info If you used a different secret name for your certificate key.

Replace `wildcard.example.aklivity.io` in the resource regular expression for:

```text:no-line-numbers
MSKProxySecretsManagerRead
```

:::

## Subscribe via AWS Marketplace

The [Zilla Plus (Public MSK Proxy)](https://aws.amazon.com/marketplace/pp/prodview-jshnzslazfm44) is available through the AWS Marketplace. You can skip this step if you have already subscribed to Zilla Plus (Public MSK Proxy) via AWS Marketplace.

To get started, visit the Proxy's Marketplace [Product Page](https://aws.amazon.com/marketplace/pp/prodview-jshnzslazfm44) and `Subscribe` to the offering. You should now see `Zilla Plus (Public MSK Proxy)` listed in your [AWS Marketplace](https://console.aws.amazon.com/marketplace) subscriptions.

## Create the Public TLS Server Certificate

We need a Public TLS Server Certificate for your custom DNS wildcard domain that can be trusted by a Kafka Client from anywhere.

Follow the [Create Server Certificate (LetsEncrypt)](../../reference/amazon-msk/create-server-certificate-letsencrypt.md) guide to create a new TLS Server Certificate. Use your own custom wildcard DNS domain in place of the example wildcard domain `*.example.aklivity.io`.

::: info
Note the server certificate secret ARN as we will need to reference it from the Public MSK Proxy CloudFormation template.
:::

## Deploy the Public MSK Proxy

> This initiates deployment of the Zilla Plus (Public MSK Proxy) (Mutual TLS) stack via CloudFormation.

Navigate to your [AWS Marketplace](https://console.aws.amazon.com/marketplace) subscriptions and select `Zilla Plus (Public MSK Proxy)` to show the manage subscription page.

- From the `Agreement` section > `Actions` menu > select `Launch CloudFormation stack`
- Select the `Public MSK Proxy (Mutual TLS)` fulfillment option
- Make sure you have selected the desired region selected, such as `us-east-1`
- Click `Continue to Launch`
  - Choose the action `Launch CloudFormation`

Click `Launch` to complete the `Create stack` wizard with the following details:

### Step 1. Create Stack

- Prepare template: `Template is ready`
- Specify template: `Amazon S3 URL`
  - Amazon S3 URL: `(auto-filled)`

### Step 2. Specify stack details

::: code-tabs

@tab Stack name

```text:no-line-numbers
my-public-msk-proxy
```

:::

Parameters:

- Network Configuration
  - VPC: `my-msk-cluster-vpc`
  - Subnets: `my-msk-cluster-1a` `my-msk-cluster-1b` `my-msk-cluster-1c`
- MSK Configuration
  - Wildcard DNS pattern: `*.aklivity.[...].amazonaws.com` *1
  - Port number: `9094`
  - Private Certificate Authority: `<private certificate authority ARN>` *2a
- MSK Proxy Configuration
  - Instance count: `2`
  - Instance type: `t3.small` *3
  - Role: `aklivity-public-msk-proxy`
  - Security Groups: `my-msk-proxy`
  - Secrets Manager Secret ARN: `<TLS certificate private key secret ARN>` *3
  - Public Wildcard DNS: `*.example.aklivity.io` *4
  - Public Port: `9094`
  - Private Certificate Authority: `<private certificate authority ARN>` *2
  - Key pair for SSH access: `my-key-pair` *5
- *Configuration Reference
  1. Follow the [Lookup MSK Server Names](../../reference/amazon-msk/lookup-msk-server-names.md) guide to discover the wildcard DNS pattern for your MSK cluster.
  2. These can be the same Private Certificate Authority that authorizes existing clients connecting directly to MSK, allowing existing trusted client certificates to connect via Public MSK Proxy.
  3. Consider the network throughput characteristics of the AWS instance type as that will impact the upper bound on network performance.
  4. Replace with your own custom wildcard DNS pattern.
  5. Follow the [Create Key Pair](../../reference/amazon-msk/create-key-pair.md) guide to create a new key pair used when launching EC2 instances with SSH access.

### Step 3. Configure stack options: `(use defaults)`

### Step 4. Review

Confirm the stack details are correct and `Submit` to start the CloudFormation deploy.

::: info
When your Public MSK Proxy is ready, the [CloudFormation console](https://console.aws.amazon.com/cloudformation) will show `CREATE_COMPLETE` for the newly created stack.
:::

## Verify Public MSK Proxy Service

Navigate to the [EC2 running instances dashboard.](https://console.aws.amazon.com/ec2/home#Instances:instanceState=running)

::: note Check your selected region
Make sure you have selected the desired region, such as `US East (N. Virginia) us-east-1`.
:::

Select either of the Public MSK Proxy instances launched by the CloudFormation template to show the details.

::: info
They each have an IAM Role name starting with `aklivity-public-msk-proxy`.
:::

Find the `Public IPv4 Address` and then SSH into the instance.

```bash:no-line-numbers
ssh -i ~/.ssh/<key-pair.cer> ec2-user@<instance-public-ip-address>
```

After logging in via SSH, check the status of the `zilla-plus` system service.

```bash:no-line-numbers
systemctl status zilla-plus.service
```

Verify that the `zilla-plus` service is active and logging output similar to that shown below.

```output:no-line-numbers
● zilla-plus.service - Zilla Plus
   Loaded: loaded (/etc/systemd/system/zilla-plus.service; enabled; vendor preset: disabled)
   Active: active (running) since Tue 2021-08-24 20:56:51 UTC; 1 day 19h ago
 Main PID: 1803 (java)
   CGroup: /system.slice/zilla-plus.service
           └─...

Aug 26 06:56:54 ip-10-0-3-104.ec2.internal zilla[1803]: Recorded usage for record id ...
```

Repeat these steps for each of the other Public MSK Proxy instances launched by the CloudFormation template.

### Configure Global DNS

> This ensures that any new Kafka brokers added to the MSK cluster can still be reached via the Public MSK Proxy.

When using a wildcard DNS name for your own domain, such as `*.example.aklivity.io` then the DNS entries are setup in your DNS provider.

Navigate to the [CloudFormation console](https://console.aws.amazon.com/cloudformation). Then select the `my-public-msk-proxy` stack to show the details.

::: note Check your selected region
Make sure you have selected the desired region, such as `US East (N. Virginia) us-east-1`.
:::

In the stack `Outputs` tab, find the public DNS name of the `NetworkLoadBalancer.`

You need to create a `CNAME` record mapping your public DNS wildcard pattern to the public DNS name of the Network Load Balancer.

::: info
You might prefer to use an Elastic IP address for each NLB public subnet, providing DNS targets for your `CNAME` record that can remain stable even after restarting the stack.
:::

## Verify Kafka Client Connectivity

To verify that we have successfully enabled public internet connectivity to our MSK cluster from the local development environment, we will use a generic Kafka client to create a topic, publish messages and then subscribe to receive these messages from our MSK cluster via the public internet.

### Install the Kafka Client

First, we must install a Java runtime that can be used by the Kafka client.

```bash:no-line-numbers
sudo yum install java-1.8.0
```

Now we are ready to install the Kafka client:

```bash:no-line-numbers
wget https://archive.apache.org/dist/kafka/2.8.0/kafka_2.13-2.8.0.tgz
tar -xzf kafka_2.13-2.8.0.tgz
cd kafka_2.13-2.8.0
```

::: tip
We use a generic Kafka client here, however the setup for any Kafka client, including [KaDeck](https://www.xeotek.com/apache-kafka-monitoring-management/), [Conduktor](https://www.conduktor.io/download/), and [akhq.io](https://akhq.io/) will be largely similar. With the Public MSK Proxy you can use these GUI Kafka clients to configure and monitor your MSK applications, clusters and streams.
:::

### Configure the Kafka Client

With the Kaka client now installed we are ready to configure it and point it at the Public MSK Proxy.

We need to import the trusted client certificate and corresponding private key into the local key store used by the Kafka client when connecting to the Public MSK Proxy.

```bash:no-line-numbers
openssl pkcs12 -export -in client-1.cert.pem -inkey client-1.pkcs8.key.pem -out client-1.p12 -name client-1
keytool -importkeystore -destkeystore /tmp/kafka.client.keystore.jks -deststorepass generated -srckeystore client-1.p12 -srcstoretype PKCS12 -srcstorepass generated -alias client-1
```

In this example, we are importing a private key and certificate with `Common Name` `client-1` signed by a private certificate authority. First the private key and signed certificate are converted into a `p12` formatted key store.

Then the key store is converted to `/tmp/kafka.client.keystore.jks` in `JKS` format. When prompted, use a consistent password for each command. We use the password `generated` to illustrate these steps.

The MSK Proxy relies on TLS so we need to create a file called `client.properties` that tells the Kafka client to use SSL as the security protocol and to specify the key store containing authorized client certificates.

::: code-tabs

@tab client.properties

```toml:no-line-numbers
security.protocol=SSL
ssl.keystore.location=/tmp/kafka.client.keystore.jks
ssl.keystore.password=generated
```

:::

The password configured in `client.properties` should match the password used in the commands above used to create the key store.

::: tip
As the TLS certificate is signed by a globally trusted certificate authority, there is no need to configure your Kafka client to override the trusted certificate authorities.
:::

### Test the Kafka Client

> This verifies internet connectivity to your MSK cluster via Zilla Plus (Public MSK Proxy).

We can now verify that the Kafka client can successfully communicate with your MSK cluster via the internet from your local development environment to create a topic, then publish and subscribe to the same topic.

If using the wildcard DNS pattern `*.example.aklivity.io`, then we use the following as TLS bootstrap server names for the Kafka client:

```text:no-line-numbers
b-1.example.aklivity.io:9094,b-2.example.aklivity.io:9094,b-3.example.aklivity.io:9094
```

::: warning
Replace these TLS bootstrap server names accordingly for your own custom wildcard DNS pattern.
:::

#### Create a Topic

Use the Kafka client to create a topic called `public-proxy-test`, replacing `<tls-bootstrap-server-names>` in the command below with the TLS proxy names of your Public MSK Proxy:

```bash:no-line-numbers
bin/kafka-topics.sh --create --topic public-proxy-test --partitions 3 --replication-factor 3 --command-config client.properties --bootstrap-server <tls-bootstrap-server-names>
```

::: tip A quick summary of what just happened

1. The Kafka client with access to the public internet issued a request to create a new topic
2. This request was directed to the internet-facing Network Load Balancer
3. The Network Load Balancer forwarded the request to the Zilla Plus (Public MSK Proxy)
4. The Zilla Plus (Public MSK Proxy) verified the client identity of the Kafka client
5. The Zilla Plus (Public MSK Proxy) selected a matching client certificate to propagate client identity
6. The Zilla Plus (Public MSK Proxy) routed the request to the appropriate MSK broker
7. The topic was created in the MSK broker
8. Public access was verified, authorized by trusted client certificate

:::

#### Publish messages

Publish two messages to the newly created topic via the following producer command:

```bash:no-line-numbers
bin/kafka-console-producer.sh --topic public-proxy-test --producer.config client.properties --broker-list <tls-bootstrap-server-names>
```

A prompt will appear for you to type in the messages:

```output:no-line-numbers
>This is my first event
>This is my second event
```

#### Receive messages

Read these messages back via the following consumer command:

```bash:no-line-numbers
bin/kafka-console-consumer.sh --topic public-proxy-test --from-beginning --consumer.config client.properties --bootstrap-server <tls-bootstrap-server-names>
```

You should see the `This is my first event` and `This is my second event` messages.

```output:no-line-numbers
This is my first event
This is my second event
```

::: info Monitor the Public MSK Proxy

Follow the [Monitoring the Public MSK Proxy](./public-proxy.md#monitoring-the-public-msk-proxy) instructions

:::

::: info Upgrade the Public MSK Proxy

Follow the [Upgrading the Public MSK Proxy](./public-proxy.md#upgrading-the-public-msk-proxy) instructions

:::