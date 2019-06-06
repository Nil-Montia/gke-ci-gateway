# Login Authentication Service

This project provides an easily set up login authentication service, storing users in a mongoDB database and providing session tokens to users while they are logged in. 

# Prerequisites

The service is set up using Kubernetes on Google Cloud, so you will need the Google Cloud SDK installed and connected to a billable GCP account, or to be on its Cloud Shell. Your system will also need to be a linux distribution with at least two cores and 4GB of RAM.

# Application Structure

![text](login-service-flow.png)

The diagram shows the workflow of services of this application: the gateway, built on nginx, redirects requests as appropiate. The user only interacts with the authentication service and client, and the dashboard once they are logged in.

Users register by going to http://URL/authentication/register (where "URL" is your URL), after which they will receive an activation link for their account. Aftewards, they can login by going to http://URL/authentication/login, or clicking "Back to Login" on the register page.

# Setup

Once on the shell, create a cluster by running
``` 
gcloud container clusters create [CLUSTER_NAME] --zone [COMPUTE_ZONE]
```
and authenticate kubernetes.
```
gcloud container clusters get-credentials [CLUSTER_NAME] --zone [COMPUTE_ZONE]
```
Clone this repository.
```
git clone https://github.com/Nil-Montia/gke-ci-gateway.git
```
You will have to provide the application with some enviroment variables. The email service needs an email account from which to contact users; set its email address, password and name of the service in the email-service-deploy.yaml file.

Run the application with:
```
kubectl apply -f gke-ci-gateway
```
Once the services are running, the LoadBalancer will be assigned an IP. Make a note of it, and edit authentication-service-deploy.yaml to substitute in the correct IP for the ACTIVATION_LINK environment variable. Then run the following command to complete setup:
```
kubectl apply -f gke-ci-gateway
```
This will configure the authentication service to provide the correct validation link.

# CI with Jenkins

Jenkins jobs can be set up to do CI with the cluster. The cluster images are currently pulled from nilmontia/image, but this can be replaced in the \*-deploy.yaml files to use your own, and respond to push events to their corresponding repositories. To execute a rolling update of the nodes, clone this repository and apply the yaml files again.
```
git clone https://github.com/Nil-Montia/gke-ci-gateway.git
kubectl apply -f gke-ci-gateway
```
If only particular files have been updated and you only wish to reapply those, simply run
```
kubectl apply -f gke-ci-gateway/"filename"
```
where "filename" is the name of the updated file.

# Setting up the Jenkins node

To set up a node with jenkins, you will need an image with kubernetes and docker. The jenkins deployment yaml currently points to such an image, but this can be changed in the file (gke-ci-gateway/jenkins/04_deployment.yaml). The Dockerfile could be set up like this:

```
FROM jenkins/jenkins:latest
USER root
RUN curl https://get.docker.com | bash
RUN curl -L "https://github.com/docker/compose/releases/download/1.23.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
RUN chmod +x /usr/local/bin/docker-compose
RUN usermod -aG docker jenkins
# GCloud & Kubectl
ENV GCLOUD_REMOTE_DOWNLOAD="https://dl.google.com/dl/cloudsdk/channels/rapid/downloads/google-cloud-sdk-213.0.0-linux-x86_64.tar.gz"
ENV GCLOUD_LOCAL_DOWNLOAD="/tmp/google-cloud-sdk.tar.gz"
RUN wget "${GCLOUD_REMOTE_DOWNLOAD}" -O "${GCLOUD_LOCAL_DOWNLOAD}"
RUN tar xzvf ${GCLOUD_LOCAL_DOWNLOAD} -C /opt
RUN ln -s /opt/google-cloud-sdk/bin/gcloud /usr/bin/gcloud
RUN gcloud components install -q kubectl
RUN ln -s /opt/google-cloud-sdk/bin/kubectl /usr/bin/kubectl
USER jenkins
```
This also provides it with docker-compose.

Once the appropiate images are ready, run
```
kubectl apply -f gke-ci-gateway/jenkins
```
This will create a jenkins node with a service account and role allowing it to run updates on the cluster. Soemtimes, the cluster may not have enough clusters. If that happens, run the following command to add nodes:
```
gcloud container clusters resize <cluster-name> --node-pool <default-pool> --num-nodes <number-of-nodes> --region <cluster-region>
```

However, the Jenkins container will not have access to correct group to use the docker socket. To fix this, run
```
kubectl describe pod jenkins
```
and make a note of the host node and the Jenkins container id.

Connect to the host node and run
```
docker exec -it -u root container-id bash
```
which will allow you to run commands in the Jenkins container as the root user. Then, give jenkins the right permissions by running
```
ls -al /var/run/docker.sock /* make a note of the docker group id */
groupdel docker
groupadd -g "group-id" docker
usermod -aG docker jenkins
```
Exit the bash (using the command "exit") and finally, run the command below to restart the container.
```
docker restart "container-id"
```

