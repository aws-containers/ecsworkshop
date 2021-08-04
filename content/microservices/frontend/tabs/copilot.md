---
title: "Acceptance and Production"
disableToc: true
hidden: true
---

## Deploy our application, service, and environment

Navigate to the frontend service repo.

```bash
cd ~/environment/ecsdemo-frontend
```

To start, we will initialize our application, and create our first service.
In the context of copilot-cli, the application is a group of related services, environments, and pipelines.
Run the following command to get started:

```bash
copilot init
```

We will be prompted with a series of questions related to the application, and then our service. Answer the questions as follows:

- Application name: ecsworkshop
- Service Type: Load Balanced Web Service
- What do you want to name this Load Balanced Web Service: ecsdemo-frontend
- Dockerfile: ./Dockerfile

After you answer the questions, it will begin the process of creating some baseline resources for your application and service.
This includes the manifest file for the frontend service, which defines the desired state of your service deployment. For more information on the Load Balanced Web Service manifest, see the [copilot documentation](https://github.com/aws/copilot-cli/wiki/Load-Balanced-Web-Service-Manifest).

Next, you will be prompted to deploy a test environment. An environment encompasses all of the resources that are required to support running your containers in ECS.
This includes the networking stack (VPC, Subnets, Security Groups, etc), the ECS Cluster, Load Balancers (if required), service discovery namespace (via CloudMap), and more.

Before we proceed, we need to do a couple of things for our application to work as we expect.
First, we need to define the backend services url's as environment variables as this is how the frontend will communicate with them.
These url's are created when the backend services are deployed by copilot, which is using service discovery with AWS Cloud Map.
The manifest file is where we can make changes to the deployment configuration for our service.
Let's update the manifest file with the environment variables.

```bash
cat << EOF >> copilot/ecsdemo-frontend/manifest.yml
variables:
  CRYSTAL_URL: "http://ecsdemo-crystal.ecsworkshop.local:3000/crystal"
  NODEJS_URL: "http://ecsdemo-nodejs.ecsworkshop.local:3000"
EOF
```

Next, the application presents the git hash to show the version of the application that is deployed.
All we need to do is run the below command to put the hash into a file for the application to read on startup.

```bash
git rev-parse --short=7 HEAD > code_hash.txt
```

We're now ready to deploy our environment.
Run the following command to get started:

```bash
copilot env init --name test --profile default --default-config
```

This part will take a few minutes because of all of the resources that are being created. This is not an action you run every time you deploy your service, it's just the one time to get your environment up and running.

Next, we will deploy our service!

```bash
copilot svc deploy
```

At this point, copilot will build our Dockerfile and deploy the necessary resources for our service to run.

Below is an example of what the cli interaction will look like:

![deployment](/images/copilot-frontend.gif)

Ok, that's it! By simply answering a few questions, we have our frontend service deployed to an environment!

Grab the load balancer url and paste it into your browser.

```bash
copilot svc show -n ecsdemo-frontend --json | jq -r .routes[].url
```

You should see the frontend service up and running.
The app may look strange or like it’s not working properly. This is because our service relies on the ability to talk to AWS services that it presently doesn’t have access to.
The app should be showing an architectural diagram with the details of what Availability Zones the services are running in. We will address this fix later in the chapter.
Now that we have the frontend service deployed, how do we interact with our environment and service? Let's dive in and answer those questions.

## Interacting with the application

To interact with our application, run the following in the terminal:

```bash
copilot app
```

This will bring up a help message that looks like the below image.

![app](/images/copilot-app.png)

We can see the available commands, so let's first see what applications we have deployed.

```bash
copilot app ls
```

The output should show one application, and it should be named "ecsworkshop", which we named when we ran copilot init earler.
When you start managing multiple applications with copilot, this will serve as that single command to get insight into all of them.

![app_ls](/images/copilot-app-ls.png)

Now that we see our application, let's get a more detailed view into what environments and services our application contains.

```bash
copilot app show ecsworkshop
```

The result should look like this:

![app_show](/images/copilot-app-show.png)

Reviewing the output, we see the environments and services deployed under the application.
In a real world scenario, we would want to deploy a production environment that is completely isolated from test. Ideally that would be in another account as well.
With this view, we see what accounts and regions our application is deployed to.

## Interacting with the environment

Let's now look deeper into our test environment. To interact with our environments, we will use the `copilot env` command.

![env_ls](/images/copilot-env-ls.png)

To list the environments, run:

```bash
copilot env ls
```

The response will come back with test, so let's get more details on the test environment by running:

```bash
copilot env show -n test
```

![env_show](/images/copilot-env-show.png)

With this view, we're able to see all of the services deployed to our application's test environment. As we add more services, we will see this grow.
A couple of neat things to point out here:

- The tags associated with our environment. The default tags have the application name as well as the environment.
- The details about the environment such as account id, region, and if the environment is considered production.

## Interacting with the frontend service

![env_show](/images/copilot-svc.png)

There is a lot of power with the `copilot svc` command. As you can see from the above image, there is quite a bit that we can do when interacting with our service.

Let's look at a couple of the commands:

- package: The copilot-cli uses CloudFormation to manage the state of the environment and services. If you want to get the CloudFormation template for the service deployment, you can simply run `copilot svc package`. This can be especially helpful if you decide to move to CloudFormation to manage your deployments on your own.
- deploy: To put it simply, this will deploy your service. For local development, this enables one to locally push their service changes up to the desired environment. Of course when it comes time to deploy to production, a proper git workflow integrated with CI/CD would be the best path forward. We will deploy a pipeline later!
- status: This command will give us a detailed view of the the service. This includes health information, task information, as well as active task count with details.
- logs: Lastly, this is an easy way to view your service logs from the command line.

Let's now check the status of the frontend service.

Run:

```bash
copilot svc status -n ecsdemo-frontend
```

![svc_status](/images/copilot-svc-status.png)

We can see that we have one active running task, and the details.

#### Scale our task count

One thing we haven’t discussed yet is ways to manage/control our service configuration.
This is done via the manifest file.
The manifest is a declarative yaml template that defines the desired state of our service.
It was created automatically when we ran through the setup wizard (running copilot init), and includes details such as docker image, port, load balancer requirements, environment variables/secrets, as well as resource allocation.
It dynamically populates this file based off of the Dockerfile as well as opinionated, sane defaults.

Open the manifest file (./copilot/ecsdemo-frontend/manifest.yml), and replace the value of the count key from 1 to 3. This is declaring our state of the service to change from 1 task, to 3.
Feel free to explore the manifest file to familiarize yourself.

```
# Number of tasks that should be running in your service.
count: 3
```

Once you are done and save the changes, run the following:

```bash
copilot svc deploy
```

Copilot does the following with this command:

- Build your image locally
- Push to your service's ECR repository
- Convert your manifest file to CloudFormation
- Package any additional infrastructure into CloudFormation
- Deploy your updated service and resources to CloudFormation

To confirm the deploy, let's first check our service details via the copilot-cli:

```bash
copilot svc status -n ecsdemo-frontend
```

You should now see three tasks running!
Now go back to the load balancer url, and you should see the service showing different IP addresses based on which frontend service responds to the request.
Note, it's still not showing the full diagram, we're going to fix this shortly.

#### Review the service logs

The services we deploy via copilot are automatically shipping logs to Cloudwatch logs by default. Rather than navigate and review logs via the console, we can use the copilot cli to see those logs locally.
Let's tail the logs for the frontend service.

```bash
copilot svc logs -a ecsworkshop -n ecsdemo-frontend --follow
```

Note that if you are in the same directory of the service you want to review logs for, simply type the below command. Of course, if you wanted to review logs for a service in a particular environment, you would pass the -e flag with the environment name.

```bash
copilot svc logs
```

Last thing to bring up is that you aren't limited to live tailing logs, type `copilot svc logs --help` to see the different ways to review logs from the command line.

## Create a CI/CD Pipeline

{{%expand "Expand here to deploy a pipeline" %}}

In this section, we'll go from a local development workflow, to a fully automated CI/CD pipeline and git workflow.
We will be using AWS CodeCommit to host our git repository, and the copilot cli to do the rest.

#### Prepare and setup the repository in AWS Codecommit

First, we are going to create a repository in AWS Codecommit.
Note that AWS Copilot supports GitHub and BitBucket as well.

```bash
repo_https_url=$(aws codecommit create-repository \
  --repository-name "ecsdemo-frontend" \
  --repository-description "ECSWorkshop frontend application" | jq -r '.repositoryMetadata.cloneUrlHttp')
```

Next, let's add the new repository as a remote, and push the current codebase up to our new repo.

```bash
# Configure git client to use the aws cli credential helper
git config --global credential.helper '!aws codecommit credential-helper $@'
git config --global credential.UseHttpPath true

# Add the new repo as a remote
git remote add cc $repo_https_url

# Push the changes
git push cc HEAD
```

Now that we have our repo, we can get started on the pipeline.

#### Creating the pipeline

Generally, when we create CI/CD pipelines there is quite a bit of work that goes architecting and creating them.
Copilot does all of the heavy lifting leaving you to just answer a couple of questions in the cli, and that's it.
Let's see it in action!

Run the following to begin the process of creating the pipeline:

```bash
copilot pipeline init
```

Once again, you will be prompted with a series of questions. Answer the questions with the following answers:

- Which environment would you like to add to your pipeline? Answer: Choose "test"
- Which repository would you like to use for your pipeline? Answer: Choose the repo url that matches the code commit repository. It will contain `git-codecommit.<region>`

The core pipeline files will be created in the ./copilot directory. Here is what the output should show:

![init](/images/copilot-pipeline-init.png)

Commit and push the new files to your repo. To get more information about the files that were created, check out the [copilot-cli documentation](https://github.com/aws/copilot-cli/wiki/Pipelines#setting-up-a-pipeline-step-by-step).
In short, we are pushing the deployment manifest (for copilot to use as reference to deploy the service), pipeline.yml (which defines the stages in the pipeline), and the buildspec.yml (contains the build instructions for the service).

```bash
git add copilot
git commit -m "Adding copilot pipeline configuration"
git push cc
```

Now that our repo has the pipeline configuration, let's build/deploy the pipeline:

```bash
copilot pipeline update
```

![output](/images/copilot-pipeline-output.png)

Our pipeline is now deployed (yes, it's that simple). Let’s interact with it!

Now there are two ways that I can review the status of the pipeline.

1. Console: Navigate here: https://${YOUR_REGION}.console.aws.amazon.com/codesuite/codepipeline/pipelines, click your pipeline, and you can see the stages.

2. Command line: `copilot pipeline status`.

![status](/images/copilot-pipeline-status.png)

Whether you’re in the console or checking from the cli, you will see the pipeline is actively running. You can watch as the pipeline executes, and when it is complete, all updates in the Status column will show "Succeeded".

Let's make an update to the frontend application, and push it to git.
The expected result will be the changes automatically get deployed via the pipeline.

Open the following file using your favorite text editor: `app/views/application/index.html.erb`.
Modify line 9 and prepend to the line `[Pipeline Deployed!]`.
It should look like the code below.

```ruby
[Pipeline Deployed!] Rails frontend: Hello! from <%= @az %> running <%= @code_hash %>
```

Now let's commit the file and push it to our repository.

```bash
git rev-parse --short=7 HEAD > code_hash.txt
git add app/views/application/index.html.erb code_hash.txt
git commit -m "Updating code hash and updating frontend"
git push cc
```

At this point we have handed off the deployment to the CI/CD pipeline.
In a real world scenario, it would be following good practices to have a code review as well as tests integrated into the pull request and pipeline.
It will take a few minutes for the changes to be deployed, so we can follow along in one of two ways.

1. Console: https://${AWS_REGION}.console.aws.amazon.com/codesuite/codepipeline/pipelines, click your pipeline, and you can see the stages.

2. Command line: `copilot pipeline status`.

When the pipeline is complete, navigate to the load balancer url. You should now see the application showing the Pipeline deployed in front of the Rails frontend data.

![complete](/images/copilot-frontend-pipeline-complete.png)

{{% /expand %}}

## Next steps

We have officially completed deploying our frontend. In the next section, we will extend our application by adding two backend services.
