.PHONY: help install test bucket build package deploy invoke-local invoke-deployed

SAM_S3_BUCKET = sam-$(shell aws sts get-caller-identity --query Account --output text)-$(shell aws configure get region)
SAM_S3_PREFIX = {{cookiecutter.project_name}}

_TEMPLATE = .aws-sam/build/template.yaml


help:           ## Show this help.
	@fgrep -h "##" $(MAKEFILE_LIST) | sed -e 's/\([^:]*\):[^#]*\(##.*\)/\1			\2/' | grep -v fgrep

build: $(_TEMPLATE)	## content of the the lambda function code zip

test:		## run the local unit tests
	pipenv install -d
	pipenv install -r */requirements.txt
	pipenv run pytest

deploy: packaged.yaml	## the custom provider to AWS
	sam deploy \
		--template-file packaged.yaml \
		--capabilities CAPABILITY_IAM \
		--stack-name {{cookiecutter.project_name}}

$(_TEMPLATE): template.yaml custom_provider/requirements.txt  custom_provider/*
	sam build --use-container
	find .aws-sam -type d -print0 | xargs -0 chmod go+rx
	find .aws-sam -type f -print0 | xargs -0 chmod go+r

packaged.yaml: $(_TEMPLATE)
	sam package \
		--s3-bucket $(SAM_S3_BUCKET) \
		--s3-prefix $(SAM_S3_PREFIX) \
		--output-template-file packaged.yaml


bucket:		## create s3 bucket for sam artifacts $(SAM_S3_BUCKET)
	@pipenv run aws s3 ls s3://$(SAM_S3_BUCKET) > /dev/null 2>&1 || pipenv run aws s3 mb s3://$(SAM_S3_BUCKET) ; echo $(SAM_S3_BUCKET) created



EVENT = events/create.json	# to send to the provider
invoke-local: $(_TEMPLATE)		## custom provider
	sam local invoke -e $(EVENT)

invoke-deployed: FUNCTION_ARN=$(shell aws cloudformation describe-stacks \
				  --stack-name {{cookiecutter.project_name}} \
				  --query 'Stacks[0].Outputs[?OutputKey==`ServiceToken`].OutputValue' \
				  --output text)
invoke-deployed:	## custom provider
	aws lambda invoke \
		--function-name $(FUNCTION_ARN) \
		--payload '$(shell cat $(EVENT))' \
		/dev/stdout