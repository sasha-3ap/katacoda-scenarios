  

# Pipeline configuration
Now that we have our cluster ready, we need to setup our tekton-demo namespace and RBAC. We will keep everything inside this single namespace for easy cleanup. In the unlikely event that you get stuck/flummoxed, the best course of action might be to just delete this namespace and start fresh. You should take note of the ingress subdomain, or the external IP of your cluster, you will need this for your Webhook in later steps.

Create the tekton-demo namespace, where all our resources will live:  
`kubectl create namespace tekton-demo`{{execute}}

Create the admin user, role and rolebinding:  
`kubectl apply -f ./rbac/admin-role.yaml`{{execute}}

Create the create-webhook user, role and rolebinding:  
`kubectl apply -f ./rbac/webhook-role.yaml`{{execute}}  

This will allow our webhook to create the things it needs to.

## Pipeline configuration
Now we have to install the Pipeline we plan to use and also our Triggers resources:  
`kubectl apply -f pipeline.yaml`{{execute}}  

Use Tekton CLI tool `tkn` to list newly created resources, like conditions:  
`tkn condition list --namespace tekton-demo`{{execute}}

Tasks:  
`tkn task list --namespace tekton-demo`{{execute}}  

Pipelines:  
`tkn pipeline list --namespace tekton-demo`{{execute}}  

  
## Triggers configuration
- We will setup an EventListener to accept and process GitHub Push events.
- A TriggerTemplate to create templated PipelineResource and PipelineRun resources per event received by the EventListener.  
  `kubectl apply -f triggers.yaml`{{execute}}
- Expose listener on node port 30300
  `kubectl apply -f event-listener-exposed.yaml`{{execute}}

Note domain `[[HOST_SUBDOMAIN]]-30300-[[KATACODA_HOST]].environments.katacoda.com` as we will need it to setup our Github webhook.

If that succeeded, your cluster is ready to start handling Events. Check TrigerBindings:  
`tkn triggerbinding list --namespace tekton-demo`{{execute}}  

TrigerTemplates:  
`tkn triggertemplate list --namespace tekton-demo`{{execute}}  

EventListeners:  
`tkn eventlistener list --namespace tekton-demo`{{execute}}  

  
## GitHub-Webhook Tasks
Now lets create our webhook Task:  
`kubectl apply -f create-webhook.yaml -n tekton-demo`{{execute}}  

Check new tekton tasks are created:  
`tkn tasks list -n tekton-demo`{{execute}}  

  
## Run GitHub Webhook Task
You will need to create a GitHub Personal Access Token with the following access:
- public_repo
- admin:repo_hook

Next, create a secret like so with your access token.
`echo "Enter Github access token?"; read token; sed "s/YOUR-GITHUB-ACCESS-TOKEN/${token}/" webhook-secret.yaml | kubectl apply -f -`{{execute}}

## Run the Webhook Task
Now lets run our updated webhook task.
`sed "s/EXTERNAL_DOMAIN/[[HOST_SUBDOMAIN]]-30300-[[KATACODA_HOST]].environments.katacoda.com/" webhook-run.yaml | kubectl apply -f -`{{execute}}

As a results webhook will be created in our repo and immediately send event that will trigger building process through our EventListener. You can follow Pipeline run in Tekton dashboard or using Tekton CLI:  
- get name of running pipeline: `tkn pipelinerun list -n tekton-demo`{{execute}}
- get log of running pipeline: `tkn pipelinerun logs -f <pipline-name> -n tekton-demo`{{copy}}
  
### CI/CD system in action
Configure git before using it:
`git config --global user.email "you@example.com"`{{copy}}
`git config --global user.name "Your Name"`{{copy}}

If you use 2 step authentication on Github, create PAT to authenticate.

Delete pod created by last webhook call: `kubectl delete pod tekton-triggers-built-me -n tekton-demo`{{execute}}

Commit and push an empty commit to your development repo. 
`git checkout -b "demo-branch"; git commit -a -m "build commit" --allow-empty && git push origin demo-branch`{{execute}}