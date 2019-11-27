
# a sample ASP.NET Core webapi

## pre-requisite

1. ensure you have dotnet core installed
2. docker engine installed

The following was tested based on the following on MacOS Mojave 10.14.6 (18G103)

```
$ dotnet --version
3.0.100
```

```
$ docker -v
Docker version 19.03.4, build 9013bf5
```

## how to create this project

```
$ dotnet new webapp -o SampleWebApi
```

the command above generated codes with https enabled, for codes without https, use the following

```
$ dotnet new webapp -o SampleWebApi --no-https
```

## build the docker image

```
$ docker build -t sample-webapi:0.1.0 .
```


## to test run image

the following command will run the application in docker with ssl cert created above and mounted to the machine folder ```${HOME}/.aspnet/https```

### without https

if you created the app the flag ```--no-https``` then you can run the container with below 

```
docker run --rm -it -p 8000:80 sample-webapi:0.1.0
```

test it with curl or browser
```
curl http://localhost:8000/weatherforecast
```

### with https, run the following command

if you created the app with https then you will need a certificate to run and test your application.


**Generate certificate and configure local machine**

This will generate a certificate and you need to specify a password for the certificate.

```
dotnet dev-certs https -ep ${HOME}/.aspnet/https/aspnetapp.pfx -p password
dotnet dev-certs https --trust
```

**run the container**

-v : mount the certificate in your local machine to the container
-p : forward a local machine port to container port e.g. forward local port 8000 (http) to container port 80
-e : environment variables for donet container to be overriden.

```
docker run --rm -it -p 8000:80 -p 8001:443 -e ASPNETCORE_URLS="https://+;http://+" -e ASPNETCORE_HTTPS_PORT=8001 -e ASPNETCORE_Kestrel__Certificates__Default__Password="password" -e ASPNETCORE_Kestrel__Certificates__Default__Path=/https/aspnetapp.pfx -v ${HOME}/.aspnet/https:/https/ sample-webapi:0.1.0
```

**test it**
```
curl https://localhost:8001/weatherforecast
```


## push to dockerhub

### tag it 

you will need to replace <dockerhub-user-id> with your dockerhub id, if you don't have one, sign up [here](https://hub.docker.com/signup).

```
docker tag sample-webapi:0.1.0 <dockerhub-user-id>/sample-webapi:0.1.0
```

### push to dockerhub


```
docker push <dockerhub-user-id>/sample-webapi:0.1.0
```

# Kubernetes deployment

if you have a kubernetes cluster you can test out the following.

for local testing you can used Docker Decktop Kubernetes, a single node local kubernetes cluster.

## testing the cluster

```
kubectl cluster-info
Kubernetes master is running at https://kubernetes.docker.internal:6443
KubeDNS is running at https://kubernetes.docker.internal:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

## check current kube context

```
kubectl config get-context
```

## set kube context

If you have docker kubernetes running, set to docker-desktop context with the following command

```
kubectl config use-context docker-desktop
Switched to context "docker-desktop".
```

## to deploy 

change folder to ./SampleWebApi and run the following commands, it will deploy the application to docker-desktop kubernetes.

ensure the image used in the deployment is the tag you create with docker build and tag.

```
kubectl apply -f k8s/sample-webapi.yaml
kubectl get deployment
kubectl get services
```

## check deployment

the deployment will deploy 2 replica

```
kubectl get pods
NAME                                    READY   STATUS    RESTARTS   AGE
js-webapi-deployment-54df447bd8-rn7xh   1/1     Running   0          2m54s
js-webapi-deployment-54df447bd8-vbnzs   1/1     Running   0          2m54s
```

to access the application in browser, you need to get the NodePort of the service.

```
kubectl get services
NAME                TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE
js-webapi-service   NodePort    10.101.26.17   <none>        8080:31740/TCP   8m15s
kubernetes          ClusterIP   10.96.0.1      <none>        443/TCP          10m
```

access the app: http://localhost:31740/weatherforecast

## scale the deployment

to scale from 2 to 3 replica, run the following command

```
kubectl scale --replicas=3 deployment js-webapi-deployment
deployment.extensions/js-webapi-deployment scaled

kubectl get pods
NAME                                    READY   STATUS    RESTARTS   AGE
js-webapi-deployment-54df447bd8-gscck   1/1     Running   0          14s
js-webapi-deployment-54df447bd8-rn7xh   1/1     Running   0          6m47s
js-webapi-deployment-54df447bd8-vbnzs   1/1     Running   0          6m47s
```

# service with LoadBalancer

edit the file k8s/sample-webapi.yaml

change service ```type: NodePort``` to ```type: LoadBalancer``` and run the following command 

```
kubectl apply -f k8s/sample-webapi.yaml
service/js-webapi-service configured
deployment.apps/js-webapi-deployment unchanged
```

you can access the application at : http://localhost:8080/weatherforecast

## to remove deployment

```
kubectl delete -f k8s/sample-webapi.yaml
service "js-webapi-service" deleted
deployment.apps "js-webapi-deployment" deleted
```

## deploy Kubernetes Dashboard

```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v1.10.1/src/deploy/recommended/kubernetes-dashboard.yaml
```

To access Dashboard from your local workstation you must create a secure channel to your Kubernetes cluster. Run the following command:

```
kubectl proxy
```

Now access Dashboard at:

http://localhost:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/

you get token to login, copy the token and provide it for login with TOKEN.

```
kubectl -n kube-system describe secret default
```

or 

```
TOKEN=`kubectl -n kube-system describe secret default | grep 'token:' | awk '{print $2}'`
kubectl config set-credentials docker-desktop --token="${TOKEN}"
```

# Using Helm

to setup helm with docker-decktop kubenetes

## Requirement
- download helm 


## install tiller 

ensure you are using docker-desktop kubernetes

```
kubectl config current-context
docker-desktop
```

then install tiller 

```
helm init
```

check tiller is up and running

```
kubectl get pods -n kube-system
NAME                                     READY   STATUS    RESTARTS   AGE
coredns-6dcc67dcbc-2sj96                 1/1     Running   0          162m
coredns-6dcc67dcbc-rts84                 1/1     Running   0          162m
etcd-docker-desktop                      1/1     Running   0          161m
kube-apiserver-docker-desktop            1/1     Running   0          161m
kube-controller-manager-docker-desktop   1/1     Running   0          161m
kube-proxy-n268j                         1/1     Running   0          162m
kube-scheduler-docker-desktop            1/1     Running   0          161m
kubernetes-dashboard-5f7b999d65-k4jbs    1/1     Running   0          104m
tiller-deploy-fc56b78dd-cdhn9            1/1     Running   0          4m32s
```

## package the app in helm


create the helm chart

```
helm create 
```

install chart

```
helm install --name my-release stable/redis
```

delete chart

```
helm delete my-release
```





# helm chart

the following commands required ```--tls``` with IBM Cloud Private 3.2.1 on Red Hat OKD 3.11 

## deploy dry-run

```
cloudctl login -a https://<Cluster Master Host>:<Cluster Master API Port> --skip-ssl-validation
```

with ICP 3.2.1 you should see the following when you run helm version command

```
helm version --tls
Client: &version.Version{SemVer:"v2.12.3", GitCommit:"eecf22f77df5f65c823aacd2dbd30ae6c65f186e", GitTreeState:"clean"}
Server: &version.Version{SemVer:"v2.12.3+icp", GitCommit:"", GitTreeState:""}
```

## create the chart

```
helm create dotnet3-webapi-chart
```

the above command will create a folder with the following structure

```
.
├── Chart.yaml
├── README.md
├── charts
├── templates
│   ├── NOTES.txt
│   ├── _helpers.tpl
│   ├── deployment.yaml
│   ├── ingress.yaml
│   ├── service.yaml
│   └── tests
│       └── test-connection.yaml
└── values.yaml
```

where I have added README.md and modified values.yaml, deployment.yaml, service.yaml 

## test run your helm chart

you can test run your helm chart to check the output of the deployment as follows.

```
helm install --dry-run --debug ./dotnet3-webapi-chart --tls
```

## to deploy it

```
helm install --name js-dotnet3-webapi ./dotnet3-webapi-chart --tls
```

## listing of helm deployment

```
helm list
```

## to access the application

the application is deployed using NodePort to get the port value, run the following command

```
$ kubectl get services
NAME                                     TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE
js-dotnet3-webapi-dotnet3-webapi-chart   NodePort    10.96.127.31   <none>        5000:31855/TCP   41s
kubernetes                               ClusterIP   10.96.0.1      <none>        443/TCP          5h32m
```

access the application API at http://localhost:31855/weatherforecast

## remove deployment

```
helm delete js-dotnet3-webapi --purge --tls
```

## ensure best practices in chart

```
helm lint ./dotnet3-webapi-chart
==> Linting ./dotnet3-webapi-chart
[INFO] Chart.yaml: icon is recommended

1 chart(s) linted, no failures
```

## to share it 

in order to share this chart, you need to package it as archive.

```
helm package ./dotnet3-webapi-chart
``` 

deploying with archive

```
helm install --name js-dotnet3-webapi dotnet3-webapi-chart-0.1.0.tgz --tls
```


# Deploying to OpenShift 4

## Requirements
- [install](https://developers.redhat.com/products/codeready-containers) codeready container which is a single node cluster on your machine.

## setup oc CLI

```
crc oc-env
```

and run the printed command shown above.

```
eval $(crc oc-env)
```

to get credential to crc console, run the following command

```
crc console --credentials
```

# Resources

- [Containerizing an Asp.Net Core 3.0 Web API](https://www.youtube.com/watch?v=Po9jQS7WBDQ)
- [Deploying an Asp.Net Core 3.0 Web API on Kubernetes](https://www.youtube.com/watch?v=ZOROT9yMp44)
- [Hosting ASP.NET Core images with Docker over HTTPS](https://docs.microsoft.com/en-us/aspnet/core/security/docker-https?view=aspnetcore-3.0)
- [.NET security tips](https://cheatsheetseries.owasp.org/cheatsheets/DotNet_Security_Cheat_Sheet.html) for developers
- [Let’s Encrypt](https://letsencrypt.org/) is a free, automated, and open Certificate Authority