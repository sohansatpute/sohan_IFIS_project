# IFIS AWS Infrastructure POC

## Purpose

This repository contains a reusable AWS infrastructure implementation
based on the IFIS / W2P-Japan architecture.

The objective is to reproduce and validate the architecture using
AWS CloudFormation so that the infrastructure can be replicated
for future environments.

## AWS Region

Primary POC Region:

ap-south-1

## Existing VPC

VPC ID:

vpc-06900f62513eff63a

CIDR:

172.31.0.0/16

## Architecture

Internet
    |
    v
CloudFront
    |
   WAF
    |
    v
CloudFront VPC Origin
    |
    v
Private EC2
    |
    v
NAT Gateway
    |
    v
Internet

AWS Backup is used for EC2 backup and recovery.

## CloudFormation Stacks

The infrastructure will be deployed using separate stacks:

1. Network
2. Compute
3. CloudFront
4. WAF
5. Backup

## Project Status

Currently preparing the repository and deployment framework.
