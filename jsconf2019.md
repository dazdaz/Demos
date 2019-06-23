## Preparation for linkerd demo, debugging a node.js microservice application
* Credit given to Brian Redmond for this demo -> 
  * https://blog.linkerd.io/2018/11/02/debugging-node-js-on-kubernetes-with-linkerd/

### Creating a container with a node.js app
```
MYUSER=myname
git clone https://github.com/dazdaz/pnew
cd pnew
docker build -t $MYUSER/pnew .
docker images $MYUSER/pnew
docker run -p 8888:3000 -d $MYUSER/pnew
az vm open-port --port 8888 --resource-group u1804-rg --name ubuntuvm
curl ifconfig.co
Point web browser to : <External_IP>:8888
```

### Using draft
```
draft config set registry $MYUSER.azurecr.io
az acr login --name $MYUSER
draft init
draft create
draft up
helm list –a
draft connect
draft delete
docker container stop $(docker container ls -aq)
```

### Deploying Linkerd 2.3.2
```
curl -sL https://run.linkerd.io/install | sh
export PATH=$PATH:$HOME/.linkerd2/bin
linkerd version
linkerd check --pre
linkerd install | kubectl apply -f -
linkerd check
kubectl -n linkerd get deploy
```

* Download linkerd CLI
* https://github.com/linkerd/linkerd2/releases


### Deploy and mesh your application in the namespace: dazns
```
kubectl apply -f deployment-nginx-dazns.yaml
kubectl get ns
kubectl get pods,svc -n dazns
SVCIPNGINX=$(kubectl get service nginx-svc -n $APPNS -o jsonpath='{.status.loadBalancer.ingress[*].ip}')
curl $SVCIPNGINX
kubectl get -n dazns deploy/nginx -o yaml | linkerd inject - | kubectl apply -f -
linkerd stat ns/dazns
linkerd tap ns/dazns
# kubectl get deploy ns/dazns -o yaml | linkerd uninject - | kubectl apply -f -
```

### Generate synthetic workload - wrk is a modern HTTP benchmarking tool
```
git clone https://github.com/wg/wrk
cd wrk ; make
# This runs a benchmark for 30 seconds, using 12 threads, and keeping 400 HTTP connections open.
sudo cp wrk /usr/local/bin
wrk -t12 -c400 -d30s http://${SVCIPNGINX}/index.html
```

### node.js microservice based app
*	Show node.js/microservice app on k8s -> kubectl get,svc -n hackfest   # show API's running as pods on K8s
*	Show node.js via web browser -> kubectl get pods,svc -n hackfest  # goto http://<IP>:8080/ to show app running
*	Show node.js in mesh (Linkerd) -> linkerd tap deploy/weather-api -n hackfest   # click on weather API


### Linkerd Dashboard
```
LDWEB=$(kubectl get pods --selector=linkerd.io/control-plane-component=web -n linkerd -o=jsonpath='{.items[0].metadata.name}')
kubectl port-forward -n linkerd $LDWEB 8084:8084
```

* Linkerd Dashboard
* http://localhost:8084/overview

* Grafana
* http://localhost:8084/grafana/?refresh=5s


## Deploying the node.js microservice app for the demo
* Code from : https://github.com/Azure/kubernetes-hackfest

```
UNIQUE_SUFFIX=cosmosmyusername
APPINSIGHTSNAME=appInsightshackfest$UNIQUE_SUFFIX
RGNAME=dazaks-rg
# Deploy the appinsights ARM template   
az group deployment create --resource-group $RGNAME --template-file ~/kubernetes-hackfest/labs/build-application/app-Insights.json \
 --parameters type=Node.js name=$APPINSIGHTSNAME regionId=southeastasia

export ACRNAME=myacr
export COSMOSNAME=cosmos$UNIQUE_SUFFIX
# Check COSMOS Name
echo $COSMOSNAME
# Persist for Later Sessions in Case of Timeout
echo export COSMOSNAME=cosmos$UNIQUE_SUFFIX >> ~/.bashrc
# Create Cosmos DB
az cosmosdb create --name $COSMOSNAME --resource-group $RGNAME --kind MongoDB

# Set the CosmosDB user and password
export MONGODB_USER=$(az cosmosdb show --name $COSMOSNAME --resource-group $RGNAME --query "name" -o tsv)
export MONGODB_PASSWORD=$(az cosmosdb list-keys --name $COSMOSNAME --resource-group $RGNAME --query "primaryMasterKey" -o tsv)

# Find AppInsights
az resource list --namespace microsoft.insights --resource-type components --query [*].[id] --out tsv
/subscriptions/36d4fd22-fafb-4444-8888-888888888888/resourceGroups/dazaks-rg/providers/microsoft.insights/components/appInsightshackfestdevans26547
# Show the Instrumentation Key
APPINSIGHTS_INSTRUMENTATIONKEY=$(az resource show --id "/subscriptions/36d4fd22-fafb-4444-8888-888888888888/resourceGroups/dazaks-rg/providers/microsoft.insights/components/appInsightshackfestdevans26547" --query properties.InstrumentationKey --o tsv)

kubectl create ns hackfest
kubectl create secret generic cosmos-db-secret --from-literal=user=$MONGODB_USER --from-literal=pwd=$MONGODB_PASSWORD \ --from-literal=appinsights=$APPINSIGHTS_INSTRUMENTATIONKEY -n hackfest

az acr build -t hackfest/data-api:1.0 -r $ACRNAME --no-logs ./kubernetes-hackfest/app/data-api
az acr build -t hackfest/flights-api:1.0 -r $ACRNAME --no-logs ./kubernetes-hackfest/app/flights-api
az acr build -t hackfest/quakes-api:1.0 -r $ACRNAME --no-logs ./kubernetes-hackfest/app/quakes-api
az acr build -t hackfest/weather-api:1.0 -r $ACRNAME --no-logs ./kubernetes-hackfest/app/weather-api
az acr build -t hackfest/service-tracker-ui:1.0 -r $ACRNAME --no-logs ./kubernetes-hackfest/app/service-tracker-ui

### You can see the status of the builds by running the command below.
az acr task list-runs -r $ACRNAME -o table
az acr task logs -r $ACRNAME --run-id aa1

perl -i -p -e 's/youracr/$ACRNAME/g;' kubernetes-hackfest/charts/data-api/values.yaml
perl -i -p -e 's/youracr/$ACRNAME/g;' kubernetes-hackfest/charts/quakes-api/values.yaml
perl -i -p -e 's/youracr/$ACRNAME/g;' kubernetes-hackfest/charts/weather-api/values.yaml
perl -i -p -e 's/youracr/$ACRNAME/g;' kubernetes-hackfest/charts/flights-api/values.yaml
perl -i -p -e 's/youracr/$ACRNAME/g;' kubernetes-hackfest/charts/service-tracker-ui/values.yaml

helm upgrade --install data-api ./kubernetes-hackfest/charts/data-api --namespace hackfest
helm upgrade --install quakes-api ./kubernetes-hackfest/charts/quakes-api --namespace hackfest
helm upgrade --install weather-api ./kubernetes-hackfest/charts/weather-api --namespace hackfest
helm upgrade --install flights-api ./kubernetes-hackfest/charts/flights-api --namespace hackfest
helm upgrade --install service-tracker-ui ./kubernetes-hackfest/charts/service-tracker-ui --namespace hackfest

kubectl get pod,svc -n hackfest
kubectl get service service-tracker-ui -n hackfest
```
* Goto http://<IP>:Port 8080
* http://52.187.100.100:8080/#/


* https://github.com/Azure/kubernetes-hackfest/blob/master/labs/servicemesh/linkerd/README.md
* Use helm template to create manifest for injection
```
mkdir mesh-inject
helm template ./kubernetes-hackfest/charts/data-api > mesh-inject/data-api.yaml
helm template ./kubernetes-hackfest/charts/flights-api > mesh-inject/flights-api.yaml
helm template ./kubernetes-hackfest/charts/quakes-api > mesh-inject/quakes-api.yaml
helm template ./kubernetes-hackfest/charts/weather-api > mesh-inject/weather-api.yaml
helm template ./kubernetes-hackfest/charts/service-tracker-ui > mesh-inject/service-tracker-ui.yaml

# Re-deploy application using linkerd inject
linkerd inject mesh-inject/data-api.yaml | kubectl apply -n hackfest -f -
linkerd inject mesh-inject/flights-api.yaml | kubectl apply -n hackfest -f -
linkerd inject mesh-inject/quakes-api.yaml | kubectl apply -n hackfest -f -
linkerd inject mesh-inject/weather-api.yaml | kubectl apply -n hackfest -f -
linkerd inject mesh-inject/service-tracker-ui.yaml | kubectl apply -n hackfest -f -
```


## Sequence of demos
#### Slide 16
*	build a simple node.js container image
*	run a simple node.js container image on VM -> http://20.184.25.139:8888
*	demo draft

#### Slide 20
*	Deploy nginx on k8s
*	Show linkerd GUI -> http://localhost:8084/overview
*	mesh nginx
*	Generate synthetic workload using wrk
*	Show RPS in Grafana

#### Slide 21
*	Show node.js/microservice app on k8s -> kubectl get,svc -n hackfest   # show API's running
*	Show node.js via web browser -> kubectl get pods,svc -n hackfest  # goto http://<IP>:8080/ to show app running
*	Show node.js in mesh (Linkerd)
*	linkerd tap deploy/weather-api -n hackfest   # click on weather API

### The weather API called the Data API and you can see both the API path and the response that you got (200 ot 404)
:method=POST :authority=data-api.hackfest.svc.cluster.local:3009
status=200


### Create a 404
```
vi kubernetes-hackfest/app/data-api/routes/api.js
:62
  // getDataObjFromDb(Weather, req.params.timestamp, (err, data) => {
  res.status(404).end()
```

```
ARNAME=myacr
az acr build -t hackfest/data-api:1.0 -r $ACRNAME --no-logs ./kubernetes-hackfest/app/data-api
az acr build -t hackfest/data-api:1.2 -r $ACRNAME --no-logs ./kubernetes-hackfest/app/data-api
# update container image to 1.2
vim mesh-inject/data-api.yaml
linkerd inject mesh-inject/data-api.yaml | kubectl apply -n hackfest -f -
```


### Deploying tmux
```
Ubuntu 1804 comes with tmux 2.6 which is old
sudo apt-get remove tmux
sudo apt-get install libevent-dev libncurses5-dev
wget https://github.com/tmux/tmux/releases/download/2.9a/tmux-2.9a.tar.gz
cd tmux-2.9a
./configure && make
sudo cp tmux /usr/bin
$ tmux -V
tmux 2.9a
```

### Using tmux
* Split screen horizontally: Ctrlb and Shift"
* Toggle between panes: Ctrlb and o
* Close current pane: Ctrlb and x

```
>>>To copy and paste across panes is done like this :
ctrl [
Highlight text and do : ctrl space
>>>Then to paste:
ctrl ]

tmux ls
# re-attach to existing session
tmux a #
# attach to sessoin #0
tmux attach -t 0

alt up
alt down
or use mouse
```
