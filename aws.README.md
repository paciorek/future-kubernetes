## MOVED TO projects/dask-kube

## UNDER CONSTRUCTION

# follow instructions to install the AWS CLI and eksctl:
# https://docs.aws.amazon.com/eks/latest/userguide/getting-started-eksctl.html

# eks seems slower than gke

eksctl version

aws configure # enter AWS credentials 

eksctl create cluster \
--name future \
--region us-west-2 \
--nodegroup-name standard-workers \
--node-type t2.micro \
--nodes 2 \
--nodes-min 1 \
--nodes-max 4 \
--ssh-access \
--ssh-public-key ~/.ssh/ec2_rsa.pub \
--managed


kubectl port-forward --namespace default svc/future-scheduler 8787:8786&


eksctl-0.31.0-rc.0 create cluster \
--name future2 \
--version 1.18 \
--region us-west-2 \
--nodegroup-name standard-workers \
--node-type t2.small \
--nodes 2 \
--nodes-min 1 \
--nodes-max 4 \
--ssh-access \
--ssh-public-key ~/.ssh/ec2_rsa.pub \
--managed

2020-11-13: trying the above - does newer eksctl, version 1.18 and nodegroup-name help?
seems like it works (using 'future2' and 't2.small')
but the worker pods keep restarting

if I connect and kill the R process, the pod dies
if I hae pod not start R worker, the pods stay alive
R process dies when I start manually in pod
strace shows quick exit:

read(3, "setup_kube()\n\0", 4096)       = 14
clock_gettime(CLOCK_REALTIME, {tv_sec=1605312151, tv_nsec=353901522}) = 0
getrusage(RUSAGE_SELF, {ru_utime={tv_sec=0, tv_usec=169019}, ru_stime={tv_sec=0, tv_usec=47325}, ...}) = 0
getrusage(RUSAGE_CHILDREN, {ru_utime={tv_sec=0, tv_usec=8417}, ru_stime={tv_sec=0, tv_usec=560}, ...}) = 0
brk(0x55e0892b9000)                     = 0x55e0892b9000
brk(0x55e0892e7000)                     = 0x55e0892e7000
brk(0x55e089316000)                     = 0x55e089316000
brk(0x55e089338000)                     = 0x55e089338000
brk(0x55e089359000)                     = 0x55e089359000
brk(0x55e08937a000)                     = 0x55e08937a000
read(3, "", 4096)                       = 0
rt_sigaction(SIGINT, {sa_handler=SIG_IGN, sa_mask=[], sa_flags=SA_RESTORER, sa_restorer=0x7fd29801d840}, {sa_handler=0x7fd298365a60, sa_mask=[INT], sa_flags=SA_RESTORER|SA_RESTART, sa_restorer=0x7fd29801d840}, 8) = 0
rt_sigaction(SIGQUIT, {sa_handler=SIG_IGN, sa_mask=[], sa_flags=SA_RESTORER, sa_restorer=0x7fd29801d840}, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=0}, 8) = 0
rt_sigprocmask(SIG_BLOCK, [CHLD], [], 8) = 0
clone(child_stack=NULL, flags=CLONE_PARENT_SETTID|SIGCHLD, parent_tidptr=0x7fff2d418dcc) = 17902
wait4(17902, [{WIFEXITED(s) && WEXITSTATUS(s) == 0}], 0, NULL) = 17902
rt_sigaction(SIGINT, {sa_handler=0x7fd298365a60, sa_mask=[INT], sa_flags=SA_RESTORER|SA_RESTART, sa_restorer=0x7fd29801d840}, NULL, 8) = 0
rt_sigaction(SIGQUIT, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=SA_RESTORER, sa_restorer=0x7fd29801d840}, NULL, 8) = 0
rt_sigprocmask(SIG_SETMASK, [], NULL, 8) = 0
--- SIGCHLD {si_signo=SIGCHLD, si_code=CLD_EXITED, si_pid=17902, si_uid=0, si_status=0, si_utime=0, si_stime=0} ---
close(3)                                = 0
exit_group(0)                           = ?
+++ exited with 0 +++

I tried to start the R worker manually and found this message when I set OUT not to be /dev/null

root@future-worker-7cffd55fd5-gfrz5:/#  Rscript -e 'setup_kube()' && Rscript --default-packages=datasets,utils,grDevices,graphics,stats,methods -e 'parallel:::.slaveRSOCK()' MASTER=future-scheduler PORT=11562 OUT=/tmp/foo TIMEOUT=2592000 XDR=TRUE &
[1] 18080
root@future-worker-7cffd55fd5-gfrz5:/# tail -f /tmp/foo
starting worker pid=18066 on future-scheduler:11565 at 00:11:03.265
starting worker pid=18096 on future-scheduler:11562 at 00:11:28.015
Error in socketConnection(master, port = port, blocking = TRUE, open = "a+b",  : 
  cannot open the connection
Calls: <Anonymous> ... tryCatchOne -> doTryCatch -> recvData -> makeSOCKmaster
In addition: There were 17 warnings (use warnings() to see them)
Execution halted

Is the port service on future-scheduler messed up?

paciorek@smeagol:/var/tmp/future-helm-chart (master)> kubectl get svc
NAME               TYPE           CLUSTER-IP       EXTERNAL-IP                                                               PORT(S)                          AGE
future-scheduler   LoadBalancer   10.100.243.192   af30c01782be145fd9b21cd9f0d8ab9e-1198671209.us-west-2.elb.amazonaws.com   11562:31032/TCP,8787:31797/TCP   39m
kubernetes         ClusterIP      10.100.0.1       <none>                                                                    443/TCP                          81m

go back to my SocketConnection code and test?
Ryan says perhaps I could have k8s start up whatever future is using to listen on 11562 on the scheduler
test socketConnection from worker to scheduler if scheduler is alreayd listening there

socket connection works fine

if I start socketConnection on scheduler and then run 
Rscript -e 'setup_kube()' && Rscript --default-packages=datasets,utils,grDevices,graphics,stats,methods -e 'parallel:::.slaveRSOCK()' MASTER=future-scheduler PORT=11562 OUT=/tmp/foo TIMEOUT=2592000 XDR=TRUE &
I get this:
starting worker pid=81 on future-scheduler:11562 at 01:26:58.578
                            Error in unserialize(node$con) : error reading from connection
Calls: <Anonymous> ... doTryCatch -> recvData -> recvData.SOCKnode -> unserialize
Execution halted

If I start the worker R process and then immediately try to run makeClusterPSOCK in RStudio on scheduler, it works

so maybe on GKE, try to run future-helm2.tgz and see if R worker proceses die given that nothing is listening on 11562 on the scheduler

could it be the version of k8s somehow if differetn on GKE and EKS


can test via

telnet future-scheduler 11562 # on worker
telnet localhost 11562 # on schduler


then do: 
kubectl port-forward --namespace default svc/future-scheduler 8786:8787

I can connect at 127.0.0.1:8786

Ryan says 8787:8787 shoudl be fine if smeagol not using 8787
he says that the port-forward is more secure - it sets up a bridge from teh VM to local machine and then connect to localhost

but
> cl <- makeClusterPSOCK(num_workers, manual = TRUE, quiet = TRUE)
Error in unserialize(node$con) : error reading from connection
> plan(cluster, workers=cl)
Error in plan(cluster, workers = cl) : object 'cl' not found


[✖]  AWS::EKS::Nodegroup/ManagedNodeGroup: CREATE_FAILED – "Your requested instance type (t2.micro) is not supported in your requested Availability Zone (us-west-2d). Please retry your request by not specifying an Availability Zone or choosing us-west-2a, us-west-2b, us-west-2c. (Service: AmazonEKS; Status Code: 400; Error Code: InvalidRequestException; Request ID: ae07b023-6f47-4121-8b12-e4f97735fd78; Proxy: null)"

## change back to micro and 'future' at some point


using up-to-date eksctl the cluster starts but with a Jhub/dask chart, not all pods start and never get external IPs

somewhat similar to R/future though there the pods keep restarting

NAME               TYPE           CLUSTER-IP      EXTERNAL-IP                                                              PORT(S)                          AGE
future-scheduler   LoadBalancer   10.100.36.102   aa24519f521474774a75d79ef2c89406-673288808.us-west-2.elb.amazonaws.com   11562:30827/TCP,8787:31016/TCP   21m

the external-ip is not reachable (and the jsonpath thing doesn't extract it)
plus kubectl get pods shows the pods failing

[ℹ]  eksctl version 0.30.0
[ℹ]  using region us-west-2
[ℹ]  deleting EKS cluster "future"
[ℹ]  deleted 0 Fargate profile(s)
[✔]  kubeconfig has been updated
[ℹ]  cleaning up AWS load balancers created by Kubernetes objects of Kind Service or Ingress
[ℹ]  2 sequential tasks: { delete nodegroup "standard-workers", delete cluster control plane "future" [async] }
[ℹ]  will delete stack "eksctl-future-nodegroup-standard-workers"
[ℹ]  waiting for stack "eksctl-future-nodegroup-standard-workers" to get deleted

Why does this hang? 

eksctl create cluster \
--name future \
--region us-west-2 \
--nodegroup-name standard-workers \
--node-type t2.micro \
--nodes 2 \
--nodes-min 1 \
--nodes-max 4 \
--ssh-access \
--ssh-public-key ~/.ssh/ec2_rsa.pub \
--managed \
--vpc-public-subnets --vpc-public-subnets subnet-41d5f224,subnet-81591cf6

## subnets obtained from AWS console; need to be in same VPC

[✔]  using existing VPC (vpc-c71445a2) and subnets (private:[] public:[subnet-81591cf6 subnet-41d5f224])
[!]  custom VPC/subnets will be used; if resulting cluster doesn't function as expected, make sure to review the configuration of VPC/subnets

Ok, so this seems to have been created correctly
1 of 4 worker pods starts and is accessible via kubectl exec -it <insert_name_of_a_worker> -- /bin/bash

can't connect to the Rstudio frontend; in fact follownig doesn't seem to work
export RSTUDIO_SERVER_IP=$(kubectl get svc --namespace default future-scheduler -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

how is it that helm/kubectl know how to interact with the cluster, particularly if started via console


subnets are created dynamically and don't match what is seen at
https://us-west-2.console.aws.amazon.com/vpc/home?region=us-west-2#subnets:sort=SubnetId

not clear how to modify mapPublicIpOnLaunch
https://aws.amazon.com/blogs/containers/upcoming-changes-to-ip-assignment-for-eks-managed-node-groups/


if start eks cluster via console how set number of nodes?

couldn't delete because of attached nodegroups, so ran
eksctl delete nodegroup --cluster future --name standard-workers
that is failing too; trying through console to delete nodegroup
failure seems to be because of a dependent ec2securitygroup
needed to delete securitygroups and network interfaces via ec2 console - a bit of a nightmare


paciorek@smeagol:/var/tmp/future-kubernetes (master)> eksctl create cluster \
> --name future \
> --region us-west-2 \
> --nodegroup-name standard-workers \
> --node-type t2.micro \
> --nodes 2 \
> --nodes-min 1 \
> --nodes-max 4 \
> --ssh-access \
> --ssh-public-key ~/.ssh/ec2_rsa.pub \
> --managed
[ℹ]  eksctl version 0.15.0
[ℹ]  using region us-west-2
[ℹ]  setting availability zones to [us-west-2a us-west-2d us-west-2c]
[ℹ]  subnets for us-west-2a - public:192.168.0.0/19 private:192.168.96.0/19
[ℹ]  subnets for us-west-2d - public:192.168.32.0/19 private:192.168.128.0/19
[ℹ]  subnets for us-west-2c - public:192.168.64.0/19 private:192.168.160.0/19
[ℹ]  using SSH public key "/accounts/gen/vis/paciorek/.ssh/ec2_rsa.pub" as "eksctl-future-nodegroup-standard-workers-ae:f0:5a:6b:3d:e7:ef:a3:e2:20:ed:40:66:ac:cb:5f" 
[ℹ]  using Kubernetes version 1.14
[ℹ]  creating EKS cluster "future" in "us-west-2" region with managed nodes
[ℹ]  will create 2 separate CloudFormation stacks for cluster itself and the initial managed nodegroup
[ℹ]  if you encounter any issues, check CloudFormation console or try 'eksctl utils describe-stacks --region=us-west-2 --cluster=future'
[ℹ]  CloudWatch logging will not be enabled for cluster "future" in "us-west-2"
[ℹ]  you can enable it with 'eksctl utils update-cluster-logging --region=us-west-2 --cluster=future'
[ℹ]  Kubernetes API endpoint access will use default of {publicAccess=true, privateAccess=false} for cluster "future" in "us-west-2"
[ℹ]  2 sequential tasks: { create cluster control plane "future", create managed nodegroup "standard-workers" }
[ℹ]  building cluster stack "eksctl-future-cluster"
[ℹ]  deploying stack "eksctl-future-cluster"



[ℹ]  building managed nodegroup stack "eksctl-future-nodegroup-standard-workers"
[ℹ]  deploying stack "eksctl-future-nodegroup-standard-workers"
[✖]  unexpected status "ROLLBACK_IN_PROGRESS" while waiting for CloudFormation stack "eksctl-future-nodegroup-standard-workers"
[ℹ]  fetching stack events in attempt to troubleshoot the root cause of the failure
[!]  AWS::EKS::Nodegroup/ManagedNodeGroup: DELETE_IN_PROGRESS
[✖]  AWS::EKS::Nodegroup/ManagedNodeGroup: CREATE_FAILED – "Nodegroup standard-workers failed to stabilize: [{Code: Ec2SubnetInvalidConfiguration,Message: One or more Amazon EC2 Subnets of [subnet-0d2944fddb54d2032, subnet-04bdc19cde2223ff0, subnet-0bad98cc3dd3bf7a5] for node group standard-workers does not automatically assign public IP addresses to instances launched into it. If you want your instances to be assigned a public IP address, then you need to enable auto-assign public IP address for the subnet. See IP addressing in VPC guide: https://docs.aws.amazon.com/vpc/latest/userguide/vpc-ip-addressing.html#subnet-public-ip,ResourceIds: [subnet-0d2944fddb54d2032, subnet-04bdc19cde2223ff0, subnet-0bad98cc3dd3bf7a5]}]"
[ℹ]  1 error(s) occurred and cluster hasn't been created properly, you may wish to check CloudFormation console
[ℹ]  to cleanup resources, run 'eksctl delete cluster --region=us-west-2 --name=future'
[✖]  waiting for CloudFormation stack "eksctl-future-nodegroup-standard-workers": ResourceNotReady: failed waiting for successful resource state
Error: failed to create cluster "future"

See here:
https://aws.amazon.com/blogs/containers/upcoming-changes-to-ip-assignment-for-eks-managed-node-groups/


eksctl delete cluster future

seems like this may not delete everything, e.g., still may be billed for NAT gatweay and/or nodegroup.
Check
https://us-west-2.console.aws.amazon.com/vpc/home?region=us-west-2#NatGateways

https://us-west-2.console.aws.amazon.com/vpc/home?region=us-west-2#securityGroups
https://us-west-2.console.aws.amazon.com/ec2/home?region=us-west-2#NIC
