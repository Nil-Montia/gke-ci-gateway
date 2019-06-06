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
Run the application with
```
kubectl apply -f gke-ci-gateway
```
and add the jenkins node, if you want automatic updates to the services.
```
cd gke-ci-gateway
kubectl apply -f jenkins
```
