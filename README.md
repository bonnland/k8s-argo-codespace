# k8s-argo-codespace
This repository provides a preconfigured environment that starts a Kubernetes cluster on Docker when launched in GitHub Codespaces

## Getting Started
Fork this repository in to your own GitHub account. Once the repository has been forked select the button labeled `<> Code` in the upper right of the webpages body. This opens a dropdown containing 2 tabs. The default tab that opens is the Local tab and contains information on how to Clone the repository. Select the Codespaces tab and use the `Create codespace on main` button to launch a new tab in your browser containing an Interactive Development Environment (IDE) with the repository code.  

When the new Codespace launches a script will setup the environment and automatically run `minikube start` to launch a Kubernetes (K8s) cluster. As long as minikube starts without any issues it can now be used to install apps and get familiar with how Argo CD deploys and manages applications. 

### Helpful Commands

```bash
# Watch pods in real-time
kubectl get pods -n argocd -w

# Check pod details if something goes wrong
kubectl describe pod <pod-name> -n argocd

# View application logs
kubectl logs -f deployment/flask-demo -n argocd
```

## Install Argo CD

The first steps we will take are to create a namespace in K8s named argocd and then install Argo CD into that namespace. This can be accomplished by pasting the following commands in to the terminal window in VS Code.

```
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl apply -f argocd-insecure-install.yaml
kubectl rollout restart deployment argocd-server -n argocd
```

Wait until all the pods in the argocd namespace are up and running before proceeding to the next steps

```
kubectl get pods -n argocd --watch
```

When successful it will look like this:

```
/workspaces/k8s-argo-codespace (main) $ kubectl get pods -n argocd
NAME                                                READY   STATUS    RESTARTS   AGE
argocd-application-controller-0                     1/1     Running   0          8m43s
argocd-applicationset-controller-7c77597d54-5qt76   1/1     Running   0          8m44s
argocd-dex-server-5d86b78484-tvbjh                  1/1     Running   0          8m44s
argocd-notifications-controller-78f5bf5947-l2g57    1/1     Running   0          8m43s
argocd-redis-757c9855f5-ssd7q                       1/1     Running   0          8m43s
argocd-repo-server-75c657669b-bp866                 1/1     Running   0          8m43s
argocd-server-7bbfdb874-s5h4l                       1/1     Running   0          8m43s
```

## Access Argo CD

### Expose the ArgoCD UI

`kubectl port-forward svc/argocd-server -n argocd 8002:80`

The Argo CD UI will now be available at [https://127.0.0.1:8002](https://127.0.0.1:8002)

**Note:** The `port-forward` command will run continuously and occupy the terminal. A new terminal tab (Terminal → New Terminal) will need to be opened to continue the tutorial. 

### Get admin password

The default username is admin and we need to get the password from a secret stored on the K8s cluster

`kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo`

With that password you can now login to the Argo CD with the admin username. 

## Install flask-helm chart

This repository contains a Helm chart that runs a very basic Flask application. There is a README file in the flask-helm directory that explains it's content in greater detail. From the Argo CD UI, after getting logged in, select the button in the upper right for `+ NEW APP`. This brings up a window where an application can be added to Argo for continuous deployment. Inside the new window there are a few fields that need to be filled out to successfully deploy the flask-helm directory. 

### General settings

Application Name: A descriptive name for the actual application. In this case use something like flask-demo

Project Name: Argo CD allows applications to be divided in to different projects and can take advantage of things like Role Based Access Controls (RBAC). For this example using the default project is fine. Click on the line below Project Name and open a drop down to select the default project. 

SYNC POLICY: This decides whether the application should automatically sync to match the source repository or if a manual sync should be required. Change this from Manual to Automatic.

The next section contains a number of checkboxes. Leave these all unchecked. 

### Source settings

Repository URL: The full URL of the repository ending in `.git`. This can be copied directly from the repository under the same `<> Code` button where the codespace was launched but under the Local tab.  

Revision: HEAD is selected by default. It is fine like this for the example. If you were working under a different repository branch you would specify that here.

Path: The path to the Helm chart inside the repository. Argo CD automatically recognizes where Helm charts are stored. If you click the line under Path it will automatically show the flask-helm directoy. Select this to wrap up the Source section. 

### Destination

Cluster URL: `https://kubernetes.default.svc` will show up by default when the line under Cluster URL is selected for this instance. A different Kubernetes endpoint could be specified here, but for this example the default is correct.

Namespace: Use the `argocd` namespace that was created after minikube started. A new namespace can be created with the `kubectl create namespace` command. For demonstration purposes `argocd` is fine. 

### Create

Those are all the fields required to deploy the application. Select the `CREATE` button at the top of the form. There will be a new pane in the Applications tab that shows the status of the newly added application. If everything worked correctly the status should move to Healthy and Synced in a few minutes. The container image needs to be downloaded from the container registry which may take a few minutes depending on your internet connection.  

### Expose the Flask UI

Once the application has synced fully in Argo CD port-forwarding can be run to expose the UI and make the Flask UI accessible via your browser with

`kubectl port-forward svc/flask-demo -n argocd 8001:5000`

After running that access the Web UI by browsing to [http://127.0.0.1:8001/](http://127.0.0.1:8001/)

### Real time changes

Argo CD looks to the repository every ~3 minutes to see if any changes have been made. If you don't want to wait for the automated sync you can always use the Sync button at the top to Sync manually. 

#### Scaling: Replicas from 1 to 3
Let's scale from 1 to 3 running pods:

1. Navigate to the `flask-helm/values.yaml` file in your codespace
2. Change `replicaCount: 1` to `replicaCount: 3`
3. Commit your changes:
   - Click the Source Control icon (should show a green circle with "1")
   - Add a commit message like "Scale to 3 replicas"
   - Click **Commit** and accept the prompts to stage and push changes

When we go back to the Argo CD UI we will see a sync within 3 minutes to match the new repository state. The sync will show a comment that matches the commit message used in our change. When the sync finishes we should see 3 running pods. The page may need to be refreshed if this is not shown. 

We can also run `kubectl get pods -n argocd` to see the 3 new running flask pods. 

#### Update Environment Variable

The Flask app displays a welcome banner using an environment variable. Let's customize it:

1. In `flask-helm/values.yaml`, find: `welcomeMessage: "Welcome to the Flask Demo!"`
2. Change it to your own message
3. Commit and push the changes
4. Wait for Argo CD to sync (or manually sync)

**Note:** This update will restart the pods, terminating your Flask port-forward. You'll need to restart the port-forward command to see your changes.

#### Bad image tag

Let's see what happens when we make a mistake:

1. In `values.yaml`, change the image tag from `"2025-06-24.16.33"` to `"bad-tag"`
2. Commit and push this "mistake"
3. **Observe what happens:**
   - Argo CD creates a new ReplicaSet
   - New pods fail with `ImagePullErr`
   - **Important:** Your original pods keep running! The site stays up.
   - Argo CD shows the application as "Degraded" but not completely failed

4. **Recovery:** Revert the image tag to the working version and push again

This style of GitOps is resilient! Bad deployments don't necessarily take down running services.

## Conclusion

This tutorial provides a brief introduction into how the CIRRUS cluster is setup with Argo CD, how applications are added to Argo CD, and how development work on Kubernetes via CI/CD can be handled. 

## Quick Reference Commands
```bash
# Install Argo CD
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl apply -f argocd-insecure-install.yaml
kubectl rollout restart deployment argocd-server -n argocd

# Watch all pods
kubectl get pods -n argocd -w

# Get admin password  
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo

# Port forward Argo CD
kubectl port-forward svc/argocd-server -n argocd 8002:80

# Port forward Flask app
kubectl port-forward svc/flask-demo -n argocd 8001:5000
```