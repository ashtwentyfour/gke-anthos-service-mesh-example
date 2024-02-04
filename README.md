## Anthos Service Mesh Setup & Microservice Configuration

* Deploy a new [GKE Cluster](https://cloud.google.com/kubernetes-engine?_gl=1*1eynwqq*_up*MQ..&gclid=CjwKCAiAiP2tBhBXEiwACslfnpOfsoiWXnkKrtcWTF8vZ5DZzLvq3kCZTsR_zEcaNKf1PA1ftjZR9xoC2xEQAvD_BwE&gclsrc=aw.ds&hl=en#the-most-scalable-and-fully-automated-kubernetes-service)

* Set the cluster properties as environment variables on the shell:

```
$ CLUSTER_NAME=cluster-1

$ CLUSTER_ZONE=us-central1-c

$ export GCLOUD_PROJECT=$(gcloud config get-value project)

$ gcloud container clusters get-credentials $CLUSTER_NAME --zone $CLUSTER_ZONE --project $GCLOUD_PROJECT
```

* Enable and Install the Service Mesh on the above cluster:

```
$ PROJECT_ID=${PROJECT_ID:-$(gcloud config get-value project)}

$ FLEET_PROJECT_ID=${FLEET_PROJECT_ID:-$PROJECT_ID}

$ DIR_PATH=${DIR_PATH:-$(pwd)}

$ curl https://storage.googleapis.com/csm-artifacts/asm/asmcli_1.17 > asmcli

$ chmod +x asmcli

$ ./asmcli install \
  --project_id $PROJECT_ID \
  --cluster_name $CLUSTER_NAME \
  --cluster_location $CLUSTER_ZONE \
  --fleet_id $FLEET_PROJECT_ID \
  --output_dir $DIR_PATH \
  --enable_all \
  --ca mesh_ca
```

* Install the Ingress Gateway:

```
$ kubectl create namespace ingress

$ kubectl label namespace ingress istio.io/rev=$(kubectl -n istio-system get pods -l app=istiod -o json | jq -r '.items[0].metadata.labels["istio.io/rev"]') --overwrite

$ git clone https://github.com/GoogleCloudPlatform/anthos-service-mesh-packages

$ kubectl apply -n ingress -f anthos-service-mesh-packages/samples/gateways/istio-ingressgateway

$ kubectl get svc -n ingress
```

* Create the Mesh Gateway and enforce Authentication on all services in the cluster:

```
$ kubectl apply -f mesh-gateway.yml

$ kubectl apply -f mesh-peerauthentication.yml
```

* Create the Namespace for the microservice and enable auto-injection on the Namespace:

```
$ kubectl create ns census

$ export DEPLOYMENT=$(kubectl get deployments -n istio-system | grep istiod)

$ export VERSION=asm-$(echo $DEPLOYMENT | cut -d'-' -f 3)-$(echo $DEPLOYMENT | cut -d'-' -f 4 | cut -d' ' -f 1)

$ kubectl label namespace census istio.io/rev=${VERSION} --overwrite
```

$ Deploy the service from https://github.com/ashtwentyfour/sturdy-guacamole (CloudSQL will have to be provisioned)

```
$ git clone https://github.com/ashtwentyfour/sturdy-guacamole

$ kubectl apply -f sturdy-guacamole/deployment/deployment.yml
```

$ Setup the Service, Destination Rule, Virtual Service, Authentication Policy, Authorization Policy for the Namespace:

```
$ kubectl apply -f censusapp-service.yml

$ kubectl apply -f censusapp-destinationrule.yml

$ kubectl apply -f censusapp-virtualservice.yml

$ kubectl apply -f censusapp-authenticationpolicy.yml

$ kubectl apply -f censusapp-authorizationpolicy.yml
```

$ Make calls to the service via the Load Balancer (Use a valid [Auth0](https://auth0.com/) token). Requests without a token should return a 403 Error while requests with invalid tokens should return 401 Errors

```
$ curl -v http://34.123.162.31/actuator/health -H "Authorization: Bearer $TOKEN"

$ curl -v http://34.123.162.31/actuator/health -H "Authorization: Bearer $TOKEN"

$ curl -v -X POST http://34.123.162.31/data/census/add -d county=Rensselaer -d state=NY -d population=1234890 -d populationmen=500010 -d tractId=590365888 -d year=2018 -H "Authorization: Bearer $TOKEN"
```
