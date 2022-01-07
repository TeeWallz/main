# Package Python Lambda via Docker
  https://docs.aws.amazon.com/lambda/latest/dg/python-image.html

# PAckage and upload Lambda ZIP file
  https://docs.aws.amazon.com/lambda/latest/dg/python-package.html

# CloudFormation Lambda Upload Makefile

    SHELL=/bin/bash
    .PHONY: init package clean

    # Makefile and start from https://hands-on.cloud/how-to-create-and-deploy-your-first-python-aws-lambda-function/
    # Finished off with https://docs.aws.amazon.com/lambda/latest/dg/python-package.html
    # NOTE: Build needs literal tabs, not spaces for whitespace below

    STACK_NAME=snowflake-license-reporting
    LAMBDA_ZIP=$(STACK_NAME).zip
    SERVICE_S3_BUCKET=lambda-functions-overwatch


    init:
      docker run --rm -v `pwd`:/src -w /src python:3.9-slim-buster /bin/bash -c "apt-get update && \
        mkdir -p ./build/packages && \
        pip install -r requirements.txt -t ./build/packages"

    lambda.zip:
      docker run --rm -v `pwd`:/src -w /src python /bin/bash -c "apt-get update && \
        apt-get install -y zip && \
        cd ./build/packages && \
        zip -r ../$(LAMBDA_ZIP) . && cd ../../ && zip -g build/deploy.zip lambda_function.py"
    package: lambda.zip

    install: lambda.zip
      aws s3 cp ./build/$(LAMBDA_ZIP) s3://$(SERVICE_S3_BUCKET)/$(LAMBDA_ZIP)
      echo "Use CFN stack to deploy Lambda from s3://$(SERVICE_S3_BUCKET)/lambdas/$(LAMBDA_ZIP)"

    deploy:
      aws cloudformation delete-stack --stack-name $(STACK_NAME)
      aws cloudformation wait stack-delete-complete --stack-name $(STACK_NAME)
      aws cloudformation create-stack \
        --stack-name $(STACK_NAME) \
        --template-body file://cloudformation_template.yml \
        --capabilities CAPABILITY_IAM \
        --parameters \
          ParameterKey=LambdaNameParameter,ParameterValue=$(STACK_NAME) \
          ParameterKey=S3BucketParameter,ParameterValue=$(SERVICE_S3_BUCKET) \
          ParameterKey=S3KeyParameter,ParameterValue=$(LAMBDA_ZIP)
      aws cloudformation wait \
        stack-create-complete \
        --stack-name $(STACK_NAME)

    fulldeploy: init install deploy

    update_lambda:
      ls

    clean:
      rm -rf ./build
