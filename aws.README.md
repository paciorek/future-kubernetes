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

eksctl delete cluster future
