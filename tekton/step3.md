
# Pipeline configuration
Now that we have our cluster ready, we need to setup our tekton-demo namespace and RBAC. We will keep everything inside this single namespace for easy cleanup. In the unlikely event that you get stuck/flummoxed, the best course of action might be to just delete this namespace and start fresh. You should take note of the ingress subdomain, or the external IP of your cluster, you will need this for your Webhook in later steps.

Create the tekton-demo namespace, where all our resources will live.
`kubectl create namespace tekton-demo`{{execute}}

Create the admin user, role and rolebinding
`kubectl apply -f ./rbac/admin-role.yaml`{{execute}}

Create the create-webhook user, role and rolebinding
`kubectl apply -f ./rbac/webhook-role.yaml`{{execute}}
This will allow our webhook to create the things it needs to.

Install the Pipeline
Now we have to install the Pipeline we plan to use and also our Triggers resources.

`kubectl apply -f pipeline.yaml`{{execute}}

`tkn condition list --namespace tekton-demo`{{execute}}

`tkn task list --namespace tekton-demo`{{execute}}

`tkn pipeline list --namespace tekton-demo`{{execute}}

## Triggers configuration

- We will setup an EventListener to accept and process GitHub Push events
- A TriggerTemplate to create templated PipelineResource and PipelineRun resources per event received by the EventListener.
  - First, update the triggers.yaml file to reflect the Docker repository you wish to push the image blob to.
  - You will need to replace the DOCKERREPO-REPLACEME string everywhere it is needed.
  - Once you have updated the triggers file, you can apply it!
    `kubectl apply -f triggers.yaml`{{execute}}
  - Expose listener on node port 30300
    `kubectl apply -f event-listener-exposed.yaml`{{execute}}

  Note domain `[[HOST_SUBDOMAIN]]-30300-[[KATACODA_HOST]].environments.katacoda.com` as we will need it to setup our Github webhook.

If that succeeded, your cluster is ready to start handling Events.
`tkn triggerbinding list --namespace tekton-demo`{{execute}}

`tkn triggertemplate list --namespace tekton-demo`{{execute}}

`tkn eventlistener list --namespace tekton-demo`{{execute}}


## GitHub-Webhook Tasks

Now lets create our webhook Task.

`kubectl apply -f create-webhook.yaml -n tekton-demo`{{execute}}

Check new tekton tasks are created
`tkn tasks list -n tekton-demo`{{execute}}


## Run GitHub Webhook Task
You will need to create a GitHub Personal Access Token with the following access.

public_repo
admin:repo_hook

Next, create a secret like so with your access token.

NOTE: You do NOT have to base64 encode this access token, just copy paste it in. Also, the secret can be any string data. Examples: mordor-awaits, my-name-is-bill, tekton, tekton-1s-awes0me.

`echo "Enter Github access token?"; read token; sed "s/YOUR-GITHUB-ACCESS-TOKEN/${token}/" webhook-secret.yaml | kubectl apply -f -`{{execute}}


### Update webhook task run
Now lets update the GitHub Task run.

There are a few fields to change, but these fields must be updated at the minimum.

GitHubOrg: The GitHub org you are using for this getting-started.
GitHubUser: Your GitHub username.
GitHubRepo: The repo we will be using for this example.
ExternalDomain: Update this to be the to something other then demo.iancoffey.com
Run the Webhook Task
Now lets run our updated webhook task.
`sed "s/EXTERNAL_DOMAIN/[[HOST_SUBDOMAIN]]-30300-[[KATACODA_HOST]].environments.katacoda.com/" webhook-run.yaml | kubectl apply -f -`{{execute}}

Commit and push an empty commit to your development repo.
`git checkout -b "demo-branch"; git commit -a -m "build commit" --allow-empty && git push origin mybranch`{{execute}}