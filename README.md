# future-kubernetes

Instructions for setting up and using a Kubernetes cluster for running R in parallel using the future package. A primary use of this would be to run R in parallel across multiple virtual machines in the cloud. Kubernetes provides the infrastructure to set things up so that the master R process and all the R workers are running and able to communicate with each other. 

At the moment, the instructions make use of Google Kubernetes Engine, but apart from the initial step of starting the cluster, I expect the other steps to work on other cloud provider's Kubernetes platforms.

The future package provides for parallel computation in R on one or more machines.

- <https://cran.r-project.org/package=future>
- <https://github.com/HenrikBengtsson/future>

These instructions rely on three Github repositories under the hood:

  - A [(slightly) hacked version of the future package](https://github.com/paciorek/future) that uses `kubectl` instead of `ssh` to start up the R workers
  - A [Docker container](https://github.com/paciorek/future-kubernetes-docker) that (slightly) extends the `rocker/rstudio` Docker container to add the (modified) future package and `kubectl`.
  - A [Helm chart](https://github.com/paciorek/future-helm-chart) based in large part on the [Dask helm chart](https://github.com/dask/helm-chart) that installs the Kubernetes pods, one pod running RStudio and acting as the master R process and (by default) four pods, each running one R worker process.
  
## Setting up your Kubernetes cluster

### Using Google Kubernetes Engine

#### Installing software to manage the cluster

You'll need to [install the Google Cloud command line interface (CLI) tools](https://cloud.google.com/sdk/install). Once installed you should be able to use `gcloud` from the terminal.

You'll also need to [install `kubectl`](https://kubernetes.io/docs/tasks/tools/install-kubectl)  to manage your cluster.

Finally you'll need to [install `helm`](https://helm.sh/docs/intro/install), which allows you to install packages on your Kubernetes cluster to set up the Kubernetes pods you'll need. These instructions assume Helm version 2 (e.g., Helm 2.16.3 for [Mac](https://get.helm.sh/helm-v2.16.3-darwin-amd64.tar.gz) or [Windows](https://get.helm.sh/helm-v2.16.3-windows-amd64.zip); I haven't yet tried using Helm version 3. 

You may be able to use the Google Cloud Shell and/or the Google Cloud Console rather than installing the Google Cloud CLI or kubectl. I need to look more into this.

#### Starting a Kubernetes cluster

Here is an example invocation to start up a Kubernetes cluster on Google's Kubernetes Engine. I suspect something similar will work for AWS (EKS) or Azure (Azure Kubernetes Service).

This asks for four n1-standard-1 (1 CPU) virtual machines. In general you'll want to have as many R workers (set via the Helm chart - see below) as total CPUs (set here) on the cluster. 

```
gcloud container clusters create \
  --machine-type n1-standard-1 \    
  --num-nodes 4 \
  --zone us-west1-a \
  --cluster-version latest \
  future
```

So if you had instead asked for n1-standard-2 (two CPUs per node) and four nodes, you'd want to have eight R workers.

Now you need to run some `kubectl` commands to modify your cluster. Make sure to provide your user name after the `--user` in the first command.

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
tar -cvzf future-helm.tgz .
helm install ./future-helm.tgz 
sleep 30
```

Make sure to wait a bit (e.g., 30 seconds as above) to let the pods start up. You'll see a message that gives the name of the "release", basically the name of your collection of pods and tells you how to connect to the RStudio front-end using your browser. The name of the release will be something like `ardent-porcupine`.

Note: Below I have instructions for installing additional R packages on your cluster. If you install any packages that take a substantial amount of time (e.g., anything relying on Rcpp), it could take multiple minutes or more for the pods to start up. The RStudio interface would NOT be available during this time.

You can check the pods are running with:

```
helm status ardent-porcupine
kubectl get pods
```

#### Connecting to the the RStudio instance running in your cluster.

Once your pods have finished starting up, you can connect to your cluster via the RStudio instance running in the master pod on the cluster.

Run the following.

```
export RSTUDIO_SERVER_IP=$(kubectl get svc --namespace default future-scheduler -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
export RSTUDIO_SERVER_PORT=8787
echo http://$RSTUDIO_SERVER_IP:$RSTUDIO_SERVER_PORT
```

Take the URL printed by that last line and connect to it in a browser tab. You can then login to RStudio using the username `rstudio` and password `future`.


#### Setting up the future `plan`

Now you should be able to do the following in RStudio to create your plan. This will start up the R workers and connect them to the master R process. 

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

## Removing your Kubernetes cluster

Make sure to do this or the cloud provider (Google in this case) will keep chargning you, hour after hour after hour.

```
gcloud container clusters delete future --zone=us-west1-a
```

## Modifying your cluster

Some simple modifications are:

  - increasing the number of R workers
  - adding additional R packages to your cluster
  - changing the RStudio password

For the most part you'll want to make these modifications by editing the files in the `future-helm-chart` directory BEFORE you run `helm install`.

However as illustrated for increasing the number of R workers, you can use `kubectl edit deployment future-worker` or `kubectl edit deployment future-scheduler` to modify the properties of the worker or scheduler pods on the fly while they are running. In some cases new pods will be created in place of the original pods.

### Increasing the number of R workers

To increase the number of R workers (before running `helm install` above), go into the  `future-helm-chart` directory and edit the `values.yaml` file. In particular, you'll need to modify the `replicas` line that is in the 'worker' stanza (the block of code under `worker:`. Don't modify the 'replicas' line in the 'scheduler' stanza.

You can also modify the number of workers after having run `helm install` by invoking the following:

```
kubectl edit deployment future-worker
```

This will put you into an editor and you can modify the `replicas` line in the `spec` stanza. Once you exit the editor, your new worker pods should start. If you rerun `helm status ardent-porcupine`, you should see the additional worker pods running.

### Adding additional R packages

Go into the `future-helm-chart` directory and edit the `values.yaml` file. Simply modify the lines that look like this:

```
  env:
  #  - name: EXTRA_R_PACKAGES
  #    value: data.table
```

by removing the "#" comment character and putting the R packages you want installed in place of 'data.table', with the names of the packages separated by spaces.

In many cases you may want these packages installed on both the scheduler (where RStudio runs) and on the workers. If so, make sure to modify the lines above in both the `scheduler` and `worker` stanzas.

WARNING: installing R packages can take substantial time. This is done when the pods are started, so the RStudio interface will not be available until all packages are installed in the scheduler pod.

### Changing the RStudio password

Simply modify the `rstudio_password` value in `values.yaml` file in the `future-helm-chart` directory.

## Troubleshooting

It can be helpful to know how to get a terminal session in your pods or in the virtual machines that host your pods.

You can access the running pods via `kubectl` like this. First we'll set environment variables that hold the names of the scheduler (aka master) pod and the worker pods.

```
export SCHEDULER=$(kubectl get pod --namespace default -o jsonpath='{.items[?(@.metadata.labels.component=="scheduler")].metadata.name}')
export WORKERS=$(kubectl get pod --namespace default -o jsonpath='{.items[?(@.metadata.labels.component=="worker")].metadata.name}')

## access the scheduler pod:
kubectl exec -it ${SCHEDULER}  -- /bin/bash
## access the 'first' worker pod:
echo $WORKERS
kubectl exec -it <insert_name_of_a_worker> -- /bin/bash
```

Suppose you need to actually connect to the VMs on which the pods are running.

```
kubectl get nodes
## now, with one of the nodes, 'gke-future-default-pool-8b490768-2q9v' in this case:
gcloud compute ssh gke-future-default-pool-8b490768-2q9v --zone us-west1-a
```

Regardless of whether you connect to a pod or a virtual machine, you should then be able to use standard commands such as `top` and `ps` to check the running processes.

This is a good way to verify that your computation is load-balanced across the vrtual machines. You want to have as many running R workers (one worker per pod) as there are compute cores on the virtual machine.

## How it works (information for developers)

 1. The future package uses ssh to start each R worker process and set up a socket connection between the worker and the master R process. While it's probably possible use SSH between Kubernetes pods, it's more natural to use `kubectl`. So in a [fork of the future package repository](https://github.com/paciorek/future) I've modified the package to use `kubectl` instead of SSH and to make sure of some new environment variables can be set via `Rprofile.site` (see item #2 below).

    - In fact, even allowing use of `kubectl` on the pods is probably not how one would really want to use future with Kubernetes. Presumably one could have Kubernetes start up the R worker processes and set up the socket connections.

    - Furthermore, one might want to create a `kubernetes` plan rather than hacking the `cluster` plan.

2. The pods run a [modified version](https://github.com/paciorek/future-kubernetes-docker) of the Rocker Rstudio docker image. The modification installs `kubectl` as well as the modified version of the R future package (see item #1 above) (plus the `future.apply` and `doFuture` packages). In addition two R functions are inserted into the system `Rprofile.site` file. One of these functions (`setup_kube`) allows one to install additional R packages and to set various environment variables that are needed by the modified future package (item #1 above). The other function (`get_kube_workers`) allows a user to determine the names of the workers for use with `future::plan()`.

    - Note that version 3.6.2 of the `rocker/rstudio` Docker container is needed because older rocker/rstudio containers set older MRAN repositories, which pull in a version of the `globals` package that is incompatible with the current `future` package.

3. The [helm chart](https://github.com/paciorek/future-helm-chart) creates a scheduler pod running RStudio server and worker pods that one can connect to from the scheduler. All the pods run the modified Docker image (item #2). When the pods start, they invoke the `setup_kube` function (item #2), which installs any additional R packages and (on the scheduler only) injects the environment variables needed in item #1 into the system level R environment file `/usr/local/lib/R/etc/Renviron`. These environment variable are then available when the user invokes `future::plan()`. Note that some of the environment variables are only available once the pods start running based on invoking `kubectl`, so they cannot be hard-coded into the Docker image. 

     - This chart is really just a simplification of the [Dask helm chart](https://github.com/dask/helm-chart). 



## Acknowledgments

This material borrows heavily from work on setting up Kubernetes clusters running Python's Dask package by the Berkeley Climate Impacts Lab and from discussions with Ian Bolliger, a recent PhD graduate from the Energy and Resources Group at UC Berkeley.

Here is some documentation on using [Jupyter with Kubernetes](https://zero-to-jupyterhub.readthedocs.io/en/latest/setup-jupyterhub/setup-jupyterhub.html) and using [Dask on Kubernetes](https://docs.dask.org/en/latest/setup/kubernetes.html).

Thanks also to Ryan Lovett of the UC Berkeley Statistical Computing Facility for helping me with Kubernetes.

