# Deploying to Knative from a Private GitHub Repo

This sample demonstrates:
* Pulling source code from a private Github repository using a deploy-key
* Pushing a Docker container to a private DockerHub repository using a username / password
* Deploying to Knative Serving using image pull secrets

## Before you begin

* [Install Knative Serving](../../../install/README.md)
* Create a local folder for this sample and download the files in this directory into it.

## Setup

### 1. Setting up the default service account

Knative Serving will run pods as the default service account in the namespace where
you created your resources.  You can see its body by entering the following command:

```shell
$ kubectl get serviceaccount default --output yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: default
  namespace: default
  ...
secrets:
- name: default-token-zd84v
```

We are going to add to this an image pull Secret.

1. Create your image pull Secret with the following command, replacing values as neccesary:
   ```shell
   kubectl create secret docker-registry dockerhub-pull-secret \
   --docker-server=https://index.docker.io/v1/ --docker-email=not@val.id \
   --docker-username=<your-name> --docker-password=<your-pword>
   ```

   To learn more about Kubernetes pull Secrets, see
   [Creating a Secret in the cluster that holds your authorization token](https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/#create-a-secret-in-the-cluster-that-holds-your-authorization-token).
   
2. Add the newly created `imagePullSecret` to your default service account by entering:
   ```shell
   kubectl edit serviceaccount default
   ```

   This will open the resource in your default text editor. Under `secrets:`, add:

   ```yaml
   secrets:
   - name: default-token-zd84v
   # This is the secret we just created:
   imagePullSecrets:
   - name: dockerhub-pull-secret
   ```


### 2. Configuring the build

The objects in this section are all defined in `build-bot.yaml`, and the fields that
need to be changed say `REPLACE_ME`. Open the `build-bot.yaml` file and make the
necessary replacements.

The following sections explain the different configurations in the `build-bot.yaml` file,
as well as the necessary changes for each section.

#### Setting up our Build service account
To separate our Build's credentials from our applications credentials, the
Build runs as its own service account:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: build-bot
secrets:
- name: deploy-key
- name: dockerhub-push-secrets
```

#### Creating a deploy key

You can set up a deploy key for a private Github repository following
[these](https://developer.github.com/v3/guides/managing-deploy-keys/)
instructions. The deploy key in the `build-bot.yaml` file in this folder is *real*;
you do not need to change it for the sample to work.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: deploy-key
  annotations:
    # This tells us that this credential is for use with
    # github.com repositories.
    build.knative.dev/git-0: github.com
type: kubernetes.io/ssh-auth
data:
  # Generated by:
  # cat id_rsa | base64 -w 1000000
  ssh-privatekey: <long string>

  # Generated by:
  # ssh-keyscan github.com | base64 -w 100000
  known_hosts: <long string>
```

#### Creating a DockerHub push credential

Create a new Secret for your DockerHub credentials. Replace the necessary values:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: dockerhub-push-secrets
  annotations:
    build.knative.dev/docker-0: https://index.docker.io/v1/
type: kubernetes.io/basic-auth
stringData:
  username: <dockerhub-user>
  password: <dockerhub-password>
```

#### Creating the build bot

When finished with the replacements, create the build bot by entering the following command:

```shell
kubectl create --filename build-bot.yaml
```

### 3. Installing a Build template and updating `manifest.yaml`
1. Install the
   [Kaniko build template](https://github.com/knative/build-templates/blob/master/kaniko/kaniko.yaml)
   by entering the following command:

   ```shell
   kubectl apply --filename https://raw.githubusercontent.com/knative/build-templates/master/kaniko/kaniko.yaml
   ```
   
1. Open `manifest.yaml` and substitute your private DockerHub repository name for
   `REPLACE_ME`.

## Deploying your application

At this point, you're ready to deploy your application:

```shell
kubectl create --filename manifest.yaml
```

To make sure everything works, capture the host URL and the IP of the ingress endpoint
in environment variables:

```
# Put the Host URL into an environment variable.
export SERVICE_HOST=`kubectl get route private-repos \
  --output jsonpath="{.status.domain}"`
```

```
# Put the IP address into an environment variable
export SERVICE_IP=`kubectl get svc knative-ingressgateway --namespace istio-system --output jsonpath="{.status.loadBalancer.ingress[*].ip}"`
```

> Note: If your cluster is running outside a cloud provider (for example, on Minikube),
  your services will never get an external IP address. In that case, use the Istio
  `hostIP` and `nodePort` as the service IP:

   ```shell
   export SERVICE_IP=$(kubectl get po --selector knative=ingressgateway --namespace istio-system --output 'jsonpath= .  {.items[0].status.hostIP}'):$(kubectl get svc knative-ingressgateway --namespace istio-system --output 'jsonpath={.spec.ports[? (@.port==80)].nodePort}')
   ```

Now curl the service IP to make sure the deployment succeeded:

```
curl -H "Host: $SERVICE_HOST" http://$SERVICE_IP
```


## Appendix: Sample Code

The sample code is in a private Github repository consisting of two files.

1. `Dockerfile`
   ```Dockerfile
   FROM golang

   ENV GOPATH /go

   ADD . /go/src/github.com/dewitt/knative-build

   RUN CGO_ENABLED=0 go build github.com/dewitt/knative-build

   ENTRYPOINT ["knative-build"]
   ```

1. `main.go`

   ```go
   package main

   import (
	   "fmt"
	   "net/http"
   )

   const (
   	   port = ":8080"
   )

   func helloWorld(w http.ResponseWriter, r *http.Request) {
	   fmt.Fprintf(w, "Hello World.")
   }

   func main() {
	   http.HandleFunc("/", helloWorld)
	   http.ListenAndServe(port, nil)
   }
   ```
