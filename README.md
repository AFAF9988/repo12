#Use AWS Organizations and SSO Login to give separate permissions to Developers, DevOps, Operations, and Testing teams. Create their permissions with the help of separate Roles or permission sets or Groups.

Step 1: Create permission sets:
In this step, we create EC2-S3FullAccess, and PowerUserAccess permission sets in AWS SSO from the AWS CLI.
Before you create the permission sets, run the following command to get the Amazon Resource Name (ARN) of the AWS SSO instance and the Identity Store ID, which you will need later in the process when you create and assign permission sets to AWS accounts and users or groups.
aws sso-admin list-instances
Permission set for Alice Developer (EC2-S3-FullAccess)	

aws sso-admin create-permission-set --instance-arn '<Instance ARN>' --name 'EC2-S3-FullAccess' --description 'EC2 and S3 access for developers'

Permission set for Frank Infosec (PowerUserAccess)

aws sso-admin create-permission-set --instance-arn '<Instance ARN>' --name 'PowerUserAccess' --description 'Power User Access for security team on ExampleOrgDev account'

Step 2: Assign policies to permission sets

Attach policies to the EC2-S3-FullAccess permission set

aws sso-admin attach-managed-policy-to-permission-set --instance-arn '<Instance ARN>' --permission-set-arn '<Permission Set ARN>' --managed-policy-arn 'arn:aws:iam::aws:policy/amazonec2fullaccess'
aws sso-admin attach-managed-policy-to-permission-set --instance-arn '<Instance ARN>' --permission-set-arn '<Permission Set ARN>' --managed-policy-arn 'arn:aws:iam::aws:policy/amazons3fullaccess'
Assign the PowerUserAccess permission set to Frank Infosec in the ExampleOrgTest account

aws sso-admin create-account-assignment --instance-arn '<Instance ARN>' --permission-set-arn '<Permission Set ARN>' --principal-id '<user/group ID>' --principal-type '<USER/GROUP>' --target-id '<AWS Account ID>' --target-type AWS_ACCOUNT


Use the AWS SSO API through AWS CloudFormation

{
    "AWSTemplateFormatVersion": "2010-09-09",
    
    "Parameters": {
      "InstanceARN" : {
        "Type" : "String",
        "AllowedPattern": "arn:aws:sso:::instance/(sso)?ins-[a-zA-Z0-9-.]{16}",
        "Description" : "Enter AWS SSO InstanceARN. Ex: arn:aws:sso:::instance/ssoins-xxxxxxxxxxxxxxxx",
        "ConstraintDescription": "must be the name of an existing AWS SSO InstanceARN associated with the management account."
      },
      "ExampleOrgDevAccountId" : {
        "Type" : "String",
        "AllowedPattern": "\\d{12}",
        "Description" : "Enter 12-digit Developer AWS Account ID. Ex: 123456789012"
        },
      "ExampleOrgTestAccountId" : {
        "Type" : "String",
        "AllowedPattern": "\\d{12}",
        "Description" : "Enter 12-digit AWS Account ID. Ex: 123456789012"
        },
      "AliceDeveloperUserId" : {
        "Type" : "String",
        "AllowedPattern": "^([0-9a-f]{10}-|)[A-Fa-f0-9]{8}-[A-Fa-f0-9]{4}-[A-Fa-f0-9]{4}-[A-Fa-f0-9]{4}-[A-Fa-f0-9]{12}$",
        "Description" : "Enter Developer UserId. Ex: 926703446b-f10fac16-ab5b-45c3-86c1-xxxxxxxxxxxx"
        },
        "FrankInfosecUserId" : {
            "Type" : "String",
            "AllowedPattern": "^([0-9a-f]{10}-|)[A-Fa-f0-9]{8}-[A-Fa-f0-9]{4}-[A-Fa-f0-9]{4}-[A-Fa-f0-9]{4}-[A-Fa-f0-9]{12}$",
            "Description" : "Enter Test UserId. Ex: 926703446b-f10fac16-ab5b-45c3-86c1-xxxxxxxxxxxx"
            }
    },
    "Resources": {
        "EC2S3Access": {
            "Type" : "AWS::SSO::PermissionSet",
            "Properties" : {
                "Description" : "EC2 and S3 access for developers",
                "InstanceArn" : {
                    "Ref": "InstanceARN"
                },
                "ManagedPolicies" : ["arn:aws:iam::aws:policy/amazonec2fullaccess","arn:aws:iam::aws:policy/amazons3fullaccess"],
                "Name" : "EC2-S3-FullAccess",
                "Tags" : [ {
                    "Key": "Name",
                    "Value": "EC2S3Access"
                 } ]
              }
        },  
        "SecurityAuditAccess": {
            "Type" : "AWS::SSO::PermissionSet",
            "Properties" : {
                "Description" : "Audit Access for Infosec team",
                "InstanceArn" : {
                    "Ref": "InstanceARN"
                },
                "ManagedPolicies" : [ "arn:aws:iam::aws:policy/SecurityAudit" ],
                "Name" : "AuditAccess",
                "Tags" : [ {
                    "Key": "Name",
                    "Value": "SecurityAuditAccess"
                 } ]
              }
        },    
        "PowerUserAccess": {
            "Type" : "AWS::SSO::PermissionSet",
            "Properties" : {
                "Description" : "Power User Access for Infosec team",
                "InstanceArn" : {
                    "Ref": "InstanceARN"
                },
                "ManagedPolicies" : [ "arn:aws:iam::aws:policy/PowerUserAccess"],
                "Name" : "PowerUserAccess",
                "Tags" : [ {
                    "Key": "Name",
                    "Value": "PowerUserAccess"
                 } ]
              }      
        },
        "EC2S3userAssignment": {
            "Type" : "AWS::SSO::Assignment",
            "Properties" : {
                "InstanceArn" : {
                    "Ref": "InstanceARN"
                },
                "PermissionSetArn" : {
                    "Fn::GetAtt": [
                        "EC2S3Access",
                        "PermissionSetArn"
                     ]
                },
                "PrincipalId" : {
                    "Ref": "AliceDeveloperUserId"
                },
                "PrincipalType" : "USER",
                "TargetId" : {
                    "Ref": "ExampleOrgDevAccountId"
                },
                "TargetType" : "AWS_ACCOUNT"
              }
          },
          "SecurityAudituserAssignment": {
            "Type" : "AWS::SSO::Assignment",
            "Properties" : {
                "InstanceArn" : {
                    "Ref": "InstanceARN"
                },
                "PermissionSetArn" : {
                    "Fn::GetAtt": [
                        "SecurityAuditAccess",
                        "PermissionSetArn"
                     ]
                },
                "PrincipalId" : {
                    "Ref": "FrankInfosecUserId"
                },
                "PrincipalType" : "USER",
                "TargetId" : {
                    "Ref": "ExampleOrgDevAccountId"
                },
                "TargetType" : "AWS_ACCOUNT"
              }
          },
          "PowerUserAssignment": {
            "Type" : "AWS::SSO::Assignment",
            "Properties" : {
                "InstanceArn" : {
                    "Ref": "InstanceARN"
                },
                "PermissionSetArn" : {
                    "Fn::GetAtt": [
                        "PowerUserAccess",
                        "PermissionSetArn"
                     ]
                },
                "PrincipalId" : {
                    "Ref": "FrankInfosecUserId"
                },
                "PrincipalType" : "USER",
                "TargetId" : {
                    "Ref": "ExampleOrgTestAccountId"
                },
                "TargetType" : "AWS_ACCOUNT"
              }
          }
    }
}


  
 Demonstrate ability to deploy app locally using Docker compose.



Create a directory for the project:
$ mkdir composetest
$ cd composetest
  
  
Create a file called app.py in your project directory and paste this in:
import time

import redis
from flask import Flask

app = Flask(__name__)
cache = redis.Redis(host='redis', port=6379)

def get_hit_count():
    retries = 5
    while True:
        try:
            return cache.incr('hits')
        except redis.exceptions.ConnectionError as exc:
            if retries == 0:
                raise exc
            retries -= 1
            time.sleep(0.5)
@app.route('/')
def hello():
    count = get_hit_count()
    return 'Hello World! I have been seen {} times.\n'.format(count)
  
  
Create another file called requirements.txt in your project directory and paste this in:
flask
redis
  
  
Create a Dockerfile
# syntax=docker/dockerfile:1
FROM python:3.7-alpine
WORKDIR /code
ENV FLASK_APP=app.py
ENV FLASK_RUN_HOST=0.0.0.0
RUN apk add --no-cache gcc musl-dev linux-headers
COPY requirements.txt requirements.txt
RUN pip install -r requirements.txt
EXPOSE 5000
COPY . .
CMD ["flask", "run"]
  
  
Define services in a Compose file (docker-compose.yml)
version: "3.9"
services:
  web:
    build: .
    ports:
      - "5000:5000"
  redis:
    image: "redis:alpine"

 Build and run your app with Compose


$ docker-compose up		

Edit the Compose file to add a bind mount


version: "3.9"
services:
  web:
    build: .
    ports:
      - "5000:5000"
    volumes:
      - .:/code
    environment:
      FLASK_ENV: development
  redis:
    image: "redis:alpine"

Re-build and run the app with Compose

$ docker-compose up













  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  













