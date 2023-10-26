
# ELTA - Home Assignment


## Setup an EKS Cluster

First, we'll create an EKS cluster on AWS with eksctl:

```bash
eksctl create cluster -f cluster.yaml
```
to apply a cluster.yaml file:

```bash
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: basic-cluster
  region: eu-central-1

nodeGroups:
  - name: ng-1
    instanceType: t3.small
    desiredCapacity: 2
```
## Setup Jenkins on Kubernetes

For setting up a Jenkins Cluster on Kubernetes, we will do the following:
1. Create a Namespace

2. Create a service account with Kubernetes admin permissions.

3. Create local persistent volume for persistent Jenkins data on Pod restarts.

4. Create a deployment YAML and deploy it.

5. Create a service YAML and deploy it.

#### Step 1: Create a Namespace for Jenkins. It is good to categorize all the DevOps tools as a separate namespace from other applications.

```bash 
kubectl create namespace devops-tools
```
#### Step 2: Create a 'serviceAccount.yaml' file and copy the following admin service account manifest.
```bash 
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: jenkins-admin
rules:
  - apiGroups: [""]
    resources: ["*"]
    verbs: ["*"]
  - apiGroups: ["apps"]
    resources: ["*"]
    verbs: ["*"]
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: jenkins-admin
  namespace: devops-tools
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: jenkins-admin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: jenkins-admin
subjects:
- kind: ServiceAccount
  name: jenkins-admin
  namespace: devops-tools
```
The 'serviceAccount.yaml' creates a 'jenkins-admin' clusterRole, 'jenkins-admin' ServiceAccount and binds the 'clusterRole' to the service account.

The 'jenkins-admin' cluster role has all the permissions to manage the cluster components. You can also restrict access by specifying individual resource actions.

Now create the service account using kubectl.

```bash 
kubectl apply -f serviceAccount.yaml
```
#### Step 3: Create 'volume.yaml' and copy the following persistent volume manifest.
```bash 
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: local-storage
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: jenkins-pv-volume
  labels:
    type: local
spec:
  storageClassName: local-storage
  claimRef:
    name: jenkins-pv-claim
    namespace: devops-tools
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  local:
    path: /mnt
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - worker-node01
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: jenkins-pv-claim
  namespace: devops-tools
spec:
  storageClassName: local-storage
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 3Gi
```
Important Note: Replace 'worker-node01' with any one of your cluster worker nodes hostname.

You can get the worker node hostname using the kubectl.

kubectl get nodes
For volume, we are using the 'local' storage class for the purpose of demonstration. Meaning, it creates a 'PersistentVolume' volume in a specific node under the '/mnt' location.

As the 'local' storage class requires the node selector, you need to specify the worker node name correctly for the Jenkins pod to get scheduled in the specific node.

If the pod gets deleted or restarted, the data will get persisted in the node volume. However, if the node gets deleted, you will lose all the data.

Ideally, you should use a persistent volume using the available storage class with the cloud provider, or the one provided by the cluster administrator to persist data on node failures.

Let’s create the volume using kubectl

```bash 
kubectl create -f volume.yaml
```
#### Step 4: Create a Deployment file named 'deployment.yaml' and copy the following deployment manifest.
```bash 
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jenkins
  namespace: devops-tools
spec:
  replicas: 1
  selector:
    matchLabels:
      app: jenkins-server
  template:
    metadata:
      labels:
        app: jenkins-server
    spec:
      securityContext:
            fsGroup: 1000
            runAsUser: 1000
      serviceAccountName: jenkins-admin
      containers:
        - name: jenkins
          image: jenkins/jenkins:lts
          resources:
            limits:
              memory: "2Gi"
              cpu: "1000m"
            requests:
              memory: "500Mi"
              cpu: "500m"
          ports:
            - name: httpport
              containerPort: 8080
            - name: jnlpport
              containerPort: 50000
          livenessProbe:
            httpGet:
              path: "/login"
              port: 8080
            initialDelaySeconds: 90
            periodSeconds: 10
            timeoutSeconds: 5
            failureThreshold: 5
          readinessProbe:
            httpGet:
              path: "/login"
              port: 8080
            initialDelaySeconds: 60
            periodSeconds: 10
            timeoutSeconds: 5
            failureThreshold: 3
          volumeMounts:
            - name: jenkins-data
              mountPath: /var/jenkins_home
      volumes:
        - name: jenkins-data
          persistentVolumeClaim:
              claimName: jenkins-pv-claim
```
In this Jenkins Kubernetes deployment we have used the following:

'securityContext' for Jenkins pod to be able to write to the local persistent volume.

Liveness and readiness probe to monitor the health of the Jenkins pod.

Local persistent volume based on local storage class that holds the Jenkins data path '/var/jenkins_home'.

The deployment file uses local storage class persistent volume for Jenkins data. For production use cases, you should add a cloud-specific storage class persistent volume for your Jenkins data.
If you don’t want the local storage persistent volume, you can replace the volume definition in the deployment with the host directory as shown below.
```bash 
volumes:
- name: jenkins-data
emptyDir: \{}
```
Create the deployment using kubectl.
```bash 
kubectl apply -f deployment.yaml
```
Check the deployment status.
```bash 
kubectl get deployments -n devops-tools
```
Now, you can get the deployment details using the following command.
```bash 
kubectl describe deployments --namespace=devops-tools
```
#### Step5: Accessing Jenkins Using Kubernetes Service

We have now created a deployment. However, it is not accessible to the outside world. For accessing the Jenkins deployment from the outside world, we need to create a service and map it to the deployment.

Create 'service.yaml' and copy the following service manifest:
```bash 
apiVersion: v1
kind: Service
metadata:
  name: jenkins-service
  namespace: devops-tools
  annotations:
      prometheus.io/scrape: 'true'
      prometheus.io/path:   /
      prometheus.io/port:   '8080'
spec:
  selector:
    app: jenkins-server
  type: NodePort
  ports:
    - port: 8080
      targetPort: 8080
      nodePort: 32000
```
Here, we are using the type as 'NodePort' which will expose Jenkins on all kubernetes node IPs on port 32000. If you have an ingress setup, you can create an ingress rule to access Jenkins. Also, you can expose the Jenkins service as a Loadbalancer if you are running the cluster on AWS, Google, or Azure cloud.
Create the Jenkins service using kubectl.
```bash 
kubectl apply -f service.yaml
```
Now, when browsing to any one of the Node IPs on port 32000, you will be able to access the Jenkins dashboard.
```bash 
http://<node-ip>:32000
```
Jenkins will ask for the initial Admin password when you access the dashboard for the first time.

You can get that from the pod logs either from the Kubernetes dashboard or CLI. You can get the pod details using the following CLI command.
```bash 
kubectl get pods --namespace=devops-tools
```
With the pod name, you can get the logs as shown below. Replace the pod name with your pod name.
```bash 
kubectl logs jenkins-deployment-2539456353-j00w5 --namespace=devops-tools
```
The password can be found at the end of the log.

Alternatively, you can run the exec command to get the password directly from the location as shown below.
```bash 
kubectl exec -it jenkins-559d8cd85c-cfcgk cat /var/jenkins_home/secrets/initialAdminPassword -n devops-tools
```
## Jenkins integration with GitLab
To configure a Jenkins integration with GitLab:

- Grant Jenkins access to the GitLab project.

- Configure the Jenkins server.

- Configure the Jenkins project.

- Configure the GitLab project.

#### Grant Jenkins access to the GitLab project

1. Create a personal, project, or group access token.

2. Set the access token scope to API.

3. Copy the access token value to configure the Jenkins server.

#### Configure the Jenkins server

Install and configure the Jenkins plugin to authorize the connection to GitLab.

1. On the Jenkins server, select Manage Jenkins > Manage Plugins.
2. Select the Available tab. Search for gitlab-plugin and select it to install. See the Jenkins GitLab documentation for other ways to install the plugin.
3. Select Manage Jenkins > Configure System.
4. In the GitLab section, select Enable authentication for ‘/project’ end-point.
5. Select Add, then choose Jenkins Credential Provider.
6. Select GitLab API token as the token type.
7. In API Token, paste the access token value you copied from GitLab and select Add.
8. Enter the GitLab server’s URL in GitLab host URL.
9. To test the connection, select Test Connection.

#### Configure the Jenkins project

Set up the Jenkins project you intend to run your build on.

1. On your Jenkins instance, select New Item.
2. Enter the project’s name.
3. Select Freestyle or Pipeline and select OK. You should select a freestyle project, because the Jenkins plugin updates the build status on GitLab. In a pipeline project, you must configure a script to update the status on GitLab.
4. Choose your GitLab connection from the dropdown list.
5. Select Build when a change is pushed to GitLab.

#### Configure the GitLab project

1. On the left sidebar, select Search or go to and find your project.
2. Select Settings > Integrations.
3. Select Jenkins.
4. Select the Active checkbox.
5. Select the events you want GitLab to trigger a Jenkins build for
6. Enter the Jenkins server URL.
7. Optional. Clear the Enable SSL verification checkbox to disable SSL verification.
8. Enter the Project name. The project name should be URL-friendly, where spaces are replaced with underscores. To ensure the project name is valid, copy it from your browser’s address bar while viewing the Jenkins project.
9. If your Jenkins server requires authentication, enter the Username and Password.
10. Optional. Select Test settings.
11. Select Save changes.

## Jenkins Kubernetes Plugin Configuration

Jenkins Kubernetes plugin is required to set up Kubernetes-based build agents. Let’s configure the plugin.

#### Step 1: Install Jenkins Kubernetes Plugin
Go to Manage Jenkins –> Manage Plugins, search for Kubernetes Plugin in the available tab, and install it. The following Gif video shows the plugin installation process.

#### Step 2: Create a Kubernetes Cloud Configuration

Once installed, go to Manage Jenkins –> Manage Node & Clouds

Click Configure Clouds

“Add a new Cloud” select Kubernetes.

Select Kubernetes Cloud Details

#### Step 3: Configure Jenkins Kubernetes Cloud

Since we have Jenkins inside the Kubernetes cluster with a service account to deploy the agent pods, we don’t have to mention the Kubernetes URL or certificate key.

However, to validate the connection using the service account, use the Test connection button. It should show a connected message if the Jenkins pod can connect to the Kubernetes master API

#### Step 4: Configure the Jenkins URL Details

For Jenkins master running inside the cluster, you can use the Service endpoint of the Kubernetes cluster as the Jenkins URL because agents pods can connect to the cluster via internal service DNS.

The URL is derived using the following syntax.

```bash
http://<service-name>.<namespace>.svc.cluster.local:8080
```

In our case, the service DNS will be,
```bash
http://jenkins-service.devops-tools.svc.cluster.local:8080
```
Also, add the POD label that can be used for grouping the containers if required in terms of bulling or custom build dashboards.

#### Step 5: Create POD and Container Template

Next, you need to add the POD template with the details. The label kubeagent will be used in the job as an identifier to pick this pod as the build agent. Next, make sure you add 'jenkins-admin' in the service account, which will give the pod the permissions to deploy the app later on.

The next configuration is the container template. If you don’t add a container template, the Jenkins Kubernetes plugin will use the default JNLP image from the Docker hub to spin up the agents. ie, jenkins/inbound-agent

We can add multiple container templates to the POD template and use them in the pipeline. I've added 2:

#### 1. kaniko - a tool to build container images from a Dockerfile, inside a container or Kubernetes cluster.
kaniko doesn't depend on a Docker daemon and executes each command within a Dockerfile completely in userspace. This enables building container images in environments that can't easily or securely run a Docker daemon, such as a standard Kubernetes cluster.

I've configured it with:
```bash
name: kaniko

docker-image: gcr.io/kaniko-project/executor:debug
```
and also gave it a secret volume with the following settings:
```bash
Secret name: kaniko-secret

Mount path: /kaniko/.docker/
```
We need a volume because it will be our docker hub authentication. When we get built, we need to push the image to the docker hub. open a CLI and write the following command:
```bash
echo -n username:password | base64
```
This command will generate a base64 password for you. Now I want you to create a config.json file, the content should be as follows:
```bash
{
    "auths": {
        "https://index.docker.io/v1/": {
            "auth": "yourbase64password"
        }
    }
}
```
After creating our JSON file, you should come to the command line and run the following command, this command will create a secret on Kubernetes.
```bash
kubectl create secret generic kaniko-secret — from-file=config.json — namespace=devops-tools
```
#### 2. alpine/k8s - All-In-One Kubernetes tools (kubectl, helm, iam-authenticator, eksctl, kubeseal, etc)

We will use this container during the pipeline to deploy the .NET Core app on the cluster with kubectl.

## Running the pipeline using a Jenkinsfile

Here, i will explain my pipeline script:
```bash
pipeline {
  agent {
    label 'kubeagent'
  }
  stages {
    stage('Build and Deliver') {
      steps {
        container('kaniko') {
          sh '''
            /kaniko/executor --context `pwd` --dockerfile Dockerfile --destination shayts/elta-project:${BUILD_NUMBER}
          '''
        }
      }
    }
    stage('Deploy') {
      steps {
        container('k8s') {
          sh 'sed -i "s,IMAGE_NAME,shayts/elta-project:${BUILD_NUMBER}," netcore-app-deployment.yaml'
          sh "kubectl apply -f netcore-app-deployment.yaml -n prod"
        }
      }
    }
  }
}
```
First, the agent block specifies that this pipeline will be run on a Jenkins agent labeled kubeagent. Which is the pod we defined earlier in the kubernetes plugin configuration.

#### Build and Deliver Stage:

This stage is responsible for building and delivering the Docker image. During the image build process we actually compile the application and then packing it to a final docker image (You can check the Dockerfile for details) which is then delivered to my Docker Hub.
It uses a container named kaniko (explaine earlier), The Kaniko executor is used to build Docker images in a container without needing to mount the Docker socket.

#### Kaniko Build:

Inside the kaniko container, it runs the Kaniko executor with parameters:

--context: Specifies the build context, which is the current directory (pwd).

--dockerfile: Specifies the Dockerfile to use (Dockerfile).

--destination: the docker registry where the built image should be pushed (shayts/elta-project:${BUILD_NUMBER}).
#### Deploy Stage:
This stage is responsible for deploying the application to Kubernetes.

It uses a container labeled k8s (alpine/k8s - explained earlier)

Before applying the deployment, it modifies the netcore-app-deployment.yaml file to replace IMAGE_NAME with the actual image name and tag.
This is done using sed (stream editor) to perform a search and replace operation.

Finally, it applies the modified netcore-app-deployment.yaml file to the Kubernetes cluster in the prod namespace using kubectl. This updates the deployment with the new image.

## Deploying an Ingress NGINX Controller on the cluster

Right now we can access the .NET Core app and the Jenkins controller, which are deployed on the cluster, via NodePort.
using an nginx-ingress controller is better than NodePort for accessing your Kubernetes cluster because it provides more advanced routing capabilities, Load Balancing, and is more secure. the steps are:

1. Deploy the nginx-ingress controller with ingress-nginx-deploy.yaml, Which will set up the nessecery components, including a Loadbalancer on AWS.

2. Apply the ingress resources for the .NET Core app and Jenkins with netcore-ingress.yaml and jenkins-ingress.yaml. this will define the routing rules from the Loadbalancer to the matching services inside the cluster.

```bash
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
  namespace: devops-tools
spec:
  ingressClassName: nginx
  rules:
    - host: jenkins.shay.com
      http:
        paths:
          - pathType: Prefix
            path: /
            backend:
              service:
                name: jenkins-service
                port:
                  number: 8080
```
```bash
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
  namespace: prod
spec:
  ingressClassName: nginx
  rules:
    - host: netcore.shay.com
      http:
        paths:
          - pathType: Prefix
            path: /
            backend:
              service:
                name: netcore-app-service
                port:
                  number: 8000
```
3. Now, for testing purposes, we'll edit our /etc/hosts file and add the AWS Loadbalancer ip address to match the domains we've set up on the ingress resources. like this:

```bash
3.126.185.11    jenkins.shay.com
3.126.185.11    netcore.shay.com
```

And that's it! we're done! now we can access our jenkins controller and our app with 2 different sub-domains!


![App Screenshot](https://i.imgur.com/Daze52P.png)
![App Screenshot](https://i.imgur.com/pjAnDSz.png)
