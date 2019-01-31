# Deploy to a Docker Container via IBM Cloud Kubernetes Service

Follow these instructions to deploy the sample application to a Kubernetes cluster and connect the application to a Cloudant database.


## Download

To download the Git repository of this sample application,

```bash
git clone https://github.com/IBM-Cloud/get-started-node
cd get-started-node
```


## Build Docker Image

To build and tag your Docker container image,

1. Start a terminal window.

2. Login to IBM Cloud.

```
ibmcloud login
```

3. Set org and space in IBM Cloud.
```
ibmcloud target --cf
```

4. Log in to the IBM Cloud Container Registry CLI. Note: Ensure that the [container-registry plug-in](https://console.bluemix.net/docs/services/Registry/index.html#registry_cli_install) is installed.

```
ibmcloud cr login
```

5. Find your container registry **namespace** by running the following command and write down the namespace that you want to use.
```
ibmcloud cr namespaces
```

6. If you don't have any, create a new one.
```
ibmcloud cr namespace-add <name>
```
For example, `ibmcloud cr namespace-add mynamespace`

7. Identify your **Container Registry** by running the following command and write down the `Container Registry` entry.
```
ibmcloud cr info
```

Example output,
```
Container Registry                registry.ng.bluemix.net
Container Registry API endpoint   https://registry.ng.bluemix.net/api
IBM Cloud API endpoint            https://api.ng.bluemix.net
......
```

8. Build and tag (`-t`) the docker image by running the command below. Replace REGISTRY and NAMESPACE with your appropriate values.

   ```
   docker build . -t <REGISTRY>/<NAMESPACE>/myapp:v1.0.0
   ```
   For example: `docker build . -t registry.ng.bluemix.net/mynamespace/myapp:v1.0.0

9. Push the docker image to your Container Registry on IBM Cloud

   ```
   docker push <REGISTRY>/<NAMESPACE>/myapp:v1.0.0
   ```
   For example: `docker push registry.ng.bluemix.net/mynamespace/myapp:v1.0.0


## Create a Kubernetes Cluster in IBM Cloud

You need the Kubernetes cluster to deploy your docker images. To create the cluster and configure your environment,

* Follow the instructions to [Create a Kubernetes Cluster](https://github.com/IBM/container-journey-template), if you don't have one. You can use a cluster that already exits.

* Export the Kubernetes cluster name to environment variable $CLUSTER_NAME:

```
export CLUSTER_NAME=<your_cluster_name>
```

* Before you continue to the next step, verify that the deployment of your worker node is complete.

```
ibmcloud ks workers <cluster_name_or_ID>
or
ibmcloud ks workers $CLUSTER_NAME
```

> When your worker node is finished provisioning, the status changes to Ready or Deployed, and you can start binding IBM Cloud services.

* Set the Kubernetes environment variable KUBECONFIG to work with your cluster:

```
ibmcloud cs cluster-config $CLUSTER_NAME
```

* The output of the above command will contain a KUBECONFIG environment variable that must be exported in order to set the context of your terminal window. `Copy and paste the output in the terminal window to set the KUBECONFIG environment variable`. An example is:

```
export KUBECONFIG=/home/rak/.bluemix/plugins/container-service/clusters/Kate/kube-config-prod-dal10-<cluster_name>.yml
```

* Verify that you can connect to your cluster by listing your worker nodes. 
```
kubectl get nodes
```


## Create a Cloudant Database 

To create a Cloudant database service in IBM Cloud,

1. Go to the [Catalog](https://console.bluemix.net/catalog/) and create a new [Cloudant](https://console.bluemix.net/catalog/services/cloudant) database instance.

2. During creating the Cloudant service, choose `Legacy and IAM` for **Authentication**

3. After creating the Cloudant service, Create new credentials in **Service Credentials** tab.

4. Extend `View credentials` and copy value of the **url** field.

4. Create a Kubernetes secret with your Cloudant credentials in your terminal window.

```bash
kubectl create secret generic cloudant --from-literal=url=<URL>
```
Example:
```bash
kubectl create secret generic cloudant --from-literal=url=https://e956f907-35d7-486a-bb59-c859f2577c4b-bluemix:1ed9793d24730fb7f9692d83c4f6132c7395d0c8e39ff34f52b953e1b76702e0@e956f907-35d7-486a-bb59-c859f2577c4b-bluemix.cloudantnosqldb.appdomain.cloud
```


## Deploy

This section discuss how to deploy the container to IBM Cloud via IBM Kubernetes service.

#### Create the deployment

To deploy the container to IBM Cloud,

1. Open `kubernetes/deployment.yaml` file in an editor.

2. Replace `<REGISTRY>` and `<NAMESPACE>` with the appropriate values that you wrote down earlier.

3. Save.

4. Create a deployment:
  ```shell
  kubectl create -f kubernetes/deployment.yaml
  ```

5. Expose te service

- **For Free Cluster**: Use the Worker IP and NodePort
  ```bash
  kubectl expose deployment get-started-node --type NodePort --port 8080 --target-port 8080
  ```

- **For Paid Cluster**: Expose the service using an External IP and Loadbalancer
  ```
  kubectl expose deployment get-started-node --type LoadBalancer --port 8080 --target-port 8080
  ```

6. Verify **STATUS** of pod is `RUNNING`

```shell
kubectl get pods -l app=get-started-node
```

#### Access the application

To verify the application,

- **Free Cluster:**

1. Identify your Worker **Public IP** using command `ibmcloud cs workers YOUR_CLUSTER_NAME`.

Example output,

```
ID                                                 Public IP         Private IP
    Machine Type   State    Status   Zone    Version
kube-hou02-pac0ee0db9bcee40d6b7ee3b0ccace10c2-w1   184.172.234.139   10.76.193.8
1   free           normal   Ready    hou02   1.10.12_1541
```

2. Identify the **NodePort** using command `kubectl describe service get-started-node`.

Example output,

```
Name:                     get-started-node
Namespace:                default
Labels:                   app=get-started-node
Annotations:              <none>
Selector:                 app=get-started-node
Type:                     NodePort
IP:                       172.21.83.47
Port:                     <unset>  8080/TCP
TargetPort:               8080/TCP
NodePort:                 <unset>  32066/TCP
Endpoints:                172.30.110.135:8080
Session Affinity:         None
External Traffic Policy:  Cluster
Events:                   <none>
```

3. Access your application at `http://<WORKER-PUBLIC-IP>:<NodePort>/`. For example,

```
http://184.172.234.139:32066/
```

- **Standard (Paid) Cluster:**

1. Identify your LoadBalancer Ingress IP using `kubectl get service get-started-node`
2. Access your application at t `http://<EXTERNAL-IP>:8080/`


## Clean Up

To cleanup,

```bash
kubectl delete deployment,service -l app=get-started-node
kubectl delete secret cloudant
```


## Troubleshooting

#### Installing Docker and Starting Docker Daemon

When executing command such as `docker build . -t registry.ng.bluemix.net/zhangllc/myapp:v1.0.0`, if you encountered error similar to the one below

```
ERRO[0000] failed to dial gRPC: cannot connect to the Docker daemon. Is 'docker daemon' running on this host?: dial unix /var/run/docker.sock: connect: no such file or directory 
```

The most common cause is that your docker deamon is not running. On macOS, the following procedure should resolve the issue.

1. Install Docker if it's installed, `brew cask install docker`.
1. Launch Docker Deamon. 
    * Press `Command` + `Space` keys together to bring up Spotlight Search
    * Enter `Docker` to launch Docker
    * Work with the `Docker Desktop` and confirm the privileged access, if prompmted.
    * A whale icon should appear in the top bar. You may have to wait for few seconds for it to start.
    * You should be able to run Docker commands. For example, `docker ps`.

On macOS, the docker binary is only a client and you cannot use it to run the docker daemon, because Docker daemon uses Linux-specific kernel features. Therefore you can’t run Docker natively in OS X.




