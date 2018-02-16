# Command your Microservices to Success with Kubernetes and Helm

## Prerequisites

### Sign up for an IBM Cloud Account

Sign up for a new account on the [IBM Cloud page](https://console.ng.bluemix.net/).

### Install the IBM Cloud CLI

Install by IBM Cloud (Bluemix) CLI - Find the appropriate installer and follow the instructions [here](https://console.bluemix.net/docs/cli/index.html#downloads).


Login to the CLI with `bx login`. When prompted, use the API endpoint `api.ng.bluemix.net`. Then target your default org and space by running the command `bx target --cf`.

You'll need to install a couple of plugins for the IBM Cloud CLI as well:

```
bx plugin install dev -r Bluemix
```

### Clone the Lab Repository

Let's clone the materials for the lab. First, `cd` to a working directory of your choice. Then jump to a terminal and type in:

> TODO
```
git clone https://github.com/svennam92/IndexK8sWorkshop.git
cd IndexK8sWorkshop
```

If this command didn't work, you can download and extract this repo as a zip file.

### Install Other Dependencies

Install the following tools and dependencies:
* [Docker CE](https://www.docker.com/community-edition)
  * Remember to launch Docker after installing.
* [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/)
* [helm](https://github.com/kubernetes/helm)
  * Need to add this to your PATH.

Verify that you've installed them properly by opening a new terminal and running the following commands. Make sure your versions are either newer or matches the ones below.

>TODO Fix docker version

```
$ docker -v
Docker version 17.09.0-ce, build afdb6d4

# Note this command will fail when checking the server version - it's expected as your CLI is not configured to connect to a server yet.
$ kubectl version
Client Version: version.Info{Major:"1", Minor:"5", GitVersion:"v1.5.6", GitCommit:"114f8911f9597be669a747ab72787e0bd74c9359", GitTreeState:"clean", BuildDate:"2017-03-28T13:54:20Z", GoVersion:"go1.7.4", Compiler:"gc", Platform:"linux/amd64"}
The connection to the server localhost:8080 was refused - did you specify the right host or port?

# Note this command will fail when trying to connect to Tiller, we'll configure this later
$ helm version
Client: &version.Version{SemVer:"v2.6.1", GitCommit:"bbc1f71dc03afc5f00c6ac84b9308f8ecb4f39ac", GitTreeState:"clean"}
Error: cannot connect to Tiller
```

## Scaffold a Microservice using the IBM Cloud CLI

Start the `bx dev` application generator:

```
bx dev create
```
> TODO add option for java app

Generate an application by following the prompts `1. Backend Service / Web App`, `3. Node`, `3. Web App - Express.js Basic`, enter a unique name and hostname, `3. No DevOps, with manual deployment`, and choosing to not add services.

```
$ bx dev create
? Select a resource type:
1. Backend Service / Web App
2. Mobile Client
Enter a number> 1

? Select a language:
1. Java - MicroProfile / Java EE
2. Java - Spring Framework
3. Node
4. Python - Flask
5. Swift
Enter a number> 3

? Select a Starter Kit or select the last option for more information:
1. Backend for Frontend - Express.js Backend
2. Microservice - Express.js Microservice
3. Web App - Express.js Basic
4. Web App - Express.js React
5. Web App - Express.js Webpack
6. Web App - MongoDb + Express + React + Node
7. Show more details
Enter a number> 3

? Enter a hostname for your project> sai-node-app
? Select from the following DevOps Toolchain and target runtime environment options:
...
1. IBM DevOps, using Cloud Foundry buildpacks
2. IBM DevOps, using Kubernetes containers
3. No DevOps, with manual deployment
Enter a number> 3

? Do you want to add services to your project? [y/n]> n

The project, my-node-app, has been successfully saved into the current directory.
```

Now the application has been scaffolded but you'll need to build/compile the app. Run the command `bx dev build`. For a node app, this command simply downloads the NPM dependencies.

Next, let's run the `bx dev run` command to execute the application in a Docker container on our local machine. The first time you run this command, this takes some time as the Docker container is being built for the first time. On future executions, only the differences in code are processed when building the container. This results in much quicker iterative runs.

>TODO Java is an option as well

Let's summarize. You first used the `IBM Cloud CLI` tool to scaffold a simple Node.js + Express.js application. You then used the `bx dev build` to install the dependencies for the application. Finally, we used the `bx dev run` command to build a Docker container for our Node.js application and execute it on our local machine. Next, we'll prepare to deploy this application to our cluster.

## Push Docker container to the Cloud

Before we can deploy our container to Kubernetes, we first need to put it somewhere that Kubernetes can access it. When deploying applications to Kubernetes, you pass in manifest files that tell Kubernetes where to fetch the image from. This ensures resiliency of your application as Kubernetes can fetch and install the latest version of your application from a cloud location, rather than your local machine.

To push our container to the cloud, [create an account on DockerHub](https://hub.docker.com/). This gives you a namespace on their registry to push your Docker images. This namespace usually matches your Docker ID.

Next, run the `docker images` command to see the Docker container you created in the previous step - it should be named something like `<project_name>-express-run` or simply `<project_name>`. Tag this image with your DockerHub namespace by prefixing your `project_name` with `<namespace>/`. For example, my tag is `svennam92/mynodeapp`.

```
docker tag <project_name>-express-run <namespace>/<project_name>
```

Note this tag as you'll need to refer to it in a later step. Next, push this image to DockerHub with the `docker push` command:

```
docker push <namespace>/<project_name>
```

Now that our container is in the cloud, proceed to configure access to the Kubernetes cluster.

## Accessing the Kubernetes Cluster

In this workshop, all attendees will be targeting the same cluster. First, you'll need to export the `KUBECONFIG` environment variable to point towards a Kubernetes cluster config. You'll find this file within the Git repository. You'll need to set KUBECONFIG to the full path to the file. As a tip, you can run `pwd` on Mac/Linux or `cd` on Windows to identify the current directory. Append the filename to this string. For example:

>TODO
```
export  KUBECONFIG=/Users/svennam/code/IndexK8sWorkshop/kube-config-dal10-IndexK8sWorkshop.yml
```

Now, `kubectl` should be properly configured to connect with the cluster. To verify this, run `kubectl cluster-info`. You should see something like the following:

```
$ kubectl cluster-info
Kubernetes master is running at https://169.46.7.238:20227
Heapster is running at https://169.46.7.238:20227/api/v1/namespaces/kube-system/services/heapster/proxy
KubeDNS is running at https://169.46.7.238:20227/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

Finally, to avoid stepping on each others toes as you run through the lab, create a custom namespace for yourself and make a note of it:

```
$ mynodeapp kubectl create namespace sai-space
namespace "sai-space" created
$ mynodeapp kubectl config set-context $(kubectl config current-context) --namespace=sai-space
Context "IndexK8sWorkshop" modified.
$ mynodeapp kubectl get namespaces
NAME             STATUS    AGE
default          Active    10h
...
sai-space        Active    29s
...
```

Next, let's work with Helm to deploy our microservice application to your namespace within the cluster.

## Streamlining Kubernetes Deployments with Helm

Helm is a tool that allows you to manage Kubernetes "charts". Charts are pre-configured Kubernetes resources. To deploy your application to the cluster, you'll be using a Helm chart.

First you need to initialize the Helm installation. Run the `helm init` command:

```
$ helm init
$HELM_HOME has been configured at /Users/svennam/.helm.
Warning: Tiller is already installed in the cluster.
(Use --client-only to suppress this message, or --upgrade to upgrade Tiller to the current version.)
Happy Helming!
```
_Note: The warning appears because the server-side component of Helm, "Tiller", has already been installed. You still need to run the command to initialize Helm locally._

Next, let's take a look at our Helm manifest files that the IBM Cloud CLI has already generated for us. `cd` to the `chart/<project_name>` directory. Here, you should something similar to the following files

```
Chart.yaml
values.yaml
templates
 - basedeployment.yaml
 - deployment.yaml
 - hpa.yaml
 - istio.yaml
 - service.yaml
```

The `Chart.yaml` file defines the basic descriptors of the Helm chart. The `values.yaml` file contains all the vital attributes that populate the `.yaml` files in the `templates` folder. The two crucial template files that are enabled by default are the `deployment.yaml` and `service.yaml` files. The `deployment.yaml` file tells Kubernetes where to fetch the container image from, what port the application receives traffic on, defines some environment variables and configures some other reasonable defaults for your image. The `service.yaml` file creates a Service component in Kubernetes so that inbound traffic can be routed to your application.

### Update Container Image Reference

In the `values.yaml` file, you should see the `repository:` and `tag:` attributes. Update both of these fields to the Docker image you tagged above -- `<namespace>/<project_name>` and `latest` respectively. For example, mine looks like this:

```
image:
  repository: svennam92/mynodeapp
  tag: latest
```

### Deploy your Helm Chart

Now that you've completed the prerequisite steps, you're ready to deploy the application to the cluster. `cd` to the top-level of your project directory again and run `helm install chart/<project_name>`. You should see the following output:

>TODO

In a few minutes, your application should be deployed and ready to access. Let's launch the Kuberentes Dashboard to track the progress. First, get the token you need for authenticating with the dashboard:

```
$ kubectl config view -o jsonpath='{.users[0].user.auth-provider.config.id-token}'
  <COPY TOKEN>
```

Next, run `kubectl proxy` and access the dashboard at `localhost:8001/ui`. When it prompts you to log-in, use the `Token` option and paste the output from above. You should see something like this:

> TODO

First, switch to the namespace you created earlier using the drop-down box on the left. Depending on how long it has been, the Pod should either show as `Ready` or still starting. Once it shows as `Ready`, proceed to the next step.

## Access your Application

The public IP for our cluster is `#TODO`. However, we need to know the port that our particular application is accessible on. This is exposed through the `Service` we created through the Helm chart. Run `kubectl get service`:

```
$ kubectl get service
NAME               TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)                         AGE
mynewapp-service   NodePort   172.21.75.73   <none>        9080:30480/TCP,9443:32482/TCP   5h
```

Copy the port associated with the 9080 port. In this example, we want "30480" from "9080:30480/TCP". Then, open a browser and access `#TODO:<service_port>`. In my example, I would access `#TODO:30480`. 

That's it! You should see a splash page for the microservice you deployed. You've just deployed your first microservice to an IBM Cloud Kubernetes cluster!

Now that you've deployed the microservices, switch hats to a typical Operations engineer who would manage these deployments.

## Managing your Kubernetes Cluster

Let's walkthrough the Kubernetes cluster dashboard. In the previous step, you ran `kubectl proxy` to expose access to the Kubernetes cluster dashboard at `localhost:8001/ui`. Access that now.

### Nodes

By clicking on the `Nodes` tab along the left, you'll see each of the worker machines (VMs) that are in your cluster. Since we used a "Lite" Kubernetes deployment, there's a single machine assigned to your cluster. Most of the time, you'll never actually have to work with your physical nodes. Kubernetes manages them for you. However, it's useful to click into your nodes to see how much of the CPU and memory is being allocated.

![Nodes](images/nodes.png)

### Workloads

The `Workloads` section allows you to directly manage orchestration of your microservices. This ranges from inspecting the `pods` that represent each instance of your microservices to deployments and replica sets responsible for actually spinning those `pods` up.

In the `Pods` section, you'll see the list of pods that are in your cluster. A Pod is the smallest deployment unit that Kubernetes allows your to manage - it's usually made up of one but can be more containers. These containers share storage and network within the pod. The best practice is to have each pod encompass a single capability or feature. In our case, each microservice is a pod of its own. You can also inspect the logs on each pod by hitting the `Logs` icon on each Pod.

In the `Deployments` section, you'll see a list of all the deployments that Kubernetes is managing for you. A deployment tells Kubernetes how many replicas of each pod you need (horizontal scaling), where to actually fetch the images (registry) that make up your pod and even lets you scale the pods directly through the UI.

![Deployments](images/deploy.png)

### Discovery and Load Balancing

The `Discovery and Load Balancing` section is crucial to microservice development. `Services` simplify how your containers find and talk to each other. As just mentioned, Kubernetes will handle scaling your pods horizontally. However, you have to consider that now each pod has a different IP address. With Pods being ephemeral, it's not reasonable to manage all of these IPs. Because of this, a load balancer is automatically setup to route requests to each of these scaled pods. This is one of the biggest advantages in Kubernetes. Access to each of your microservices is simplified to the "name" of the service you setup for it. This means your application code doesn't need to know the IP or URL of each of microservice in your cluster - it simply needs the service name. The microservices can access each other using the service name, which can be as simple as "schedule" or "session".

The `Ingresses` section is used for a number of things, but primarily used for creating an externally-accessible URL for your cluster. Your front-end web application has an Ingress so you can access it from the public web.

### Config and Storage

The `Config and Storage` section allows you to manage sets of config that your microservices use, manage persistent volumes as well as secrets.

You can edit the configuration live in your cluster in the `Config Maps` view. This means you don't need to redeploy your apps to use the latest configuration values.

The `Secrets` tab allows you to manage things like SSH keys, API Keys, tokens and more. They are securely accessible from the containers that are running in your cluster.

## Next Steps and More Resources
