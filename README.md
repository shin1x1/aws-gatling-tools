# aws-gatling-tools

STATUS: WIP

aws-gatling-tools is the aws stress-test tool used by [gatling](https://gatling.io/). This system can perform stress-tests using processes on Fargate, output the report to S3, and notify the report url to chats.

<img src="https://raw.githubusercontent.com/j5ik2o/aws-gatling-tools/master/doc/system-layout.png"/>

## How to prepare

- installing tool
    ```sh
    $ brew install sbt awscli jq
    ```
- installing Docker for Mac
- register an IAM user in AWS account
- add a profile for the IAM user as `aws-gatling-tools` to `~/.aws/credentails`.
    ```sh
    [aws-gatling-tools]
    aws_access_key_id = XXXXX
    aws_secret_access_key = XXXXX
    region = ap-northeast-1 
    ```

## How to build the AWS environment

```sh
$ cd terraform
terraform $ cp terraform.tfvars.default terraform.tfvars
terraform $ terraform init
terraform $ terraform plan
terraform $ terraform apply
```

## How to build the test application

```sh
# api-server docker build & push
$ AWS_DEFAULT_PROFILE=aws-gatling-tools sbt api-server/ecr:push
```

## How to build the stress-test tools

```sh
# gatling-runner docker build & push
$ AWS_DEFAULT_PROFILE=aws-gatling-tools sbt gatling-runner/ecr:push

# gatling-s3-reporter docker build & push
$ cd gatling-s3-reporter && make release && cd ..

# gatling-aggregate-runner build & push
$ AWS_DEFAULT_PROFILE=aws-gatling-tools sbt gatling-aggregate-runner/ecr:push
```

## How to run a stress-test

```sh
$ AWS_PROFILE=aws-gatling-tools \
    GATLING_NOTICE_SLACK_INCOMING_WEBHOOK_URL=https://hooks.slack.com/services/xxxxx \
    GATLING_TARGET_HOST=http://XXXXX.XXXXXX.XXXXX/hello \
    sbt gatling-aggregate-runner/gatling::runTask
```

1. `Aggregate Runner` starts on the ECS cluster.
1. `Aggregate Runner` starts the specified number of `Runners`, notifies to chat.
1. Wait for all `Runners` until finish.
1. After all runners have finished, launch the `S3 Reporter`, `Aggregate Runner` notifies to chat
1. `Aggregate Runner` notifies the url to gatling report on S3.

### chat log

```
Gatling Runner started:
task arns = [
https://ap-northeast-1.console.aws.amazon.com/ecs/home?region=ap-northeast-1#/clusters/j5ik2o-aws-gatling-tools-ecs/tasks/xxxxxxxxxx/details
...
https://ap-northeast-1.console.aws.amazon.com/ecs/home?region=ap-northeast-1#/clusters/j5ik2o-aws-gatling-tools-ecs/tasks/xxxxxxxxxx/details
]
runTaskCount = 10, runTaskEnvironments = Map(GATLING_S3_BUCKET_NAME -> j5ik2o-aws-gatling-tools-logs, GATLING_PAUSE_DURATION -> 3s, GATLING_TARGET_ENDPOINT_BASE_URL -> http://tf-xxxxxxxxxx-xxxxxxxx.ap-northeast-1.elb.amazonaws.com, GATLING_EXECUTION_ID -> api-server/xxxxxxxx-xxxxxxxxx, GATLING_SIMULATION_CLASS -> com.github.j5ik2o.gatling.BasicSimulation, GATLING_RAMP_DURATION -> 200s, GATLING_RESULT_DIR -> target/gatling, GATLING_HOLD_DURATION -> 5m, GATLING_USERS -> 10, AWS_REGION -> ap-northeast-1)
Gatling Runner finished: task arns = [
https://ap-northeast-1.console.aws.amazon.com/ecs/home?region=ap-northeast-1#/clusters/j5ik2o-aws-gatling-tools-ecs/tasks/xxxxxxxxxx/details
...
https://ap-northeast-1.console.aws.amazon.com/ecs/home?region=ap-northeast-1#/clusters/j5ik2o-aws-gatling-tools-ecs/tasks/xxxxxxxxxx/details
]
Gatling Reporter started: task arns = https://ap-northeast-1.console.aws.amazon.com/ecs/home?region=ap-northeast-1#/clusters/j5ik2o-aws-gatling-tools-ecs/tasks/xxxxx-xxxxx-xxxxx-xxxx-xxxxxxxxxxx/details
runTaskReporterEnvironments = Map(AWS_REGION -> ap-northeast-1, GATLING_BUCKET_NAME -> j5ik2o-aws-gatling-tools-logs, GATLING_RESULT_DIR_PATH -> api-server/xxxxxxxxxxx-xxxxxxxxxxxx)
Gatling Reporter finished: report url: https://j5ik2o-aws-gatling-tools-logs.s3.amazonaws.com/api-server/xxxxxxxxx-xxxxxxxxxxx/index.html
```
