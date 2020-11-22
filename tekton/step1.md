
## Git repo
Throughout this workshop we will need configuration files to configure our Kubernetes cluster and later Tekton. Intentionally we keep our configuration togather with app ulmacea source code. Let's clone the repo and change into folder cluster:
`git clone https://github.com/sashamirkovic/ulmaceae.git; cd ulmaceae/cluster`{{execute}}

Next, let's setup private image repository as we will need it for our CI/CD.

## Docker registry
Add stable repo:
`helm repo add stable https://kubernetes-charts.storage.googleapis.com`{{execute}}

Install docker registry:

`helm install registry stable/docker-registry \
  --version 1.9.4 \
  --namespace kube-system \
  --set service.type=NodePort \
  --set service.nodePort=30100`{{execute}}


## Registry proxy
Add helm repo
`helm repo add incubator http://storage.googleapis.com/kubernetes-charts-incubator`{{execute}}

Install registry proxy:
`helm install registry-proxy incubator/kube-registry-proxy \
  --version 0.3.2 \
  --namespace kube-system \
  --set registry.host=registry-docker-registry.kube-system \
  --set registry.port=5000 \
  --set hostPort=5000`{{execute}}

## Registry UI
Install registry UI:
`kubectl apply -f registry-ui.yaml`{{execute}}

Open registry ui at https://[[HOST_SUBDOMAIN]]-31000-[[KATACODA_HOST]].environments.katacoda.com/
