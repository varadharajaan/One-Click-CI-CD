
**_PROJECT IMPLEMENTATION:_** 

![modules Infra](img/ecs-terraform-modules.png)

**Terraform project is composed of the following structure:**

`~~├── modules
│ └── code_pipeline 
│ └── ecs 
│ └── networking 
│ └── rds
├── pipeline.tf
├── production.tf
├── production_key.pub
├── terraform.tfvars
└── variables.tf~~`

![Network Infra](img/ecs-infra.png)

_``Modules are where we will store the code that handles the creation of a group of resources. It can be reused by all environments (Production, Staging, QA, etc.) without needing to duplicate a lot of code.
production.tf is the file that defines the e****nvironment itself. It calls the modules passing variables to it.
pipeline.tf Since the pipeline can be a global resource without needing to isolate per environment. This file will handle the creation of this pipeline using the `code_pipeline` module.``_

**`Creating a VPC (Virtual Private Cloud)`**

`_The first thing that we need to create is the VPC with 2 subnets (1 public and 1 private) in each Availability Zone. Each Availability Zone is a geographically isolated region. Keeping our resources in more than one zone is the first thing to achieve high availability. If one physical zone fails for some reason, your application can answer from the others._
`

`Keeping the cluster on the private subnet protects your infrastructure from external access. The private subnet is allowed only to be accessed from resources inside the public network (In our case, will be the Load Balancer only).
Here tf apply creates, 4 subnets (2 public and 2 private) in each Availability zone. It also creates a NAT to allow the private network access the internet.`

**_`The Database`_**

`We will create a RDS database. It will be located on the private subnet. Allowing only the public subnet to access it.
we create the RDS resource with values received from the variables. We also create the security group that should be used by resources that want to connect to the database (in our case, the ECS cluster). Ok. Now we have the database. Let’s finally create our ECS to deploy our app.`

**_`**The ECR**`_**

`The first thing is to create the repository to store our built images.
this is the part that we define the ECS resources needed for our app.`

`_**The ECS cluster**_`
`Next, we need our ECS cluster. Even using Fargate (that doesn’t need any EC2), we need to define a cluster for the application.
`

![ECS- task Definition](img/deployment.png)

**_`The tasks definitions`_**
`Now, we will define 2 task definitions.
Web: Contains the definition of the web app itself.
Db Migrate: This task will only run the command to migrate our database and will die. Since it is a single run task, we don’t need a service for it.
The tasks definitions are configured in a JSON file and rendered as a template in Terraform.
we are defining the task to ECS. We pass the created ECR image repository as variable to it. We also configure other variables so ECS can start our Rails app. 
The definition of the DB migration task is almost the same. We only change the command that will be executed.`

**_`The load balancers`_**

![Load Balancers](img/alb.png)

**_`Before creating the Services, we need to create the load balancers. They will be on the public subnet and will forward the requests to the ECS service.
we define that our target group will use HTTP on port 80. We also create a security group to allow access into the port 80 from the internet. After, we create the Application Load Balancer and the listener. To use Fargate, you should use an Application Load Balancer instead an Elastic Load Balancer.**_
`
**_`Finally, the ECS service`_**

Now we will create the service. To use Fargate, we need to specify the lauch_type as Fargate.

**_`Auto-scaling`_**

_`Fargate allows us to auto-scale our app easily. We only need to create the metrics in CloudWatch and trigger to scale it up or down.
We create 2 auto scaling policies. One to scale up and other to scale down the desired count of running tasks from our ECS service.
After, we create a CloudWatch metric based on the CPU. If the CPU usage is greater than 85% from 2 periods, we trigger the alarm_action that calls the scale-up policy. If it returns to the Ok state, it will trigger the scale-down policy.`_

**_`The Pipeline to deploy our app`_**

![Code Pipeline](img/code_pipeline_1.png)

![Code Pipeline](img/code_pipeline_2.png)

_`Our infrastructure to run our Docker app is ready. But it is still boring to deploy it to ECS. We need to manually push our image to the repository and update the task definition with the new image and update the new task definition. We can run it through Terraform, but it could be better if we have a way to push our code to Github in the master branch and it deploys automatically for us.
Entering, CodePipeline and CodeBuild.
CodePipeline is a Continuous Integration and Continuous Delivery service hosted by AWS.
CodeBuild is a managed build service that can execute tests and generate packages for us (in our case, a Docker image).
With it, we can create pipelines to delivery our code to ECS. The flow will be:
You push the code to master’s branch
CodePipeline gets the code in the Source stage and calls the Build stage (CodeBuild).
Build stage process our Dockerfile building and pushing the Image to ECR and triggers the Deploy stage
Deploy stage updates our ECS with the new image`_


_``pre_build: Upgrade aws-cli, set some environment variables: REPOSITORY_URL with the ECR repository and IMAGE_TAG with the CodeBuild source version. The ECR repository is passed as a variable by Terraform.
build: Build the Dockerfile from the repository tagging it as LATEST in the repository URL.
post_build: Push the image to the repository. Creates a file named imagedefinitions.json with the following content: 
‘[{“name”:”web”,”imageUri”:REPOSITORY_URL”}]’
This file is used by CodePipeline to upgrade your ECS cluster in the Deployment stage.
artifacts: Get the file created in the last phase and uses as the artifact.
After, we create a CodePipeline resource with 3 stages:
Source: Gets the repository from Github (change it by your repository information) and pass it to the next stage.
Build: Calls the CodeBuild project that we created in the step before.
Production: Gets the artifact from Build stage (imagedefinitions.json) and deploy to ECS.
``_
**_`Zero Downtime Deployment`_**

**_`point your DNS record to the new load balancer using an ALIAS`_**

![DNS  Name](img/alb_name.png)

**What is Blue/Green deployment?**

![Blue - Green - Infra](img/blue_green.png)

_`Blue/Green deployment is a DevOps practice that aims to reduce downtime on updates by creating a new copy of the desired component, while maintaining the current. 
Given that, you end with two versions of the system: One with the actual version (blue) and another with a newer one (green). When the new version is up and running, 
you can seamlessly switch traffic to it. This is useful not only to reduce downtime, but also to improve rollback time when something bad happens.`_

**Blue/Green Infrastructure**

_`While Blue/Green deployment is a technique more commonly used with application deployment, the reduced costs of the cloud, in conjunction with the tools we have right now, make possible to have two copies of an entire cloud infrastructure with little to no pain.

It is important to note that doing Blue/Green deployment of an entire Cloud Infrastructure is not a silver bullet and certainly a bit too much if you are doing small changes (for example, adding a new EC2 Instance to your stack). But for major/breaking changes is a win and I personally recommend it.`_

**_`Terraform Exceutions`_**

_`terraform init`_

will initalise  the cloud provider and download latest version of api's. In our case it is AWS provider

![terraform int](img/init.png)

**_`terraform plan`_**

will do planning and does show how the env looks after exceution.

![terraform plan](img/plan.png)

**_`terraform apply`_**

will read tfstate file and create the needed provisioning in the cloud

![terraform apply](img/apply.png)

_**`terraform destroy`**_

will destroy complete infra which is created earlier by terraform apply.

 







``