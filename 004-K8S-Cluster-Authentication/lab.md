# Authentication and Authorization

The goal of this lab is to enhance the security of our cluster using built-in Kubernetes primitives. We will explore authentication and authorization strategies and apply them to our GKE cluster.

We will be setting up our cluster with two namespaces - `development` and `production`. User 1 (our admin user) will have full-access to administer the objects in the cluster and we will restrict access to only the `development` namespace for User 2 by using RBAC policies.

### Lab Setup
Your GCP Project has two end users provisioned with the following permissions:
```
User 1: <your-username>@manicode.us
Roles:
Kubernetes Engine Admin
Editor

User 2: <your-intern-email>@manicode.us
Roles: 
Minimal GKE Role
Browser
```
The role titled `Minimal GKE Role` is a custom role in GCP. It includes the bare-minimum permissions to be able to access the cluster but not the resources within. The `Minimal GKE Role` has only the following permissions:
```
container.apiServices.get
container.apiServices.list
container.clusters.get
container.clusters.getCredentials
```
### Task 1: Launch Your Infrastructure
First, we will spin up our application in both a `development` and `production` namespace. 

Note: You should be logged in to Cloud Shell using the custom account given to you on the slip of paper to run the following commands, not <your-intern-email>@manicode.us.

We need to retrieve the credentials of our running cluster using the `gcloud get-credentials` command. This command updates our kubeconfig in Cloud Shell file with appropriate credentials and endpoint information to point kubectl at a specific cluster in Google Kubernetes Engine. 

```
# Use gcloud get-credentials to retrieve the cert
gcloud container clusters get-credentials <cluster-id> --zone us-west1-a --project <project-id>
```
Now we launch our pods and services for each Namespace:
```
# in the manifests/development directory

kubectl create -f link-unshorten-ns.yaml
kubectl create -f link-unshorten-service.yaml
kubectl create -f link-unshorten-deployment.yaml
```
Make sure our pods are running in the `development` namespace:
```
kubectl get pods --namespace=development
```

Do the same for the production namespace:
```
# in the manifests/production directory

kubectl create -f link-unshorten-ns.yaml
kubectl create -f link-unshorten-service.yaml
kubectl create -f link-unshorten-deployment.yaml
```
Ensure pods are running without error in both namespaces:
```
kubectl get pods --all-namespaces
```

Take note of this process. Our user has full administrative access to our cluster due to being provisioned with the `Kubernetes Engine Admin` role. We will now see how RBAC helps give us granular access control at the object-level within our cluster.

### Task 2: Authenticate as a Developer
We will now log in using a separate user who has very locked down access to the entire project. In an incognito window browse to `cloud.google.com` and authenticate with the user `<your-intern-email>@manicode.us` and the same password that was provided to you for the admin user. 

Note: *Using the same password for multiple accounts is bad. Don't do this at home.* 

Now open up Cloud Shell and use the following `gcloud get-credentials` command to retrieve the credentials for your user so we can start interacting with the cluster. This is the same cluster you just launched the `production` and `development` namespace / infrastructure in. 

```
gcloud container clusters get-credentials <cluster-id> --zone us-west1-a --project <project-id>

```
Now, attempt to run some `kubectl` queries on the cluster.
```
kubectl get pods --namespace=production
kubectl get pods --namespace=development
kubectl get secrets 
kubectl run link-unshorten --image=jmbmxer/link-unshorten:0.1 --port=8080
```
These should all fail with a `Forbidden` error. While <your-intern-email>@manicode.us does technically have an account on the cluster, RBAC is stopping it from accessing any of the objects.

### Task 3: Create RBAC Rules 
Our user `<your-intern-email>@manicode.us` is an intern so we only want to grant access to read pods in the `development` namespace and nothing more. We will use RBAC to enforcy a policy

Switch back to your Cloud Shell for User 1 (the administrative user) and run the following commands:

(!)You must grant your user the ability to create roles in Kubernetes by running the following command.
```
kubectl create clusterrolebinding cluster-admin-binding \
  --clusterrole cluster-admin \
  --user $(gcloud config get-value account)
```
Now, open the file `user-role.yaml` in the `manifests/role` directory and replace <your-intern-email> with the one provided to you. It will be the same as your admin account but with the word `intern` at the end (eg. `manicode0003intern@manicode.us`). 
```
# In the manifests/role directory
kubectl create -f .
kubectl get role --all-namespaces
```

### Task 4: Verify Pods can be Accessed by the Intern

Switch back to the Cloud Shell for `<your-intern-email>@manicode.us` and run the following commands:
```
kubectl get pods --namespace=development
# success
kubectl get pods --namespace=production
# fail
```

We have successfully limited access using RBAC.

### Task 5: Cleanup
In the `manifests` directory:
```
kubectl delete -f development -f production -f role
```

