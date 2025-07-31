# Deployment of the webMethods products using Helm

## Install the Helm charts

Note: we're using a fork on the offocial Helm charts, slightly adapted to support the webMethods v11.1 products.
```
helm repo add webmethods https://staillanibm.github.io/webmethods-helm-charts/charts
```

## Install the API Gateway

### Install the ECK Operator

This operator is installed system wide.
```
helm repo add elastic https://helm.elastic.co
helm repo update
helm install elastic-operator elastic/eck-operator -n elastic-system --create-namespace
```

### Create the apigateway namespace

```
kubectl create ns apigateway
```

### Create the image pull secret in this namespace

```
kubectl create secret docker-registry regcred --docker-server=${WM_CR_SERVER} \
    --docker-username=${WM_CR_USERNAME} --docker-password=${WM_CR_PASSWORD} -n apigateway
```

### Create the TLS secret for the certificate exposed by the Ingresses

This secret is referenced in the ingresses specified in the apigw-values.yaml file.
```
kubectl create secret tls tls-cert \
    --key="${TLS_PRIVATEKEY_FILE_PATH}" \
    --cert="${TLS_PUBLICKEY_FILE_PATH}" -n apigateway
```

### Modify the apigw-values.yaml file

You could possibily change the number of replicas for the API gateway, Elastic and Kibana.  
In the provided file they are are set as follows:
-   2 API Gateway pods (replicaCount = 2)
-   3 Elastic pods (elasticsearch.defaultNodeSet.count = 3)
-   1 Kibana pod (kibana.count = 1)

You might also need to change the name of the storage class used to persist Elastic data (see the elasticsearch.storageClassName attribute). By default, the allocated storage size is 1 Gb, you can change it by setting elasticsearch.storage.  

The ingresses domain names need to be changed, they are currently set to apigateway-ui.local, apigateway-rt.local and apigateway-admin.local.  

For a development sandbox, 1 pod for gateway, elastic and kibana together with 1 Gb storage is sufficient.  

### Get the deployment template

Use this command to preview the kubernetes manifests before applying them using Helm.
```
helm template api-gateway-obs -n apigateway webmethods/apigateway -f apigw-values.yaml > apigw-template.yaml
```

### Deploy the API Gateway

```
helm upgrade --install api-gateway-obs -n apigateway webmethods/apigateway -f apigw-values.yaml
```
The Administrator password can be displayed using:
```
echo "Admin Password: $(kubectl get secret --namespace apigateway api-gateway-obs-apigateway-admin-password -o jsonpath="{.data.password}" | base64 --decode)"
```

### Undeploy the API gateway (if needed)

```
helm delete api-gateway-obs -n apigateway
```

## Install the Developer Portal

### Install the ECK Operator (if not already done)

This operator is installed system wide.
```
helm repo add elastic https://helm.elastic.co
helm repo update
helm install elastic-operator elastic/eck-operator -n elastic-system --create-namespace
```

### Create the devportal namespace

```
kubectl create ns devportal
```

### Create the image pull secret in this namespace
```
kubectl create secret docker-registry regcred --docker-server=${WM_CR_SERVER} \
    --docker-username=${WM_CR_USERNAME} --docker-password=${WM_CR_PASSWORD} -n devportal
```

### Create the TLS secret for the certificate exposed by the Ingress

This secret is referenced in the ingress specified in the devportal-values.yaml file.
```
kubectl create secret tls tls-cert \
    --key="${TLS_PRIVATEKEY_FILE_PATH}" \
    --cert="${TLS_PUBLICKEY_FILE_PATH}" -n devportal
```

### Modify the devportal-values.yaml file

We have 1 dev portal pod (replicaCount = 1) and one elastic pod (elastic.defaultNodeSet.count = 1) here.  
The ingress domain name is set to devportal.local, you might need to change it.

### Get the deployment template

Use this command to preview the kubernetes manifests before applying them using Helm.
```
helm template dev-portal-obs -n devportal webmethods/developerportal -f devportal-values.yaml > devportal-template.yaml
```

### Deploy the Developer Portal
```
helm upgrade --install dev-portal-obs -n devportal webmethods/developerportal -f devportal-values.yaml
```

To check the Ignite cluster status:
```
kubectl exec -it dev-portal-obs-developerportal-0 -n devportal -- grep "Topology snapshot" /opt/softwareag/DeveloperPortal/logs/dev-portal.log | tail -n 1
```
You should see the pods in the aliveNodes array.  

The Administrator password is the default one.  

Note: the Helm chart does not include an init container that waits for the Elastic Search cluster to be up. Which means the dev portal containers will probably restart a couple of times until the Elastic datastore is ready.  

### Undeploy the Developer Portal (if needed)
```
helm delete dev-portal-obs -n devportal 
```

## Install the Microservice Runtime (MSR)

You would usually not deploy the MSR base product image "as is". The idea is to build custom images on top of this base image, with your integrations and all their dependencies.  
The following therefore only provides some guidance to manage the Helm deployment of any MSR image, whether it's the base product image or a custom image.  

### Create the integration namespace

```
kubectl create ns integration
```

### Create the image pull secret in this namespace
```
kubectl create secret docker-registry regcred --docker-server=${WM_CR_SERVER} \
    --docker-username=${WM_CR_USERNAME} --docker-password=${WM_CR_PASSWORD} -n integration
```

### Create the TLS secret for the certificate exposed by the Ingress

This secret is referenced in the ingress specified in the msr-values.yaml file.
```
kubectl create secret tls tls-cert \
    --key="${TLS_PRIVATEKEY_FILE_PATH}" \
    --cert="${TLS_PUBLICKEY_FILE_PATH}" -n integration
```

### Create a secret for the MSR

It's meant to contain the Administrator password and various connection information to external resources, such as databases or APIs.  Here we only set the Administrator password.  
```
kubectl create secret generic msr-obs \
  --from-literal=MSR_ADMIN_PASSWORD='Manage12345' \
  -n integration
```

### Modify the msr-values.yaml file

You could change the replicaCount, which is set to 1 here.  
Min and max heap size could also be adjusted depending on the integrations that need to run in the microservice (but you need to ensure the resource memory limit is higher than the max heap size).  
The ingress host name is set to msr.local, this would also have to be changed.  

### Get the deployment template

Use this command to preview the kubernetes manifests before applying them using Helm.
```
helm template msr-obs -n integration webmethods/microservicesruntime -f msr-values.yaml > msr-template.yaml
```

### Deploy the Developer Portal
```
helm upgrade --install msr-obs -n integration webmethods/microservicesruntime -f msr-values.yaml
```

The Administrator password is the one you defined earlier in the msr-obs secret (which I have set to "Manage12345" here).  
Note that is you have more than 1 pod replica, you will not be able to connect to the MSR admin console with the current ingress and service configuration. You would need to specify session affinity (like what's done for the API Gateway and Developer Portal).  This is just a basic deployment here and the good practice is to segregate the admin console ports and the service ports: session affinity to the admin console ports, but usually not for the service ports.  

### Undeploy the MSR (if needed)
```
helm delete msr-obs -n integration 
```