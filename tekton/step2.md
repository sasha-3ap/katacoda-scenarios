
## Install tekton pipelines and triggers
Install tekton pielines:
`kubectl apply --filename https://storage.googleapis.com/tekton-releases/pipeline/latest/release.yaml`{{execute}}

Install tekton triggers:
`kubectl apply --filename https://storage.googleapis.com/tekton-releases/triggers/latest/release.yaml`{{execute}}

Watch as tekton is getting installed:
`kubectl get pods --namespace tekton-pipelines --watch`{{execute}}
Return to console by press CTRL-c.

## Install tekton CLI
Let's install tekton CLI. Get the tar.xz:
`curl -LO https://github.com/tektoncd/cli/releases/download/v0.14.0/tkn_0.14.0_Linux_x86_64.tar.gz`{{execute}}

Extract tkn to your PATH (e.g. /usr/local/bin):
`sudo tar xvzf tkn_0.14.0_Linux_x86_64.tar.gz -C /usr/local/bin/ tkn`{{execute}}

And add tkn command completion:
`source <(tkn completion bash)`{{execute}}

## Install tekton dashboard
Tekton has simple dashboard UI:
`kubectl apply --filename https://storage.googleapis.com/tekton-releases/dashboard/latest/tekton-dashboard-release.yaml`{{execute}}

Expose dashboard on port 30200:
`kubectl apply -f tekton-dashboard-exposed.yaml`{{execute}}

Access dashboard at https://[[HOST_SUBDOMAIN]]-30200-[[KATACODA_HOST]].environments.katacoda.com/