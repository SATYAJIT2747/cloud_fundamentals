# AWS for ML Application Deployment --- Revision Notes

> **Goal:** Revise the AWS services and deployment concepts used to host
> a machine-learning API. These notes focus on understanding the
> complete workflow---not memorizing every AWS service.

## Table of Contents

1.  [The Big Picture](#1-the-big-picture)
2.  [Basic Vocabulary](#2-basic-vocabulary)
3.  [IAM --- Identity and Access
    Management](#3-iam--identity-and-access-management)
4.  [Regions and Availability Zones](#4-regions-and-availability-zones)
5.  [VPC --- Your Private Network](#5-vpc--your-private-network)
6.  [EC2 --- Virtual Servers](#6-ec2--virtual-servers)
7.  [Docker --- Package the
    Application](#7-docker--package-the-application)
8.  [ECR --- Store Docker Images](#8-ecr--store-docker-images)
9.  [EKS and Kubernetes --- Run and Manage
    Containers](#9-eks-and-kubernetes--run-and-manage-containers)
10. [Deployment, Service, and Load
    Balancer](#10-deployment-service-and-load-balancer)
11. [S3 --- Store Models and
    Datasets](#11-s3--store-models-and-datasets)
12. [AWS Tools and What They Do](#12-aws-tools-and-what-they-do)
13. [End-to-End ML Deployment
    Workflow](#13-end-to-end-ml-deployment-workflow)
14. [What If We Don't Use EKS?](#14-what-if-we-dont-use-eks)
15. [Interview Quick Revision](#15-interview-quick-revision)
16. [Self-Test Questions](#16-self-test-questions)

------------------------------------------------------------------------

## 1. The Big Picture

Imagine that you trained a machine-learning model that predicts whether
a customer will leave a company.

Your project contains:

-   `app.py` --- the Python API, perhaps built using Flask or FastAPI.
-   `model.pkl` --- the trained machine-learning model.
-   `requirements.txt` --- Python dependencies such as Flask, NumPy, and
    scikit-learn.
-   `Dockerfile` --- instructions for building a Docker image.

The application works on your laptop. You now want other people to
access it through an endpoint and receive predictions.

### The deployment flow

``` text
Your ML application
(app.py + model.pkl + dependencies)
             |
             v
          Docker
  Builds a container image
             |
             v
       Amazon ECR
  Stores the container image
             |
             v
        Amazon EKS
 Kubernetes runs and manages
     the application Pods
             |
             v
 Kubernetes Service / AWS Load Balancer
             |
             v
       User sends a request
             |
             v
  Flask/FastAPI → model inference
             |
             v
      Prediction returned
```

Other AWS components support this flow:

-   **IAM** controls who or what is allowed to access AWS resources.
-   **VPC** provides the network where resources communicate.
-   **EC2** can provide the virtual machines used as EKS worker nodes.
-   **S3** can store model files, datasets, and other artifacts.
-   **Security groups and routing** help control network connectivity.

### Do not confuse these three

  -----------------------------------------------------------------------
  Component                           Main purpose
  ----------------------------------- -----------------------------------
  Docker                              Packages the application and its
                                      dependencies into an image and runs
                                      containers.

  ECR                                 Stores Docker/container images so
                                      AWS workloads can retrieve them.

  EKS                                 Provides managed Kubernetes for
                                      deploying and managing
                                      containerized applications.
  -----------------------------------------------------------------------

**Memory trick:** Docker packages it → ECR stores it → EKS runs and
manages it.

------------------------------------------------------------------------

## 2. Basic Vocabulary

  -----------------------------------------------------------------------
  Term                    Simple meaning          Example
  ----------------------- ----------------------- -----------------------
  Cloud computing         Renting computing       Running your API on AWS
                          resources from a cloud  rather than keeping
                          provider instead of     your laptop switched
                          owning all the          on.
                          hardware.               

  Server                  A computer that runs    A server receives
                          software and responds   customer details and
                          to requests.            returns a prediction.

  Container               An isolated way to run  Your API runs with the
                          an application with its intended Python
                          required dependencies.  libraries.

  Networking              The connections and     Users can reach the
                          rules that determine    API, but cannot
                          how systems             directly reach private
                          communicate.            worker nodes.

  Image                   A packaged template     A Docker image
                          used to create          containing your API.
                          containers.             

  Endpoint                A network address       A URL used to send a
                          through which an        prediction request.
                          application or service  
                          can be accessed.        
  -----------------------------------------------------------------------

------------------------------------------------------------------------

## 3. IAM --- Identity and Access Management

**IAM controls who or what can perform which actions on AWS resources.**

Imagine AWS as an office building. Different people and applications
need different permissions. A developer may be allowed to deploy an
application, while an application may only need permission to read a
particular model file.

### Important IAM terms

  -----------------------------------------------------------------------
  Term                                Simple meaning
  ----------------------------------- -----------------------------------
  IAM user                            An identity that can represent a
                                      person or workload needing
                                      credentials. Long-term user
                                      credentials should be used only
                                      when appropriate.

  IAM policy                          A document that defines which
                                      actions are allowed or denied on
                                      which resources.

  IAM role                            An identity with permissions that
                                      an authorized person or service can
                                      assume. Roles are commonly used to
                                      grant temporary AWS credentials.

  IAM group                           A way to organize IAM users and
                                      assign common permissions.
  -----------------------------------------------------------------------

### Example in an ML deployment

Suppose you need to upload a Docker image to ECR and deploy an
application to EKS.

1.  You authenticate to AWS.
2.  IAM permissions determine which AWS actions you can perform.
3.  The EKS nodes or application may need their own AWS permissions.
4.  If the application downloads `model.pkl` from S3, an appropriate
    role can grant read access to that object or bucket.

**Authentication** asks, "Who are you?"\
**Authorization** asks, "What are you allowed to do?"

**Security rule:** Give identities only the permissions they need. This
is called the **principle of least privilege**. Avoid putting AWS access
keys directly in application source code.

**Interview answer:**\
"IAM is AWS's identity and access management service. It uses
identities, roles, and policies to control access to AWS resources
according to the principle of least privilege."

------------------------------------------------------------------------

## 4. Regions and Availability Zones

These concepts explain **where your AWS resources run**.

### Region

A Region is a geographical area where AWS operates infrastructure.

### Availability Zone (AZ)

An Availability Zone is an isolated location within a Region, consisting
of one or more data centers.

``` text
AWS Region
├── Availability Zone A
│   └── Data center infrastructure
└── Availability Zone B
    └── Separate data center infrastructure
```

### Why does this matter?

Running resources across multiple AZs can improve resilience if an
individual location has an infrastructure problem. It does not happen
automatically---you must design and configure the application for it.

For a small practice project, one worker node may be enough. For a
production service, you would usually consider multiple nodes and AZs.

**Interview answer:**\
"A Region is a geographical AWS location. An Availability Zone is an
isolated location within a Region. Using multiple AZs can improve
availability when the application and infrastructure are configured
appropriately."

------------------------------------------------------------------------

## 5. VPC --- Your Private Network

**VPC stands for Virtual Private Cloud.** It is a logically isolated
virtual network in AWS where you organize resources and control network
communication.

Think of a VPC as your own apartment complex. Subnets divide the space,
route tables act like navigation maps, security groups act like virtual
firewalls, and an Internet Gateway can provide a path to the internet.

### 5.1 Subnet --- A smaller network

A subnet is a smaller network inside your VPC and is associated with a
particular Availability Zone.

-   **Public subnet:** Commonly has a route to an Internet Gateway.
    Resources still need suitable public addressing and security
    configuration to be reachable from the internet.
-   **Private subnet:** Generally does not provide a direct route for
    unsolicited inbound internet traffic. It can still have controlled
    outbound connectivity, for example through a NAT Gateway when
    configured.

A subnet is not automatically public simply because you call it
"public"; its routing configuration matters.

### 5.2 Route table --- Where should traffic go?

A route table defines where network traffic should be directed.

Example: A public subnet can have a route for internet-bound traffic
through an Internet Gateway.

Without the appropriate route, traffic cannot use that path.

### 5.3 Security group --- Which connections are allowed?

A security group is a virtual firewall associated with supported AWS
resources and network interfaces. It controls inbound and outbound
traffic using rules.

Example rules for an API might include:

-   Allow HTTPS traffic on port `443` to the public-facing endpoint.
-   Restrict administrative access such as SSH on port `22` to trusted
    sources, if SSH is needed.
-   Allow only required communication between application components.

Security groups are **stateful**: response traffic for an allowed
connection is automatically permitted.

### 5.4 Internet Gateway --- A path to the internet

An Internet Gateway connects a VPC to the internet when the necessary
routes, addressing, and security rules are configured.

Attaching an Internet Gateway alone does not make every resource in the
VPC publicly accessible.

### How do these pieces work together?

1.  A user sends a request to a publicly accessible endpoint.
2.  Network routes direct traffic toward the appropriate destination.
3.  Security rules determine whether the connection is permitted.
4.  The application receives the request and generates a prediction.

**Interview answer:**\
"A VPC is a logically isolated virtual network in AWS. Subnets organize
resources, route tables determine traffic paths, security groups control
permitted connections, and an Internet Gateway can provide internet
connectivity when configured appropriately."

------------------------------------------------------------------------

## 6. EC2 --- Virtual Servers

**EC2 stands for Elastic Compute Cloud.** It provides virtual servers,
called instances, that you can use to run software.

Think of EC2 as renting a computer from AWS.

### Example

You could:

1.  Launch an EC2 instance.
2.  Set up an appropriate runtime and dependencies.
3.  Copy your application and model, or pull a Docker image.
4.  Start the API.
5.  Configure networking so authorized clients can reach it.

You can also install Docker on an EC2 instance and run containers there.

### EC2 vs Docker vs EKS

  -----------------------------------------------------------------------
  Technology                          What it is
  ----------------------------------- -----------------------------------
  EC2                                 A virtual machine/computing
                                      service.

  Docker                              A technology for building images
                                      and running containers.

  EKS                                 AWS's managed Kubernetes service
                                      for orchestrating containerized
                                      applications.
  -----------------------------------------------------------------------

With an EC2-backed EKS setup, EC2 instances can act as worker nodes, and
Pods run on those nodes. AWS manages the EKS control plane, while you
remain responsible for configuring and operating your workloads and
worker capacity.

For a small application, Docker on EC2 may be simpler than EKS. EKS
becomes useful when you want Kubernetes orchestration and its deployment
ecosystem.

**Interview answer:**\
"EC2 provides resizable virtual servers in AWS. I can run a Python ML
application directly on an instance or use EC2 instances as worker nodes
for containerized workloads in EKS."

------------------------------------------------------------------------

## 7. Docker --- Package the Application

Docker helps package an application and its dependencies into an image,
making it easier to run consistently across environments.

### Example project structure

``` text
ml-project/
├── app.py
├── model.pkl
├── requirements.txt
└── Dockerfile
```

-   `app.py`: API logic.
-   `model.pkl`: trained model, if you choose to package it inside the
    image.
-   `requirements.txt`: Python libraries.
-   `Dockerfile`: instructions for building the image.

**Important:** The model file does not have to be inside the Docker
image. You can package it inside the image, or download a version from
S3 or another model store when the application starts.

### Image vs container vs Dockerfile

  Term           Meaning
  -------------- -------------------------------------------
  Dockerfile     Instructions for building an image.
  Docker image   The packaged template.
  Container      A running instance created from an image.

### Simple command example

``` bash
docker build -t ml-app .
docker run -p 5000:5000 ml-app
```

-   `docker build` builds an image named `ml-app` from the current
    directory.
-   `docker run` starts a container from that image.
-   `-p 5000:5000` maps port 5000 on your machine to port 5000 inside
    the container.

The application must actually listen on the container's port for this
mapping to work.

**Interview answer:**\
"Docker packages an application and its dependencies into an image. A
container is a running instance of that image."

------------------------------------------------------------------------

## 8. ECR --- Store Docker Images

**Amazon ECR stands for Elastic Container Registry.** It is AWS's
managed registry for container images.

After building an image on your laptop, you need to store it somewhere
that your AWS deployment can access. ECR provides that place.

``` text
Development machine
      |
      | Build Docker image
      v
Amazon ECR
      |
      | EKS worker infrastructure retrieves image
      v
Kubernetes Pod runs the container
```

### Why not keep the image on your laptop?

Your AWS infrastructure needs to retrieve the image independently of
your laptop. A registry stores images and versions so deployment systems
can pull them when needed.

ECR integrates with AWS permissions and EKS. Other container registries
can also be used.

**Interview answer:**\
"ECR is AWS's managed container image registry. I can push a Docker
image to ECR so my EKS workloads can retrieve and run it."

------------------------------------------------------------------------

## 9. EKS and Kubernetes --- Run and Manage Containers

**Amazon EKS stands for Elastic Kubernetes Service.** It is AWS's
managed Kubernetes service.

First, understand the problem. If your ML API becomes popular, one
application instance might not be enough. You may want multiple copies,
a way to replace failed copies, and a consistent way to access the
application.

Kubernetes helps automate the deployment and management of containerized
workloads.

### 9.1 The four important terms

  -----------------------------------------------------------------------
  Term                                Simple meaning
  ----------------------------------- -----------------------------------
  Cluster                             The overall Kubernetes environment,
                                      including the control plane and the
                                      resources that run workloads.

  Control plane                       The part that exposes the
                                      Kubernetes API and coordinates
                                      cluster management. AWS manages the
                                      EKS control plane.

  Node                                A machine that runs workloads. In
                                      an EC2-backed EKS setup, worker
                                      nodes are EC2 instances.

  Pod                                 The smallest deployable Kubernetes
                                      unit. It contains one or more
                                      containers. Your API commonly runs
                                      in a container inside a Pod.
  -----------------------------------------------------------------------

### Simplified architecture

``` text
Amazon EKS cluster
│
├── Control plane (managed by AWS)
│
└── Worker nodes (often EC2 instances)
    ├── Pod 1
    │   └── ML API container
    ├── Pod 2
    │   └── ML API container
    └── Pod 3
        └── ML API container
```

This is a simplified view. A real cluster can have multiple nodes and
other supporting components.

### 9.2 What does Kubernetes do?

Suppose you declare that you want three replicas of your prediction API.

-   **Desired state:** You declare that three replicas should run.
-   **Self-healing:** If a Pod fails, Kubernetes can create a
    replacement, provided the cluster has the resources and
    configuration needed.
-   **Scaling:** You can change the replica count or configure
    autoscaling.
-   **Rolling updates:** Kubernetes can replace old application versions
    with new ones in a controlled way.

Kubernetes does not automatically make your model more accurate or make
inference faster. It manages how your application is deployed and runs.

### 9.3 EC2 vs ECS vs EKS vs Fargate

  -----------------------------------------------------------------------
  Service                             Main purpose
  ----------------------------------- -----------------------------------
  EC2                                 Provides virtual machines.

  ECS                                 AWS-native container orchestration
                                      service.

  EKS                                 Managed Kubernetes service.

  Fargate                             Serverless compute option for
                                      supported container workloads in
                                      ECS or EKS.
  -----------------------------------------------------------------------

ECS and EKS are alternatives for container orchestration. Fargate is a
compute option that can be used with either; it is not itself a
replacement for Kubernetes.

**Interview answer:**\
"EKS is AWS's managed Kubernetes service. Kubernetes runs applications
in Pods on available compute resources and helps maintain the desired
number of replicas, recover failed Pods, and support scaling and rolling
updates."

------------------------------------------------------------------------

## 10. Deployment, Service, and Load Balancer

Knowing the name EKS is not enough. You should also understand how an
application becomes reachable.

### 10.1 Deployment --- How should the Pods run?

A Kubernetes **Deployment** describes the desired state of your
application Pods.

You can specify:

-   The container image to use.
-   The desired number of replicas.
-   How updates should be performed.

Example: "I want three copies of my ML API running."

Kubernetes works to maintain that desired state.

**Remember:** Deployment = manages the desired configuration and
replicas of Pods.

### 10.2 Service --- How do we find the Pods?

Pod IP addresses can change when Pods are replaced. A Kubernetes
**Service** provides a stable network endpoint for a group of Pods and
directs traffic to the appropriate ones.

**Remember:** Service = a stable way to communicate with a group of
Pods.

### 10.3 Load Balancer --- How can users reach the application?

A load balancer can provide an accessible endpoint and distribute
incoming traffic among appropriate targets.

In EKS, a Kubernetes `Service` of type `LoadBalancer`, or an Ingress
configured with a suitable controller, can provision or use AWS
load-balancing infrastructure. The exact setup depends on your cluster
configuration.

**Remember:** Load balancer = helps expose the application and
distribute traffic.

### Example request flow

``` text
User / Client
      |
      v
AWS Load Balancer
      |
      v
Kubernetes Service
      |
      +------------+
      |            |
      v            v
    Pod 1        Pod 2
      |            |
      +------┬-----+
             v
        Flask API
             |
             v
       Load the model
             |
             v
      Generate prediction
             |
             v
       Return response
```

This is a simplified architecture; the exact network path depends on the
Service type, load-balancer configuration, and cluster networking.

### Quick distinction

  -----------------------------------------------------------------------
  Component                           Main job
  ----------------------------------- -----------------------------------
  Deployment                          Defines and maintains the desired
                                      Pods.

  Service                             Provides stable access to a group
                                      of Pods.

  Load Balancer                       Exposes the application and
                                      distributes incoming traffic across
                                      appropriate targets.
  -----------------------------------------------------------------------

**Interview answer:**\
"A Deployment manages the desired number and configuration of Pods. A
Service provides a stable endpoint for those Pods. A load balancer can
expose the application and distribute incoming traffic."

------------------------------------------------------------------------

## 11. S3 --- Store Models and Datasets

**Amazon S3 stands for Simple Storage Service.** It is cloud object
storage. You store objects, such as files and their associated metadata,
in buckets.

For an ML project, S3 can store:

-   Training and evaluation datasets.
-   A trained `model.pkl` file.
-   Model artifacts and versioned files.
-   Logs or reports exported for later analysis.

### Example: Retrieve a model from S3

``` text
Amazon S3
  └── model.pkl
          |
          | Authorized application downloads model
          v
      ML application
          |
          v
      Load model into memory
          |
          v
   Input → Prediction → Output
```

Suppose the model is large or changes independently of your application
code. You might store model versions in S3 and have the application
download the required version when it starts. Another option is to
package the model inside the Docker image.

The better choice depends on how you manage model versions, deployments,
and storage.

### Security point

Do not make a private model bucket public just to let the application
read it. Use appropriate IAM permissions so the application can access
only the resources it needs.

**Interview answer:**\
"S3 is an object storage service. In ML workflows, I can use it to store
datasets, trained models, and other artifacts, then let authorized
applications retrieve them."

------------------------------------------------------------------------

## 12. AWS Tools and What They Do

You do not need to memorize every command. First understand the purpose
of each tool.

  -----------------------------------------------------------------------
  Tool                    Purpose                 Example use
  ----------------------- ----------------------- -----------------------
  AWS Console             Web interface for AWS   Create or inspect an
                          resources.              EC2 instance.

  AWS CLI                 Manage AWS services     Interact with AWS
                          through terminal        resources and ECR.
                          commands.               

  Docker CLI              Build images and run or Build and test your ML
                          push containers.        image.

  `kubectl`               Communicate with a      Inspect Pods and apply
                          Kubernetes cluster.     Kubernetes
                                                  configuration.

  `eksctl`                Simplifies creating and Create an EKS cluster.
                          managing EKS clusters.  

  CloudFormation          Creates and manages AWS Define a repeatable set
                          infrastructure from     of AWS resources.
                          templates.              
  -----------------------------------------------------------------------

### How they can work together

1.  Use **Docker** to build and test the image.
2.  Use **AWS CLI** and the required permissions to interact with AWS
    and ECR.
3.  Use **eksctl** if you need to create an EKS cluster.
4.  Use **kubectl** to deploy the application and inspect Kubernetes
    resources.
5.  Use **CloudFormation** when you want infrastructure defined and
    managed through templates.

These tools are not all required for every deployment. For example,
`eksctl` can use CloudFormation behind the scenes to provision
infrastructure.

------------------------------------------------------------------------

## 13. End-to-End ML Deployment Workflow

Let's connect everything using a customer-churn prediction API.

### Step 1 --- Prepare the application

Train your model and save it as `model.pkl`. Create `app.py` using Flask
or FastAPI. The API accepts input, uses the model to generate a
prediction, and returns the result.

### Step 2 --- Package with Docker

Write a `Dockerfile` and build an image containing your application and
its dependencies. Decide whether to package the model inside the image
or retrieve it separately. Test the container locally.

### Step 3 --- Set up AWS access and infrastructure

Use appropriate IAM permissions. Configure the VPC, networking, and
other required AWS resources. Choose the compute and deployment approach
that suits your application.

### Step 4 --- Upload the image to ECR

Push the Docker image to your container registry so your AWS deployment
can retrieve it.

### Step 5 --- Deploy using EKS

Create or use an EKS cluster. Configure the necessary compute resources,
then define Kubernetes Deployments and Services. Kubernetes schedules
the Pods and runs the application containers using the image from ECR.

### Step 6 --- Expose the API

Configure the appropriate Service or Ingress/load-balancer setup and
network permissions. Obtain the endpoint and send a test request.

### Step 7 --- Verify the result

Confirm that the API receives input, loads or accesses the correct
model, returns predictions, and handles failures as expected.

**Key idea:** You are deploying an application that knows how to load
and use the model---not merely uploading the model file.

------------------------------------------------------------------------

## 14. What If We Don't Use EKS?

EKS is useful, but it is not automatically the best choice for every
application.

  -----------------------------------------------------------------------
  Option                  How it works            Main trade-off
  ----------------------- ----------------------- -----------------------
  EC2 + Docker            Run a Docker container  Simple for small
                          directly on a virtual   workloads, but you
                          machine.                manage more server and
                                                  deployment tasks.

  ECS + Fargate           Use ECS for             Avoids worker-server
                          orchestration and       management, but IAM,
                          Fargate for supported   networking, deployment,
                          container compute       and monitoring still
                          without managing EC2    need configuration.
                          worker servers.         

  EKS                     Use Kubernetes to       Powerful and flexible,
                          orchestrate             but introduces more
                          containerized           Kubernetes and cluster
                          applications.           concepts.
  -----------------------------------------------------------------------

For interviews, understand EKS because it is part of this workflow. Also
be able to explain why a simpler option might be sufficient for a small
API.

------------------------------------------------------------------------

## 15. Interview Quick Revision

### One-line definitions

  -----------------------------------------------------------------------
  Concept                             Revision definition
  ----------------------------------- -----------------------------------
  IAM                                 Controls identities, permissions,
                                      and access to AWS resources.

  Region                              A geographical AWS location.

  Availability Zone                   An isolated location within a
                                      Region.

  VPC                                 A logically isolated virtual
                                      network in AWS.

  Subnet                              A smaller network within a VPC,
                                      associated with an AZ.

  Route table                         Defines where network traffic
                                      should go.

  Security group                      Controls permitted inbound and
                                      outbound traffic for supported
                                      resources.

  Internet Gateway                    Provides a VPC-to-internet path
                                      when configured correctly.

  EC2                                 Provides virtual servers.

  Dockerfile                          Instructions for building a Docker
                                      image.

  Docker image                        A packaged template for creating
                                      containers.

  Container                           A running instance of an image.

  ECR                                 Stores container images.

  EKS                                 AWS's managed Kubernetes service.

  Cluster                             The overall Kubernetes environment.

  Node                                A machine that runs Kubernetes
                                      workloads.

  Pod                                 The smallest deployable Kubernetes
                                      unit, containing one or more
                                      containers.

  Deployment                          Defines the desired state and
                                      replicas of Pods.

  Service                             Provides stable network access to a
                                      group of Pods.

  Load balancer                       Exposes an application and
                                      distributes incoming traffic across
                                      targets.

  S3                                  Object storage for datasets,
                                      models, and other artifacts.

  AWS CLI                             Terminal tool for managing AWS
                                      services.

  `kubectl`                           CLI for interacting with
                                      Kubernetes.

  `eksctl`                            Tool for simplifying EKS cluster
                                      management.

  CloudFormation                      AWS infrastructure management using
                                      templates.
  -----------------------------------------------------------------------

### Explain the complete architecture in 30 seconds

"I package my ML API and its dependencies into a Docker image. I push
the image to Amazon ECR. I deploy it to an EKS cluster, where Kubernetes
runs the application in Pods and manages their desired state. A Service
provides stable access to the Pods, and a configured load balancer can
expose the API to users. IAM controls AWS permissions, VPC and security
rules control network communication, and S3 can store the model and
datasets."

### Priority for revision

**Priority 1 --- Understand well**

-   IAM users, roles, and policies.
-   EC2 and VPC fundamentals.
-   Docker image vs container.
-   ECR vs EKS.
-   Kubernetes cluster, node, Pod, Deployment, and Service.
-   Load balancer and S3.

**Priority 2 --- Know the purpose**

-   Regions and Availability Zones.
-   AWS CLI, `kubectl`, `eksctl`, and CloudFormation.
-   Basic networking and security rules.

**Learn later when needed**

-   Advanced VPC networking.
-   Advanced Kubernetes operations and autoscaling.
-   CI/CD, Terraform, and detailed ECS/Fargate configuration.

You do not need to learn every AWS service before you can explain this
deployment architecture.

------------------------------------------------------------------------

## 16. Self-Test Questions

Try answering these without looking at the notes.

1.  What is the difference between Docker, ECR, and EKS?
2.  What is the difference between an EC2 instance and a Docker
    container?
3.  What is IAM, and why should an application use an IAM role instead
    of hard-coded AWS keys?
4.  What is the difference between a VPC, a subnet, a route table, and a
    security group?
5.  What are a Kubernetes cluster, node, and Pod?
6.  What happens if you ask Kubernetes to run three replicas of an API
    and one Pod fails?
7.  What is the difference between a Deployment and a Service?
8.  What role can an AWS Load Balancer play in an EKS deployment?
9.  When might you store `model.pkl` in S3 rather than package it inside
    the Docker image?
10. What does each tool do: AWS CLI, Docker CLI, `kubectl`, `eksctl`,
    and CloudFormation?
11. Describe the path from a user request to the ML model and back.
12. When might EC2 + Docker be simpler than EKS?

### Answers for quick checking

1.  Docker packages and runs containers; ECR stores images; EKS provides
    managed Kubernetes.
2.  EC2 is a virtual machine; a container is an isolated running
    application created from an image.
3.  IAM controls permissions. Roles provide authorized access without
    embedding long-term keys in source code.
4.  VPC is the network; a subnet is a section of it; route tables define
    paths; security groups control permitted connections.
5.  A cluster is the overall environment, a node runs workloads, and a
    Pod is the smallest deployable unit.
6.  Kubernetes can create a replacement if the cluster has the required
    resources and configuration.
7.  A Deployment manages the desired state of Pods; a Service provides
    stable access to a group of Pods.
8.  It can provide a public or otherwise configured endpoint and
    distribute traffic across appropriate targets.
9.  S3 can be useful when model artifacts are large or versioned
    independently of the application image.
10. AWS CLI manages AWS services; Docker CLI builds and runs containers;
    `kubectl` manages Kubernetes resources; `eksctl` simplifies EKS
    cluster management; CloudFormation manages AWS infrastructure from
    templates.
11. A user reaches the configured endpoint, traffic is routed to the
    application, the API processes input and uses the model, then
    returns a prediction.
12. For a small application that does not need Kubernetes orchestration,
    EC2 + Docker may require fewer concepts and less cluster
    configuration.

------------------------------------------------------------------------

## Final Mental Model

Remember the purpose of each layer:

``` text
IAM          → Who is allowed to do what?
VPC          → How are AWS resources networked?
EC2          → Which virtual machines provide compute?
Docker       → How is the application packaged?
ECR          → Where is the container image stored?
EKS          → How are containerized workloads orchestrated?
Deployment   → What Pods should be running?
Service      → How do workloads find and reach Pods?
Load Balancer→ How can clients reach the application?
S3           → Where can model files and datasets be stored?
AWS tools    → How do we create, deploy, and manage the resources?
```

**Revision goal:** Be able to explain what each component does, why it
is needed, and how a request travels from the user to your ML model and
back. Understanding this connected story is more valuable than
memorizing isolated service definitions.
