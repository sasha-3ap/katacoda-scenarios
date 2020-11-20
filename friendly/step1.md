Setup private image repository

# docker registry
`helm repo add stable https://kubernetes-charts.storage.googleapis.com`{{execute}}
`helm install registry stable/docker-registry \
  --version 1.9.4 \
  --namespace kube-system \
  --set service.type=NodePort \
  --set service.nodePort=30100`{{execute}}


# Registry proxy
`helm repo add incubator http://storage.googleapis.com/kubernetes-charts-incubator`{{execute}}
`helm install registry-proxy incubator/kube-registry-proxy \
  --version 0.3.2 \
  --namespace kube-system \
  --set registry.host=registry-docker-registry.kube-system \
  --set registry.port=5000 \
  --set hostPort=5000`{{execute}}

# Registry UI
`kubectl apply -f ~/registry-ui.yaml`{{execute}}
Open it at https://[[HOST_SUBDOMAIN]]-31000-[[KATACODA_HOST]].environments.katacoda.com/



# Install tekton pipelines and triggers
`kubectl apply --filename https://storage.googleapis.com/tekton-releases/pipeline/latest/release.yaml`{{execute}}
`kubectl apply --filename https://storage.googleapis.com/tekton-releases/triggers/latest/release.yaml`{{execute}}
`kubectl get pods --namespace tekton-pipelines --watch`{{execute}}
`source <(tkn completion bash)`{{execute}}

# Install tekton CLI
`curl -LO https://github.com/tektoncd/cli/releases/download/v0.14.0/tkn_0.14.0_Darwin_x86_64.tar.gz`{{execute}}
`sudo tar xvzf tkn_0.14.0_Darwin_x86_64.tar.gz -C /usr/local/bin tkn`{{execute}}

# Install tekton dashboard
`kubectl apply --filename https://storage.googleapis.com/tekton-releases/dashboard/latest/tekton-dashboard-release.yaml`{{execute}}

expose dashboard on 30200
`kubectl apply -f tekton-dashboard-exposed.yaml`{{execute}}
Open it at https://[[HOST_SUBDOMAIN]]-30200-[[KATACODA_HOST]].environments.katacoda.com/

# Cluster configuration
Now that we have our cluster ready, we need to setup our tekton-demo namespace and RBAC. We will keep everything inside this single namespace for easy cleanup. In the unlikely event that you get stuck/flummoxed, the best course of action might be to just delete this namespace and start fresh. You should take note of the ingress subdomain, or the external IP of your cluster, you will need this for your Webhook in later steps.

## Create the tekton-demo namespace, where all our resources will live.
kubectl create namespace tekton-demo

## Create the admin user, role and rolebinding
kubectl apply -f ./rbac/admin-role.yaml

## Create the create-webhook user, role and rolebinding
kubectl apply -f ./rbac/webhook-role.yaml
This will allow our webhook to create the things it needs to.

# Install the Pipeline and Triggers
## Install the Pipeline
Now we have to install the Pipeline we plan to use and also our Triggers resources.

`kubectl apply -f pipeline.yaml`
`tkn condition list --namespace tekton-demo`
`tkn task list --namespace tekton-demo`
`tkn pipeline list --namespace tekton-demo`

## Install Triggers

- We will setup an EventListener to accept and process GitHub Push events
- A TriggerTemplate to create templated PipelineResource and PipelineRun resources per event received by the EventListener.
  - First, update the triggers.yaml file to reflect the Docker repository you wish to push the image blob to.
  - You will need to replace the DOCKERREPO-REPLACEME string everywhere it is needed.
  - Once you have updated the triggers file, you can apply it!
    `kubectl apply -f triggers.yaml`
  - Expose listener on node port 30300
    `kubectl apply -f event-listener-exposed.yaml`
If that succeeded, your cluster is ready to start handling Events.
`tkn triggerbinding list --namespace tekton-demo`
`tkn triggertemplate list --namespace tekton-demo`
`tkn eventlistener list --namespace tekton-demo`


## Add Ingress and GitHub-Webhook Tasks
We will need an ingress to handle incoming webhooks and we will make use of our new ingress by configuring GitHub with our GitHub Task.

First lets create our ingress Task.

`kubectl apply -f create-ingress.yaml -n tekton-demo`

Now lets create our webhook Task.

`kubectl apply -f create-webhook.yaml -n tekton-demo`

Check new tekton tasks are created
`tkn tasks list`


## Run Ingress Task
### Update the Ingress TaskRun
Note: If you are running on GKE, the default Ingress will not work. Instead, follow the instructions to use an Nginx Ingress here

Lets first update the TaskRun to make any needed changes

Edit the ingress-run.yaml file to adjust the settings.

At the minimum, you will need to update the ExternalDomain field to match your own DNS name.

Run the Ingress Task
When you are ready, run the ingress Task.

`kubectl apply -f ingress-run.yaml`


## Run GitHub Webhook Task
You will need to create a GitHub Personal Access Token with the following access.

public_repo
admin:repo_hook

Next, create a secret like so with your access token.

NOTE: You do NOT have to base64 encode this access token, just copy paste it in. Also, the secret can be any string data. Examples: mordor-awaits, my-name-is-bill, tekton, tekton-1s-awes0me.

`echo "Enter token?"; read token; sed "s/YOUR-GITHUB-ACCESS-TOKEN/${token}/" webhook-secret.yaml | kubectl apply -f -`


### Update webhook task run
Now lets update the GitHub Task run.

There are a few fields to change, but these fields must be updated at the minimum.

GitHubOrg: The GitHub org you are using for this getting-started.
GitHubUser: Your GitHub username.
GitHubRepo: The repo we will be using for this example.
ExternalDomain: Update this to be the to something other then demo.iancoffey.com
Run the Webhook Task
Now lets run our updated webhook task.

`kubectl apply -f webhook-run.yaml`





# Complete solution
- Installing kubernetes cluster tools https://kubernetes.io/docs/setup/production-environment/tools/
- Cluster configuration (cicd, monitoring, ...)
- 



Our Pipeline will do a few things.

Retrieve the source code
Build and push the source code into a Docker image
Push the image to the specified repository
Run the image locally
If your cluster doesn't have access to your docker registry you may need to add the secret to your Kubernetes cluster and pipeline.yaml. For that please follow the instructions here and also add

  env:
    - name: "DOCKER_CONFIG"
      value: "/tekton/home/.docker/"
to the pipeline.yaml tasks similar to this and re-apply the yaml file.

What does it do?
The Pipeline will build a Docker image with img and deploy it locally via kubectl image.

Install the TriggerTemplate, TriggerBinding and EventListener
The Triggers project will pickup from there.

We will setup an EventListener to accept and process GitHub Push events
A TriggerTemplate to create templated PipelineResource and PipelineRun resources per event received by the EventListener.
First, update the triggers.yaml file to reflect the Docker repository you wish to push the image blob to.
You will need to replace the DOCKERREPO-REPLACEME string everywhere it is needed.
Once you have updated the triggers file, you can apply it!
kubectl apply -f ./docs/tekton-demo/triggers.yaml
If that succeeded, your cluster is ready to start handling Events.
Add Ingress and GitHub-Webhook Tasks
We will need an ingress to handle incoming webhooks and we will make use of our new ingress by configuring GitHub with our GitHub Task.

First lets create our ingress Task.

kubectl apply -f ./docs/tekton-demo/create-ingress.yaml -n tekton-demo

Now lets create our webhook Task.

kubectl apply -f ./docs/tekton-demo/create-webhook.yaml -n tekton-demo

Run Ingress Task
Update the Ingress TaskRun
Note: If you are running on GKE, the default Ingress will not work. Instead, follow the instructions to use an Nginx Ingress here

Lets first update the TaskRun to make any needed changes

Edit the docs/tekton-demo/ingress-run.yaml file to adjust the settings.

At the minimum, you will need to update the ExternalDomain field to match your own DNS name.

Run the Ingress Task
When you are ready, run the ingress Task.

kubectl apply -f docs/tekton-demo/ingress-run.yaml

Run GitHub Webhook Task
You will need to create a GitHub Personal Access Token with the following access.

public_repo
admin:repo_hook
Next, create a secret like so with your access token.

NOTE: You do NOT have to base64 encode this access token, just copy paste it in. Also, the secret can be any string data. Examples: mordor-awaits, my-name-is-bill, tekton, tekton-1s-awes0me.

apiVersion: v1
kind: Secret
metadata:
  name: webhook-secret
  namespace: tekton-demo
stringData:
  token: YOUR-GITHUB-ACCESS-TOKEN
  secret: random-string-data
Update webhook task run
Now lets update the GitHub Task run.

There are a few fields to change, but these fields must be updated at the minimum.

GitHubOrg: The GitHub org you are using for this tekton-demo.
GitHubUser: Your GitHub username.
GitHubRepo: The repo we will be using for this example.
ExternalDomain: Update this to be the to something other then demo.iancoffey.com
Run the Webhook Task
Now lets run our updated webhook task.

kubectl apply -f docs/tekton-demo/webhook-run.yaml

Watch it work!
Commit and push an empty commit to your development repo.
git commit -a -m "build commit" --allow-empty && git push origin mybranch
Now, you can follow the Task output in kubectl logs.
First the image builder task.
kubectl logs -l somelabel=somekey --all-containers
Then our deployer task.
kubectl logs -l tekton.dev/pipeline=tekton-demo-pipeline -n tekton-demo --all-containers
We can see now that our CI system is working! Images pushed to this repo result in a running pod in our cluster.
We can examine our pod like so.
kubectl logs tekton-triggers-built-me -n tekton-demo --all-containers
Now we can see our new image running our cluster, after having been retrieved, tested, vetted and built, docker pushed (and pulled) and finally ran on our cluster as a Pod.

Clean up
Delete the tekton-demo namespace!
kubectl delete namespace tekton-demo
