## Customizing the Docker Image

In a real application of this repository, it is likely that you will update the dockerfile to reflect your needs. In the following sections we deal with a few noteworthy issues that you may encounter: 

1. Secrets in the dockerfile, and;
2. Working with Amazon ECR

### Using Secrets in the dockerfile

The easiest way to deal with secrets is to supply them to your containers at build time using build arguments. In the example below, the `ARG` directive is used in the Dockerfile followed by an `ENV` directive to give the build access to an argument passed via `--build-arg`. For this case, our goal is to install R packages from a proprietary (private) github repository. In our Dockerfile we would include the following:

```
# Dockerfile

# ...

ARG github_pat
ENV GITHUB_PAT=$github_pat

# ...

RUN Rscript -e 'remotes::install_github("organization/repository")'
```

The function `remotes::install_github()` will use the environment variable to install the package and it will be ready when you spin up your container. 

All that's left to do is build the container: 

```
docker build -t <container_name> . --build-arg github_pat='YOUR_PAT'
```

This same concept could be extended to utilize as many arguments as your build process requires. 

#### .Rprofile

Another useful trick, especially if you have a large number of secrets, is copying an .Rprofile file to the active user directory of your Rstudio pods. 

```
# Dockerfile

COPY .Rprofile .Rprofile
COPY .Rprofile /home/rstudio/.Rprofile 
```

You may be wondering why you need to copy it to the root directory `/` and also the user directory. This is because in actuality only your scheduler will be running code from `/home/rstudio`, your workers will be running from `/` and would not load your profile into the environment from `/home/rstudio`. 

WARNING: This is only appropriate to do when you are using a private docker registry, everything on a public container is public!

## Working with Amazon ECR

AWS provides a private container registry that can be quite cost-efficient and secure when using proprietary code. The steps below, provide an overview of how one can integrate ECR with this repository. 

### 1. Create an ECR Repository

1. Login to the AWS console. 
2. Select your region in the top right. 
3. In the search box type "elastic container registry"
4. On the left hand side click "Repositories" under the subtitle "Amazon ECR"
5. Then click "Create Repository" on the top right of the inner window.

Name your repository appropriately to your containers purpose. Take note of the URI, you will need this for future steps. It will be in the form: 

```
<aws-account-id>.dkr.ecr.<aws-region>.amazonaws.com/<container-name>
```

### 2. Login to ECR Using Docker

To login to "docker" using ECR, you would need to change the account number and region accordingly with the following command: 

```{shell}
aws ecr get-login-password --region <aws-region> | docker login --username AWS --password-stdin <aws-account-id>.dkr.ecr.<aws-region>.amazonaws.com
```

### 3. Preparing Your Container

Using the dockerfile in this repository as an example: 

```{shell}
# build
docker build -t <container-name> . --build-arg github_pat='YOUR_GITHUB_PAT'
# tag, in this example "latest" is used
docker tag <container-name>:latest <aws-account-id>.dkr.ecr.<aws-region>.amazonaws.com/<container-name> 
# push
docker push <aws-account-id>.dkr.ecr.<aws-region>.amazonaws.com/<container-name>
```

### 4. Give your Cluster Access to ECR

IMPORTANT: This step occurs before you install the helm chart (before `helm install ...`), but after you have bound the admin (after `kubectl create clusterrolebinding ...`)

1. Open the helm chart YAML file called "values.yaml". Uncomment the lines: 

```
...
   pullSecrets:
     - name: regcred
...
```

Assuming you are using the same scheduler and worker container, you will need to uncomment this line in both sections of the YAML files. Next, you must tell kubernetes/helm where to find the configuration file to create your regcred. The file specified in the example path is what you created by running the command in step 2, above. 

WARNING: ECR automatically invalidates your credential after 12 hours. This means that you may experience problems accessing your ECR service unless you have run the command in step 2 within the last 12 hours. It is common practice to set this command on a chron. However, keep in mind that the command below must also be run after a refresh and with an active set of pods. 

```
kubectl create secret generic regcred \
    --from-file=.dockerconfigjson=/home/<YOUR-USER>/.docker/config.json \
    --type=kubernetes.io/dockerconfigjson
```

### 5. Tar and Install Helm Chart

Now you can continue with your typical process by taring the helm chart and installing it: 

```
tar czf future-helm.tgz -C future-helm-chart .
helm install --wait my-release ./future-helm.tgz
```

Often this process may be iterative, remember that you must first remove a helm chart to install a similarly named one. This is what you would do if you made changes to the container, or the helm chart.

```
helm uninstall my-release 
tar czf future-helm.tgz -C future-helm-chart .
helm install --wait my-release ./future-helm.tgz
```

