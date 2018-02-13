# DevOps Challenge

Dockerize and Deploy a sample ruby on rails application With Redis, Postgress, Unicorn and Sidekiq using Kubernates

## Prerequisites

Follow The Blog in https://semaphoreci.com/community/tutorials/dockerizing-a-ruby-on-rails-application to dockerize the application using a docker-compose file

Install kubectl, minikube and kompose to run and deploy the application in Kubernates as below

```
$curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl && chmod +x ./kubectl && sudo mv ./kubectl /usr/local/bin/kubectl
```

```
$curl -Lo minikube https://storage.googleapis.com/minikube/releases/v0.25.0/minikube-linux-amd64 && chmod +x minikube && sudo mv minikube /usr/local/bin/
```

```
$curl -L https://github.com/kubernetes/kompose/releases/download/v1.1.0/kompose-linux-amd64 -o kompose && chmod +x ./kubectl && sudo mv ./kubectl /usr/local/bin/
```

## Getting Started

### Build and Push Docker Image

Regarding The above blog after creating Dockerfile the docker image will be created and pushed to dockerhub to be used by kubernates

```
$docker build -t  mennatallahshaaban/drkiq:1.3 .
```

```
$docker push  mennatallahshaaban/drkiq:1.3
```

#### Note

Please note that the tag of the image must be created with the username which is used to login in dockerhub

### Modify docker-compose.yml to be able to run it with no need to create volume before or intialize the database with "docker-compose run <command>" as attached in the repo and test it with "$docker-compose up" everything should be working in the application then run the below command to convert it to kubernates yml files

```
$kompose convert
```
That will create yml files of the services, deployments and PVCs of kubernates and to create and start them in Kubernates run:

```
$kubectl create -f drkiq-deployment.yaml,drkiq-service.yaml,drkiq-postgres-persistentvolumeclaim.yaml,drkiq-redis-persistentvolumeclaim.yaml,postgres-deployment.yaml,postgres-service.yaml,redis-deployment.yaml,redis-service.yaml,sidekiq-deployment.yaml,sidekiq-service.yaml
```
Another way to do that is to only run:

```
$kompose up
```

To validate that the application is running, the port 8000 needs to be forwarded to localhost to test it by running:

```
$kubectl port-forward <pod name> 8000:8000
```

Then Open the browser in http://localhost:8000 or run:

``` 
$curl -v http://localhost:8000
```

#### Note
The application may take few minutes to be up

## Use another way to pass environment variables to the application containers in kubernates
There is different ways to pass variables to the containers as below:

### Use env parameter in the yaml file of the deployment
To test that let's remove the env_file from docker-compose.yml file for both drkik and sidekiq containers to be as below:

```
#    env_file:
#      - ./drkiq/.drkiq.env
```

Now we need to remove the created yaml files to be recreated again but without including the variables using "$kompose convert" and to add the same variables in the .env file modify the deployment files of drkiq and sidekiq and add env parameter to be as below under spec:

```
        env:
        - name: CACHE_URL
          value: redis://redis:6379/0
        - name: DATABASE_URL
          value: postgresql://drkiq:yourpassword@postgres:5432/drkiq?encoding=utf8&pool=5&timeout=5000
        - name: JOB_WORKER_URL
          value: redis://redis:6379/0
        - name: LISTEN_ON
          value: 0.0.0.0:8000
        - name: SECRET_TOKEN
          value: asecuretokenwouldnormallygohere
        - name: WORKER_PROCESSES
          value: "1"
```

To Validate that it works let's run the containers with "$kubectl create -f <yml files>", forward the port 8000 and test it from the browser as we did before.

### Use ConfigMaps and Secrets to make the variables isolated from the applications so no need to rebuild the containers when variables need to be changed and also not to duplicate variables if they are needed for multiple containers.

The difference between Secretes and ConfigMaps is that Secrets is Used for confidential data by using base64 encoding and ConfigMap is used for normal data.

Here we will use ConfigMap by applying the below:

#### Create configMap and pass a variable to it by running:

```
$kubectl create configmap app-config --from-literal=SECRET_TOKEN=asecuretokenwouldnormallygohere
```

#### Edit The config map to add the rest of the variable by running 

```
$kubectl edit configmap app-config
```

To look like this:

```
apiVersion: v1
data:
  CACHE_URL: redis://redis:6379/0
  DATABASE_URL: postgresql://drkiq:yourpassword@postgres:5432/drkiq?encoding=utf8&pool=5&timeout=5000
  JOB_WORKER_URL: redis://redis:6379/0
  LISTEN_ON: 0.0.0.0:8000
  SECRET_TOKEN: asecuretokenwouldnormallygohere
  WORKER_PROCESSES: "1"
kind: ConfigMap
metadata:
  creationTimestamp: 2018-02-13T04:54:29Z
  name: app-config
  namespace: default
  resourceVersion: "28784"
  selfLink: /api/v1/namespaces/default/configmaps/app-config
  uid: f92b4a6c-1079-11e8-b597-080027ebf50c
```

OR by using the Minikube dashboard and using UI to look like this:

```
{
  "kind": "ConfigMap",
  "apiVersion": "v1",
  "metadata": {
    "name": "app-config",
    "namespace": "default",
    "selfLink": "/api/v1/namespaces/default/configmaps/app-config",
    "uid": "f92b4a6c-1079-11e8-b597-080027ebf50c",
    "resourceVersion": "28784",
    "creationTimestamp": "2018-02-13T04:54:29Z"
  },
  "data": {
    "CACHE_URL": "redis://redis:6379/0",
    "DATABASE_URL": "postgresql://drkiq:yourpassword@postgres:5432/drkiq?encoding=utf8&pool=5&timeout=5000",
    "JOB_WORKER_URL": "redis://redis:6379/0",
    "LISTEN_ON": "0.0.0.0:8000",
    "SECRET_TOKEN": "asecuretokenwouldnormallygohere",
    "WORKER_PROCESSES": "1"
  }
}
```

#### To make the containers use the configMaps modify the env portion of the deployment.yaml files for drkiq and sidekiq to look like this:
 
```
        env:
        - name: CACHE_URL
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: CACHE_URL
        - name: DATABASE_URL
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: DATABASE_URL
        - name: JOB_WORKER_URL
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: JOB_WORKER_URL
        - name: LISTEN_ON
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: LISTEN_ON
        - name: SECRET_TOKEN
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: SECRET_TOKEN
        - name: WORKER_PROCESSES
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: WORKER_PROCESSES
```

#### Apply the deployment changes by running:

```
$kubectl replace -f drkiq-deployment.yaml
$kubectl replace -f sidekiq-deployment.yaml
```

#### Note
We can always validate the work by testing the application from the browser and to get the info about resources run

```
kubectl get  deployment,svc,pods,pvc
```

## Author

* **Mennatallah Shaaban** 
