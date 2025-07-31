# Deployment of the webMathods API Management products using Helm

## Clone the Github repo
```
cd $HOME/git
git clone https://github.com/staillanibm/webmethods-helm-charts.git
cd $OLDPWD
```

## Install the API Gateway

### Fetch the chart dependencies
```
cd $HOME/git/webmethods-helm-charts/apigateway/helm
helm dependency build
cd $OLDPWD
```

### Modify the apigw-values.yaml file

You could possibily change the number of replicas for the API gateway, Elastic and Kibana.  
In the provided file they are are set as follows:
-   2 API Gateway pods (replicaCount = 2)
-   3 Elastic pods (elasticsearch.defaultNodeSet.count = 3)
-   1 Kibana pod (kibana.count = 1)

You might also need to change the name of the storage class used to persist Elastic data (see the elasticsearch.storageClassName attribute). By default, the allocated storage size is 1 Gb, you can change it by setting elasticsearch.storage. 

For a development sandbox, 1 pod for gateway, elastic and kibana together with 1 Gb storage is sufficient.  

### Get the deployment template
Use this command to preview the kubernetes manifests before applying them using Helm.
```
helm template api-gateway-obs -n apigateway $HOME/git/webmethods-helm-charts/apigateway/helm -f apigw-values.yaml > apigw-template.yaml
```

### Install the ECK Operator
```
helm repo add elastic https://helm.elastic.co
helm repo update
helm install elastic-operator elastic/eck-operator -n elastic-system --create-namespace
```

### Create the image pull secret
```
kubectl create secret docker-registry regcred --docker-server=${WM_CR_SERVER} \
    --docker-username=${WM_CR_USERNAME} --docker-password=${WM_CR_PASSWORD} -n apigateway
```

### Deploy the API Gateway
```
helm upgrade --install api-gateway-obs -n apigateway $HOME/git/webmethods-helm-charts/apigateway/helm -f apigw-values.yaml
```

### Undeploy the API gateway (if needed)
```
helm delete api-gateway-obs -n apigateway
```

## Install the Developer Portal

### Fetch the chart dependencies
```
cd $HOME/git/webmethods-helm-charts/developerportal/helm
helm dependency build
cd $OLDPWD
```

### Modify the devportal-values.yaml file

We have 1 dev portal pod (replicaCount = 1) and one elastic pod (elastic.defaultNodeSet.count = 1) here.  

### Get the deployment template
Use this command to preview the kubernetes manifests before applying them using Helm.
```
helm template dev-portal-obs -n devportal $HOME/git/webmethods-helm-charts/developerportal/helm -f devportal-values.yaml > devportal-template.yaml
```

### Install the ECK Operator (if not already done)
```
helm repo add elastic https://helm.elastic.co
helm repo update
helm install elastic-operator elastic/eck-operator -n elastic-system --create-namespace
```

### Create the image pull secret
```
kubectl create secret docker-registry regcred --docker-server=${WM_CR_SERVER} \
    --docker-username=${WM_CR_USERNAME} --docker-password=${WM_CR_PASSWORD} -n devportal
```

### Deploy the Developer Portal
```
helm upgrade --install dev-portal-obs -n devportal $HOME/git/webmethods-helm-charts/developerportal/helm -f devportal-values.yaml
```
Note: the Helm chart does not include an init container that waits for the Elastic Search cluster to be up. Which means the dev portal containers will probably restart a couple of times until the Elastic datastore is ready.

### Undeploy the Developer Portal (if needed)
```
helm delete dev-portal-obs -n devportal 
```