# docker-gitlab-runner-fargate-driver

A docker image containing the Gitlab Runner and the Fargate driver that can be configurable using container environment variables. The Runner automatically registers itself as a Runner in your Gitlab project using a specified registration token.

Important: if the container is shutdown gracefully, the Runner will try to unregister itself in the Gitlab project before the container is stopped.

## Container parameters

| Parameter | Required | Description |
|-|-|-|
| GITLAB_REGISTRATION_TOKEN | Yes | The registration token for the Gitlab project. To obtain it you will need to go to your project CI/CD settings (i.e: https://gitlab.com/your/project/-/settings/ci_cd), open the `Runners` section and check the registration token under  `Set up a specific Runner manually`. |
| GITLAB_URL                | No  | URL to the GitLab instance (defaults to GitLab SaaS, https://gitlab.com), no trailing slash. |
| RUNNER_TAG_LIST           | No  | You can set up jobs to only use Runners with specific tags. Separate each tag by comma. |
| FARGATE_CLUSTER           | Yes | Name of the ECS Fargate Cluster where CI jobs should be processed. |
| FARGATE_REGION            | Yes | AWS region where CI jobs should be processed. |
| FARGATE_SUBNET            | Yes | AWS subnet where CI jobs should be processed. |
| FARGATE_SECURITY_GROUP    | Yes | The AWS security group name. |
| FARGATE_TASK_DEFINITION   | Yes | Task Definition that will be used to create the task to process the CI job (as mentioned previously, creating it is not scope of this article). |

## How to configure and run

High level overview of the solution:

1. Fine tune the template files for your need (config_runner_template.toml and config_driver_template.toml)
1. Create a docker image for your the task that will execute your jobs. Build and push it into a docker repository
1. Create a Task Definition for the task the will execute your jobs
1. Run the docker inside an AWS node passing the expected environment variables

Bellow we describe those steps in details.

### Fine tune the template files for your need (config_runner_template.toml and config_driver_template.toml)

- config_runner_template.toml

The config_runner_template.toml presented in this repository is just an example. You may want to fine tune it to your needs.

- config_driver_template.toml

This template reflects the Gitlab AWS Fargate driver config at the time of this image was ceated. As future releases of the driver may change the configuration file structure or content, it may be necessary to be update this template.

### Create a docker image for your the task that will execute your jobs. Build and push it into a docker repository

Refer to steps 1 and 2 of the [Gitlab configuration guide](https://docs.gitlab.com/runner/configuration/runner_autoscale_aws_fargate/).

### Create a Task Definition for the task the will execute your jobs

Refer to [Create an ECS Task definition step](https://docs.gitlab.com/runner/configuration/runner_autoscale_aws_fargate/#step-8-create-an-ecs-task-definition).

### Run the docker inside a AWS node passing the expected environment variables

I.e:
```
docker build --rm -t docker-gitlab-runner-fargate-driver .

docker run \
    -e GITLAB_REGISTRATION_TOKEN="my-token" \
    -e RUNNER_TAG_LIST="other-runner,test-runner" \
    -e FARGATE_CLUSTER="my-fargate-cluster" \
    -e FARGATE_REGION="us-east-1" \
    -e FARGATE_SUBNET="subnet-XYZ" \
    -e FARGATE_SECURITY_GROUP="sg-XYZ" \
    -e FARGATE_TASK_DEFINITION="my-jobs-task-definition" \
    -it docker-gitlab-runner-fargate-driver:latest
```

Important: in order to have it working properly, you must be inside a AWS node with permission to access ECS (i.e: `AmazonECS_FullAccess` role)

Note: refer to [Running Docker on AWS EC2](https://medium.com/appgambit/part-1-running-docker-on-aws-ec2-cbcf0ec7c3f8) for different ways of running docker inside an EC2 node.

## How to adapt it to run the docker in Fargate

Besides the steps 1, 2 and 3 of the previous section, you will also need a few more steps:

- Build this docker image and push it to a docker repository
- Create a Task Definition for running the docker in Fargate
- Start a new task based on this Task Definition

Bellow we describe those steps in details.

### Build this docker image and push it to a docker repository

Build the `docker-gitlab-runner-fargate-driver` docker image and push it into the repository you prefer to use.

### Create a Task Definition for running the docker in Fargate

1. Go to https://console.aws.amazon.com/ecs/home#/taskDefinitions.
1. Click Create new Task Definition.
1. Choose Fargate and click Next step.
1. Give it a name (i.e: runner-task)
1. Select values for Task memory (GB) and Task CPU (vCPU).
1. Click Add Container, then:
    - Give it a name.
    - Set the image for the one you pushed in the previous step.
    - In the environment variables field, insert  one entry for each environment variable expected by the container image (refer to the [container parameters section](#container-parameters)) and set the correct value for your project.
    - Click Add.
1. Click Create.
1. Click View Task Definition.

### Start a new task based on this Task Definition

1. Go to https://console.aws.amazon.com/ecs/home#/taskDefinitions.
1. Click on the Task Definition you created in the last step.
1. Click on the revision.
1. Click on the `Actions` button and in the menu select `Run Task`.
1. Fill the mandatory fields with the correct values.
1. Click in `Run Task`.
