# Enviroment Variables In Kubernetes

#### we have 3 methods to make enviroment variables : -
1. pass it normailly in the object creation yaml file
2. make a configMap and add the enviroment variables in it normally, then include that file in our object (we can select specific enviroment from the configMap or add it all)
3. encrypt the enviroment variables and pass it as a secret

### Step-by-Step Demo

#### 1. **Create a Kubernetes Deployment with Environment Variables**

In this example, we'll create a Kubernetes Deployment for a simple Nginx web server and pass environment variables to the container, then create a NodePort to Access the Nginx Web Server from the browser.

##### **Deployment YAML File**

```yaml
# deployment.kub.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-dep
  labels:
    app: nginx
spec:
  replicas: 1
  selector: 
    matchLabels:
      app: nginx
  template:
    metadata: 
      name: nginx-pod
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx-pod
          image: nginx
          ports:
          - containerPort: 80
            hostPort: 8080
          env:
            - name: ENVIROMENT
              value: "development"
            - name: SERVER_NAME
              value: "nginx-server"
```
### Explanation:
- **Deployment**: The file defines a deployment named `nginx-dep`.
- **Environment Variables**:
  - `ENVIRONMENT` is set to `"production"`.
  - `SERVER_NAME` is set to `"nginx-server"`.

##### **NodePort YAML File**

```yaml 
# NodePort.kub.yaml
apiVersion: v1
kind: Service
metadata: 
  name: nginx
  labels:
    name: nginx-nodePort
spec:
  type: NodePort
  selector: 
    app: nginx
  ports:
    - targetPort: 80
      port: 80
      nodePort: 30001 
```
### Explanation:
- The file defines a NodePort Service named `nginx` that can be accessed through port 30001.

##### **Commands to run the example **:

```bash
# Apply the deployment
kubectl apply -f deployment.kub.yaml

# Apply the service NodePort to access the nginx server from the browser
kubectl apply -f NodePort.kub.yaml

# Check the deployment
kubectl get deployments

# Get the NodePort URL by running the following command, and paste it into 
# your browser to check that the nginx is running correctly
minikube service --all
```

##### You will find the URL like that 

😿  service default/kubernetes has no node port

 | NAMESPACE | NAME  | TARGET PORT |            URL            |
 |-----------|-------|-------------|---------------------------|
 | default   | nginx |          80 | http://192.168.49.2:30001 |
 |-----------|-------|-------------|---------------------------|

##### **Commands to view the enviroment variabels inside the POD **:

```bash
# get inside the minikube running container
docker container exec -it minikube /bin/bash

# get the nginx running container ID
docker container ls

# show the enviroment variables (in my case the nginx container ID is cc4bd875f419)
docker container exec cc4bd875f419 env
```
##### the output should be as the following : -

```yaml
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
HOSTNAME=nginx-dep-86cc764b8d-t2mg9
ENVIROMENT=development
SERVER_NAME=nginx-server
NGINX_SERVICE_HOST=10.106.34.5
NGINX_PORT_80_TCP_ADDR=10.106.34.5
NGINX_SERVICE_PORT=80
NGINX_PORT_80_TCP_PROTO=tcp
KUBERNETES_SERVICE_PORT_HTTPS=443
KUBERNETES_PORT=tcp://10.96.0.1:443
KUBERNETES_PORT_443_TCP_ADDR=10.96.0.1
NGINX_PORT=tcp://10.106.34.5:80
NGINX_PORT_80_TCP=tcp://10.106.34.5:80
KUBERNETES_SERVICE_HOST=10.96.0.1
KUBERNETES_PORT_443_TCP=tcp://10.96.0.1:443
KUBERNETES_PORT_443_TCP_PROTO=tcp
NGINX_PORT_80_TCP_PORT=80
KUBERNETES_SERVICE_PORT=443
KUBERNETES_PORT_443_TCP_PORT=443
NGINX_VERSION=1.27.2
NJS_VERSION=0.8.6
NJS_RELEASE=1~bookworm
PKG_RELEASE=1~bookworm
DYNPKG_RELEASE=1~bookworm
HOME=/root
```

#### 2. **Using `ConfigMap` to Manage Environment Variables**

Using a `ConfigMap` to store non-sensitive environment variables makes your deployment more flexible.
We can include all the enviroment variables from the ConfigMap or some of it
##### **ConfigMap YAML File**

```yaml
# configMap.kub.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-configfile
data:
  ENVIROMENT: "development"
  SERVER_NAME: "nginx"
```

##### **Deployment YAML File Using `ConfigMap`**

###### Example to include some of the env

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-dep
  labels:
    app: nginx
spec:
  replicas: 1
  selector: 
    matchLabels:
      app: nginx
  template:
    metadata: 
      name: nginx-pod
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx-pod
          image: nginx
          ports:
          - containerPort: 80
            hostPort: 8080
          env:
            - name: ENVIROMENT # env name in the pod
              valueFrom:
                configMapKeyRef:
                  name: nginx-configfile # ConfigFile name
                  key: ENVIROMENT        # env name in the ConfigFile
```

###### Example to include all of the env

```yaml
# deployment_2.kub.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-dep
  labels:
    app: nginx
spec:
  replicas: 1
  selector: 
    matchLabels:
      app: nginx
  template:nginx-configfile
    metadata: 
      name: nginx-pod
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx-pod
          image: nginx
          ports:
          - containerPort: 80
            hostPort: 8080
          envFrom:
            - configMapRef:
                name: nginx-configfile # Name of the config file
 
```

### Explanation:
- **ConfigMap**: A `ConfigMap` named `nginx-configfile` stores key-value pairs for environment variables (`ENVIRONMENT` and `SERVER_NAME`).
- **Using `ConfigMap` in Deployment**:
  - `valueFrom` is used to reference the `ConfigMap`.
  - The key-value pairs are passed as environment variables to the container.

##### **Commands to run the example  **:

```bash
# Apply the ConfigMap
kubectl apply -f configMap.kub.yaml

# Apply the deployment (choose between 1 or 2)
kubectl apply -f deployment_1.kub.yaml

# Apply the service NodePort to access the nginx server from the browser
kubectl apply -f NodePort.kub.yaml

# Check the deployment
kubectl get deployments

# Get the NodePort URL by running the following command, and paste it into 
# your browser to check that the nginx is running correctly
minikube service --all
```

##### You will find the URL like that 

😿  service default/kubernetes has no node port

 | NAMESPACE | NAME  | TARGET PORT |            URL            |
 |-----------|-------|-------------|---------------------------|
 | default   | nginx |          80 | http://192.168.49.2:30001 |
 |-----------|-------|-------------|---------------------------|

##### **Commands to view the enviroment variabels inside the POD **:

```bash
# get inside the minikube running container
docker container exec -it minikube /bin/bash

# get the nginx running container ID
docker container ls

# show the enviroment variables (in my case the nginx container ID is cc4bd875f419)
docker container exec cc4bd875f419 env
```
##### the output should be as the following in case of deplotment_1 is used 
```yaml
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
HOSTNAME=nginx-dep-86cc764b8d-t2mg9
ENVIROMENT=development
SERVER_NAME=nginx-server
NGINX_SERVICE_HOST=10.106.34.5
NGINX_PORT_80_TCP_ADDR=10.106.34.5
NGINX_SERVICE_PORT=80
NGINX_PORT_80_TCP_PROTO=tcp
KUBERNETES_SERVICE_PORT_HTTPS=443
KUBERNETES_PORT=tcp://10.96.0.1:443
KUBERNETES_PORT_443_TCP_ADDR=10.96.0.1
NGINX_PORT=tcp://10.106.34.5:80
NGINX_PORT_80_TCP=tcp://10.106.34.5:80
KUBERNETES_SERVICE_HOST=10.96.0.1
KUBERNETES_PORT_443_TCP=tcp://10.96.0.1:443
KUBERNETES_PORT_443_TCP_PROTO=tcp
NGINX_PORT_80_TCP_PORT=80
KUBERNETES_SERVICE_PORT=443
KUBERNETES_PORT_443_TCP_PORT=443
NGINX_VERSION=1.27.2
NJS_VERSION=0.8.6
NJS_RELEASE=1~bookworm
PKG_RELEASE=1~bookworm
DYNPKG_RELEASE=1~bookworm
HOME=/root
```
##### the output should be as the following in case of deplotment_2 is used 
```yaml
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
HOSTNAME=nginx-dep-86cc764b8d-t2mg9
ENVIROMENT=development
NGINX_SERVICE_HOST=10.106.34.5
NGINX_PORT_80_TCP_ADDR=10.106.34.5
NGINX_SERVICE_PORT=80
NGINX_PORT_80_TCP_PROTO=tcp
KUBERNETES_SERVICE_PORT_HTTPS=443
KUBERNETES_PORT=tcp://10.96.0.1:443
KUBERNETES_PORT_443_TCP_ADDR=10.96.0.1
NGINX_PORT=tcp://10.106.34.5:80
NGINX_PORT_80_TCP=tcp://10.106.34.5:80
KUBERNETES_SERVICE_HOST=10.96.0.1
KUBERNETES_PORT_443_TCP=tcp://10.96.0.1:443
KUBERNETES_PORT_443_TCP_PROTO=tcp
NGINX_PORT_80_TCP_PORT=80
KUBERNETES_SERVICE_PORT=443
KUBERNETES_PORT_443_TCP_PORT=443
NGINX_VERSION=1.27.2
NJS_VERSION=0.8.6
NJS_RELEASE=1~bookworm
PKG_RELEASE=1~bookworm
DYNPKG_RELEASE=1~bookworm
HOME=/root
```

#### 3. **Using `Secrets` for Sensitive Environment Variables**

Sensitive data such as passwords or API tokens should be stored in Kubernetes Secrets.
The main difference between Secret and ConfigMap that in Secret we pass the value of the enviroment variable encrypted to increase the security.

##### **Secret YAML File**

```yaml
# secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: nginx-secret
type: Opaque
data:
  SECRET_API_KEY: c2VjcmV0X2tleV92YWx1ZQ==
```

> Note: The `SECRET_API_KEY` is base64-encoded (`echo -n "secret_key_value" | base64`) and to decode the secret use (`echo -n "secret_key_value_encrypted" | base64 --decode`).

##### **Deployment YAML File Using `Secret`**

```yaml
# nginx-deployment-secret.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:latest
          ports:
            - containerPort: 80
          env:
            - name: SECRET_API_KEY
              valueFrom:
                secretKeyRef:
                  name: nginx-secret
                  key: SECRET_API_KEY
```

### Explanation:
- **Secret**: A `Secret` named `nginx-secret` stores the base64-encoded API key.
- **Using `Secret` in Deployment**:
  - `valueFrom.secretKeyRef` is used to reference the secret, passing it as an environment variable to the container.

##### **Commands to Create and Apply the Secret and Deployment**:
```bash
# Apply the ConfigMap
kubectl apply -f secret.yaml

# Apply the deployment 
kubectl apply -f nginx-deployment-secret.yaml

# Apply the service NodePort to access the nginx server from the browser
kubectl apply -f NodePort.kub.yaml

# Check the deployment
kubectl get deployments

# Get the NodePort URL by running the following command, and paste it into 
# your browser to check that the nginx is running correctly
minikube service --all
```

##### You will find the URL like that 

😿  service default/kubernetes has no node port

 | NAMESPACE | NAME  | TARGET PORT |            URL            |
 |-----------|-------|-------------|---------------------------|
 | default   | nginx |          80 | http://192.168.49.2:30001 |
 |-----------|-------|-------------|---------------------------|

##### **Commands to view the enviroment variabels inside the POD **:

```bash
# get inside the minikube running container
docker container exec -it minikube /bin/bash

# get the nginx running container ID
docker container ls

# show the enviroment variables (in my case the nginx container ID is cc4bd875f419)
docker container exec cc4bd875f419 env
```
##### the output should be as the following 
```yaml
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
HOSTNAME=nginx-dep-86cc764b8d-t2mg9
SECRET_API_KEY=secret_key_value
NGINX_SERVICE_HOST=10.106.34.5
NGINX_PORT_80_TCP_ADDR=10.106.34.5
NGINX_SERVICE_PORT=80
NGINX_PORT_80_TCP_PROTO=tcp
KUBERNETES_SERVICE_PORT_HTTPS=443
KUBERNETES_PORT=tcp://10.96.0.1:443
KUBERNETES_PORT_443_TCP_ADDR=10.96.0.1
NGINX_PORT=tcp://10.106.34.5:80
NGINX_PORT_80_TCP=tcp://10.106.34.5:80
KUBERNETES_SERVICE_HOST=10.96.0.1
KUBERNETES_PORT_443_TCP=tcp://10.96.0.1:443
KUBERNETES_PORT_443_TCP_PROTO=tcp
NGINX_PORT_80_TCP_PORT=80
KUBERNETES_SERVICE_PORT=443
KUBERNETES_PORT_443_TCP_PORT=443
NGINX_VERSION=1.27.2
NJS_VERSION=0.8.6
NJS_RELEASE=1~bookworm
PKG_RELEASE=1~bookworm
DYNPKG_RELEASE=1~bookworm
HOME=/root
```

### Conclusion:
1. **Direct Variables**: The first example shows how to pass environment variables directly in the YAML file.
2. **ConfigMap**: The second example uses `ConfigMap` for non-sensitive environment data.
3. **Secret**: The third example uses `Secret` for sensitive data like passwords or API keys.

By using `ConfigMaps` and `Secrets`, you can manage configuration data and sensitive information more securely and flexibly across environments.
