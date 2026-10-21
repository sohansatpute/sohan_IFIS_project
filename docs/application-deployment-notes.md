# W2P Japan Application Deployment Notes

This document outlines the boundary between the AWS CloudFormation infrastructure and the W2P Japan application deployment. 

**Important:** The application layer is currently configured manually by the developer and does not utilize a CI/CD pipeline.

## 1. Infrastructure vs. Application Responsibilities

*   **CloudFormation (AWS Infra):** Provisions the VPC, private subnet, NAT Gateway, CloudFront, WAF, AWS Backup, and a bare Amazon Linux 2023 EC2 instance with Apache (`httpd`) installed.
*   **Developer (Manual Setup):** Installs PHP 8.3, MySQL 8.4, all associated dependencies, configures the database, and deploys the application code.

## 2. Application Source Code

*   **Repository:** The application code is hosted on the Client's GitHub repository.
*   **Production Path:** `/var/www/w2pjapan`
*   **Initial Deployment:** The code is manually cloned or transferred to the EC2 instance.

## 3. Software Stack (Manual Installation)

The developer manually installs and configures the following on the EC2 instance after CloudFormation completes:
*   PHP 8.3 (and required extensions)
*   MySQL 8.4
*   WordPress 7.0.3
*   Custom Theme: `w2p1508`
*   Parent Theme: `Habakiri`

## 4. Future Code Updates

There is no automated CI/CD pipeline (e.g., AWS CodePipeline) for this application.
*   **Process:** Minor code-level changes (e.g., theme tweaks, plugin updates) are made directly on the EC2 instance.
*   **Risk:** Because changes are made directly on the server, they are not tracked in Git unless the developer manually commits and pushes them.

## 5. Backup and Rollback

During the initial upgrade, the developer created local backups:
*   Application Backup: `/tmp/w2pjapan-before-upgrade.tar.gz`
*   Database Backup: `/tmp/w2pjapan-before-upgrade.sql`

> **AWS Warning:** `/tmp` is an ephemeral directory in AWS. These files will be deleted if the EC2 instance is stopped, rebooted, or terminated. It is highly recommended to move these backups to a persistent location (e.g., `/var/backups` or Amazon S3) and rely on the AWS Backup snapshots provisioned by CloudFormation.

## 6. Database Rollback Commands (If needed)

If a rollback is required and the local `/tmp` files are still present:

1. Stop Apache:
   `sudo systemctl stop httpd`
2. Restore Files:
   `sudo mv /var/www/w2pjapan /var/www/w2pjapan-failed`
   `sudo mkdir -p /var/www/w2pjapan`
   `sudo tar -xzf /tmp/w2pjapan-before-upgrade.tar.gz -C /var/www/w2pjapan --strip-components=1`
   `sudo chown -R apache:apache /var/www/w2pjapan`
3. Restore Database:
   `mysql -u root -p [DATABASE_NAME] < /tmp/w2pjapan-before-upgrade.sql`
4. Start Apache:
   `sudo systemctl start httpd`