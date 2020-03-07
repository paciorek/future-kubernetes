# future-kubernetes

Instructions for setting up and using a Kubernetes cluster for runnign R in parallel using the future package.

[UNDER CONSTRUCTION]

At the moment, the instructions are only available for Google Kubernetes Engine, but I plan to look into use with Azure and AWS.

The future package provides for parallel computation in R on one or more machines.

- <https://cran.r-project.org/package=future>
- <https://github.com/HenrikBengtsson/future>

These instructions rely on three Github repositories under the hood:

  - A (slightly) hacked version of the future package that uses `kubectl` instead of `ssh` to start up the R workers
  - A Docker container that (slightly) extends the `rocker/rstudio` Docker container to add the (modified) future package and `kubectl`.
  - A Helm chart based in large part on the [Dask helm chart](https://github.com/dask/helm-chart) that installs the Kubernetes pods, one pod running RStudio and acting as the master R process and (by default) four pods, each running one R worker process.
  
## Setting up your Kubernetes cluster

### Using Google Kubernetes Engine

#### Installing software to manage the cluster 
You'll need to [install the Google cloud command line interface tools](https://cloud.google.com/sdk/install). Once installed you should be able to use `gcloud` from the terminal.

You'll also need to [install `kubectl`](https://kubernetes.io/docs/tasks/tools/install-kubectl)  to manage your cluster.


#### Starting a Kubernetes cluster

This asks for four n1-standard-1 (1 CPU) virtual machines. In general you'll want to have as many R workers (set via the Helm chart - see below) as total CPUs (set here) on the cluster. 

```
gcloud container clusters create \
  --machine-type n1-standard-1 \    
  --num-nodes 4 \
  --zone us-west1-a \
  --cluster-version latest \
  future
```

You may be able to do this instead through the Google Cloud Shell and/or the Google Cloud Console. I need to look more into this.

Now you need to run some `kubectl` commands to modify your cluster. Make sure to provide your user name after the `--user`.

```
kubectl create clusterrolebinding cluster-admin-binding \
  --clusterrole=cluster-admin \
  --user=username@domain.com

## This is needed so kubectl can be run on the Kubernetes pods
kubectl create rolebinding all-access \
  --clusterrole=cluster-admin \
  --serviceaccount=default:default

## This will change with the new version of Helm (>= 3.0.0)
kubectl --namespace kube-system create serviceaccount tiller
kubectl create clusterrolebinding tiller --clusterrole cluster-admin --serviceaccount=kube-system:tiller
helm init --service-account tiller --wait

kubectl patch deployment tiller-deploy --namespace=kube-system --type=json --patch='[{"op": "add", "path": "/spec/template/spec/containers/0/command", "value": ["/tiller", "--listen=localhost:44134"]}]'
```

Now we're ready to install the helm chart that creates the pods (essentially containers) on the Kubernetes cluster. There will be one pod running R studio and (by default) four pods running R workers.

```
git clone https://github.com/paciorek/future-helm-chart
cd future-helm-chart
tar -cvzf future-helm.tgz
helm install ./future-helm.tgz 
sleep 30
```

Make sure to wait a bit (e.g., 30 seconds as above) to let the pods start up. You'll see a message that gives the name of the "release", basically the name of your collection of pods and tells you how to connect to the RStudio front-end using your browser. The name of the release will be something like `ardent-porcupine`.

You can check the pods are running with:

```
helm status ardent-porcupine
```

#### Connecting the the RStudio instance running in your cluster.

```
export RSTUDIO_SERVER_IP=$(kubectl get svc --namespace default future-scheduler -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
export RSTUDIO_SERVER_PORT=8787
echo http://$RSTUDIO_SERVER_IP:$RSTUDIO_SERVER_PORT
```

Take the URL printed by that last line and connect to it in a browser tab. You can then login to RStudio using the username `rstudio` and password `future`.


#### Setting up the future `plan`

Now you should be able to do this in R to create your plan. This will start up the R workers and connect them to the master R process. 

```{r}
library(future)
workers <- get_kube_workers()
plan(cluster, workers = workers, revtunnel = FALSE) 
```

## Example usage of your cluster

Once you've set up your plan, the following example should run in parallel.

```{r}
library(future.apply)
output <- future_lapply(1:40, function(i) mean(rnorm(1e7)), future.seed = 1)
```


## Modifying your cluster

Some simple modifications are:

  - increasing the number of R workers
  - adding additional R packages to your cluster
  - changing the RStudio password

## Troubleshooting

ssh to VMs
ssh to pods


### is procps (ps, top) installed in the pods?

## More details and notes


put a setup_kube function in Rprofile.site. This is run when pods are created and on the scheduler it installs user-requested packages and sets environment variables based on kubectl (only available at pod run time) by modifing Renviron so these variables are availbe to all R sessions. On the worker only the user-requested R packages are installed.

The paciorek/future-kubernetes container already has a modified version of the future package that makes use of the kubernetes cluster installed. It also has kubectl installed - this is needed for the scheduler to contact the workers from within R and setup the socket connection between workers and master R process.
The container also has setup_kube n Rprofile.site

Note that rocker/rstudio:3.6.2 is needed because older rocker/rstudio sets older MRAN repo, and that pulls in a version of globals incompatible with current future package.

## Acknowledgments

This material borrows heavily from 
