![Python application](https://github.com/velimir/canoe/workflows/Python%20application/badge.svg)

# canoe

Send kayako ticket updates to Slack

## Development

Use it with pipenv. `pytest` should be installed in the virtualenv. See `build.sh` for a full
deployment to aws. 

See `env.json.sample` for required environment variables.

## Running Tests

1. Export all required environment variables
2. `pipenv shell`
3. Verify `pytest` is running from `pipenv`: `which pytest`, if no then uninstall `pytest` outside
   of the `pipenv` by running `pip uninstall pytest`
4. `pipenv install`
5. `pytest` 

## Requirements

* AWS CLI with Administrator permission
* [Python 3 installed](https://www.python.org/downloads/)
* [Pipenv installed](https://github.com/pypa/pipenv)
    - `pip install pipenv`
* [Docker installed](https://www.docker.com/community-edition)
* [SAM Local installed](https://github.com/awslabs/aws-sam-local) 


As you've chosen the experimental Makefile we can use Make to automate Packaging and Building steps as follows:

```bash
        ...::: Installs all required packages as defined in the Pipfile :::...
        make install

        ...::: Run Pytest under tests/ with pipenv :::...
        make test

        ...::: Creates local dev environment for Python hot-reloading w/ packages:::...
        make build SERVICE="canoe"

        ...::: Run SAM Local API Gateway :::...
        make run

        # or

        ...::: Run SAM Invoke Function :::...
        make invoke SERVICE="FirstFunction" EVENT="events/canoe_event.json"
```


## Testing

`Pytest` is used to discover tests created under `tests` folder - Here's how you can run tests our initial unit tests:


```bash
make test
```


## Packaging

AWS Lambda Python runtime requires a flat folder with all dependencies including the application. To facilitate this process, the pre-made SAM template expects this structure to be under `canoe/build/`:

```yaml
...
    FirstFunction:
        Type: AWS::Serverless::Function
        Properties:
            CodeUri: canoe/build/
            ...
```

With that in mind, we will:

1. Generate a hashed `requirements.txt` out of our `Pipfile` dep file
1. Install all dependencies directly to `build` sub-folder
2. Copy our function (app.py) into `build` sub-folder


Given that you've chosen a Makefile these steps are automated by simply running: ``make build SERVICE="canoe"``



### Local development

Given that you followed Packaging instructions then run the following to invoke your function locally:


**Invoking function locally without API Gateway**

```bash
echo '{"lambda": "payload"}' | sam local invoke FirstFunction
```

You can also specify a `event.json` file with the payload you'd like to invoke your function with:

```bash
sam local invoke -e event.json FirstFunction
```



## Deployment

First and foremost, we need a S3 bucket where we can upload our Lambda functions packaged as ZIP before we deploy anything - If you don't have a S3 bucket to store code artifacts then this is a good time to create one:

```bash
aws s3 mb s3://BUCKET_NAME
```

Provided you have a S3 bucket created, run the following command to package our Lambda function to S3:

```bash
aws cloudformation package \
    --template-file template.yaml \
    --output-template-file packaged.yaml \
    --s3-bucket REPLACE_THIS_WITH_YOUR_S3_BUCKET_NAME
```

Next, the following command will create a Cloudformation Stack and deploy your SAM resources.

```bash
aws cloudformation deploy \
    --template-file packaged.yaml \
    --stack-name canoe \
    --capabilities CAPABILITY_IAM
```

> **See [Serverless Application Model (SAM) HOWTO Guide](https://github.com/awslabs/serverless-application-model/blob/master/HOWTO.md) for more details in how to get started.**



# Appendix


## Makefile

It is important that the Makefile created only works on OSX/Linux but the tasks above can easily be turned into a Powershell or any scripting language you may want too.

The following make targets will automate that we went through above:

* Find all available targets: `make`
* Install all deps and clone (OS hard link) our lambda function to `/build`: `make build SERVICE="canoe"`
    - `SERVICE="canoe"` tells Make to start the building process from there
    - By creating a hard link we no longer need to keep copying our app over to Build and keeps it tidy and clean
* Run `Pytest` against all tests found under `tests` folder: `make test`
* Install all deps and builds a ZIP file ready to be deployed: `make package SERVICE="canoe"`
    - You can also build deps and a ZIP file within a Docker Lambda container: `make package SERVICE="canoe" DOCKER=1`
    - **This is particularly useful when using C-extensions that if built on your OS may not work when deployed to Lambda (different OS)**


## AWS CLI commands

AWS CLI commands to package, deploy and describe outputs defined within the cloudformation stack:

```bash
aws cloudformation package \
    --template-file template.yaml \
    --output-template-file packaged.yaml \
    --s3-bucket REPLACE_THIS_WITH_YOUR_S3_BUCKET_NAME

aws cloudformation deploy \
    --template-file packaged.yaml \
    --stack-name canoe \
    --capabilities CAPABILITY_IAM \
    --parameter-overrides MyParameterSample=MySampleValue

aws cloudformation describe-stacks \
    --stack-name canoe --query 'Stacks[].Outputs'
```

## Adding it to Slack


* Create Slack Application Integration
* Add Oauth Scope chat:write 
* Copy generated API token 
* Change API token for UpdatesNotificationsFunction on Amazon Lambda / London function
* Invite the new app to the room you want to post
