# AWS EC2 Instance Setup and SSH Access Recovery

## Overview

This project documents the setup and troubleshooting of an Amazon EC2 instance running Amazon Linux 2023 on AWS. It covers SSH authentication using an RSA private key, AWS Systems Manager Session Manager access, and restoring SSH connectivity without replacing the existing private key.

## Architecture

- **Cloud Provider:** Amazon Web Services (AWS)
- **Compute:** Amazon EC2
- **Operating System:** Amazon Linux 2023
- **Authentication:** RSA SSH key pair
- **Remote Access:** SSH and AWS Systems Manager Session Manager
- **Region:** `us-west-2` (US West — Oregon)
- **IAM Instance Profile:** `OpenClaw-EC2`

## Technologies and Tools

- AWS EC2
- AWS Systems Manager (SSM)
- AWS CloudShell
- Ubuntu Linux
- OpenSSH
- AWS CLI
- Linux file permissions and SSH key management

## EC2 Configuration

| Setting | Value |
|---|---|
| AWS Region | `us-west-2` |
| Operating System | Amazon Linux 2023 |
| SSH Username | `ec2-user` |
| Key Pair Name | `bairamov-ai-us-west-2` |
| Key Type | RSA |
| SSM Agent Version | `3.3.5226.0` |
| Instance Profile | `OpenClaw-EC2` |

> **Security note:** The instance ID and public IP are intentionally omitted from this public documentation. Never publish private keys, credentials, AWS account IDs, or other sensitive infrastructure details.

## Project Objectives

1. Configure secure SSH access to an EC2 instance.
2. Verify private key permissions on Ubuntu.
3. Diagnose SSH `Permission denied (publickey)` errors.
4. Use Systems Manager Session Manager as an alternative access method.
5. Inspect the SSH `authorized_keys` configuration.
6. Restore access using the public key derived from an existing private key.
7. Understand how EC2 public IP addresses can change after stopping and starting an instance.

## 1. Configure the SSH Private Key

Store the downloaded PEM file in your Ubuntu home directory.

```bash
ls -l ~/bairamov-ai-us-west-2.pem
```

Restrict access to the private key:

```bash
chmod 400 ~/bairamov-ai-us-west-2.pem
```

SSH requires private keys to have appropriately restrictive permissions.

## 2. Generate the Public Key

Derive the public key from the existing private key:

```bash
ssh-keygen -y \
  -f ~/bairamov-ai-us-west-2.pem \
  > ~/bairamov-ai-us-west-2.pub
```

Verify the public key fingerprint:

```bash
ssh-keygen -lf ~/bairamov-ai-us-west-2.pub
```

This command derives the public key without changing the original private key.

## 3. Connect to the EC2 Instance

Use the Amazon Linux default username, `ec2-user`, and the instance's current public IP address.

```bash
ssh -o IdentitiesOnly=yes \
  -i ~/bairamov-ai-us-west-2.pem \
  ec2-user@YOUR_EC2_PUBLIC_IP
```

Replace `YOUR_EC2_PUBLIC_IP` with the current public IP shown in the EC2 console.

If authentication fails, enable detailed SSH diagnostics:

```bash
ssh -vvv -o IdentitiesOnly=yes \
  -i ~/bairamov-ai-us-west-2.pem \
  ec2-user@YOUR_EC2_PUBLIC_IP
```

Look for messages indicating which public key is offered and whether the server accepts or rejects it.

## 4. Troubleshoot SSH Authentication

A `Permission denied (publickey)` error can occur when:

- The private key does not correspond to an authorized public key.
- The wrong SSH username is used.
- The public key is missing from `authorized_keys`.
- The SSH directory or file permissions are incorrect.
- SSH server configuration or account restrictions prevent authentication.

Check the authorized keys on the EC2 instance:

```bash
ls -ld ~/.ssh
ls -l ~/.ssh/authorized_keys
ssh-keygen -lf ~/.ssh/authorized_keys
```

Compare the fingerprints with the public key derived from the local PEM file.

## 5. Restore Access Using Session Manager

AWS Systems Manager Session Manager provides shell access without requiring SSH authentication.

From AWS CloudShell, start a session:

```bash
aws ssm start-session \
  --target YOUR_INSTANCE_ID \
  --region us-west-2
```

Replace `YOUR_INSTANCE_ID` with your EC2 instance ID.

Once connected, inspect the authorized keys:

```bash
sudo ls -ld /home/ec2-user/.ssh
sudo ls -l /home/ec2-user/.ssh/authorized_keys
sudo ssh-keygen -lf /home/ec2-user/.ssh/authorized_keys
```

Session Manager requires the appropriate IAM permissions, a functioning SSM Agent, and connectivity to the Systems Manager service.

## 6. Restore the Existing Public Key

If the public key corresponding to your private key is missing from the authorized keys file, add it without deleting existing entries.

On the EC2 instance:

```bash
sudo nano /home/ec2-user/.ssh/authorized_keys
```

Append the public key generated from the existing PEM file as a new line. Preserve existing authorized keys.

Set appropriate ownership and permissions:

```bash
sudo chown -R ec2-user:ec2-user /home/ec2-user/.ssh
sudo chmod 700 /home/ec2-user/.ssh
sudo chmod 600 /home/ec2-user/.ssh/authorized_keys
```

Then retry the SSH connection from Ubuntu using the current public IP.

**Important:** Only add a public key you control and have verified. Never upload or paste your private PEM key into `authorized_keys`.

## 7. Check the EC2 Instance Status

From AWS CloudShell:

```bash
aws ec2 describe-instances \
  --instance-ids YOUR_INSTANCE_ID \
  --region us-west-2 \
  --query 'Reservations[0].Instances[0].[State.Name,PublicIpAddress,KeyName]' \
  --output table
```

This command checks the instance state, current public IP, and assigned key pair.

When an EC2 instance is stopped and restarted, its automatically assigned public IPv4 address can change. Always retrieve the current address before connecting.

## 8. Security Best Practices

- Never commit `.pem` files, private keys, passwords, or AWS credentials to GitHub.
- Add sensitive files to `.gitignore`.
- Use IAM roles rather than storing long-term AWS credentials on EC2.
- Restrict SSH inbound access to trusted IP addresses.
- Prefer AWS Systems Manager for administrative access where practical.
- Keep authorized keys limited to trusted users.
- Use least-privilege IAM permissions.
- Review AWS resource usage to avoid unnecessary charges.

Example `.gitignore` entries:

```gitignore
*.pem
*.key
.env
.aws/
credentials
```

## Key Lessons Learned

- A readable private key does not guarantee that an EC2 instance authorizes it.
- The public key derived from a private key must match an entry authorized by the server.
- SSH debugging helps identify authentication failures.
- Systems Manager can provide an alternative route for recovering SSH access.
- Stopping and restarting an EC2 instance can change its public IPv4 address.
- Public-key authentication can be restored without replacing the existing private key when the corresponding public key is available.

## Disclaimer

This repository documents a personal AWS learning and troubleshooting project. Resource configurations, IAM permissions, network rules, and AWS costs depend on the deployment environment.

Do not publish real private keys, account credentials, or other secrets in this repository.
