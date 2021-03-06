# Multi-Regional AWS Redis Deployment
This is an example deployment of Redis to AWS using a Kubernetes cluster.

Obviously, building an entire K8S cluster for a single Redis instance is absurd, but I'm using this as a demonstration of how I might build a larger infrastructue footprint. 

These clusters were built with [kops](https://github.com/kubernetes/kops).  For production use, I would use a configuration management tool like Ansible or Terraform, where I could check configuration into a repository and manage it with proper organizational controls.

# Components of this environment
* [Redis](https://github.com/antirez/redis/) - The in-memory database.  I'm running the vanilla version from Docker Hub, which is built using the [community-maintained Dockerfiles](https://github.com/docker-library/redis).
* [spiped](https://github.com/Tarsnap/spiped) - A simple authenticating, encrypted TCP proxy.  Used to proxy the Redis replication streams because [Redis does not support TLS](https://github.com/antirez/redis/issues/2178). (what a fail...)
* [Kubernetes](https://kubernetes.io/) - v1.8.4 - The orchestration framework of choice.  Spins up our applications as needed, reallocates resources as servers come off- and on-line, provisions storage, etc.  
* [CoreOS Container Linux](https://coreos.com/os/docs/latest/) - The Linux distribution which runs on all Kubernetes servers.   Based on ChromeOS, I chose CoreOS because it offers a best-of-breed, curated installation of [Docker](https://www.docker.com/).  It also provides automatic software updates and is able to coordinate with Kubernetes to shift application loads around the cluster while individual cluster servers are updated and rebooted.
* [Docker](https://www.docker.com/) - Packaging for the Redis and spiped applications.  Chosen because it provides a concise way to describe application distribution and execution from a human-readable text file.  Also provides convenient packaging and quick distribution of software updates.

# Key files in this repo
* [kube/deployment.yaml](https://github.com/chrissnell/redis-demo-deployment/blob/master/kube/deployment.yaml) - The Kubernetes **Deployment** object, which describes what containers comprise the smallest complete unit (called a "**Pod**")of this application and specifies how many replicas of these units are needed.  It also specifies the Docker image(s) from which to run the app, along with network ports on which the application listens, any data volumes needed, and any per-deployment configuration data.
* [kube/persistentvolumeclaim.yaml](https://github.com/chrissnell/redis-demo-deployment/blob/master/kube/persistentvolumeclaim.yaml) - The Kubernetes **PersistentVolumeClaim** object, which desribes any persistent storage that is needed by the Pods.
* [kube/secret.yaml](https://github.com/chrissnell/redis-demo-deployment/blob/master/kube/secret.yaml) - The Kubernetes **Secret** object, which contains base64-encoded copies of any secrets that might need to be provided to the application at runtime.  Also occasionally used to provide configuration data.  I'm using it to provide the encryption key to spiped.
* [kube/service.yaml](https://github.com/chrissnell/redis-demo-deployment/blob/master/kube/service.yaml) - The Kubernetes **Service** object, which describes how network traffic is directed towards the pods of this Deployment.  In this case, I am using **LoadBalancer** objects to automatically provision EC2 load balancers that send traffic towards the pods.

# AWS Architecture
This architecture uses one Kubernetes cluster per AWS region.  For a two-region footprint, two complete Kubernetes clusters are used.  Because EBS volumes are restricted to a single availability zone (AZ), K8S compute nodes should be built such that there is sufficient node redundancy within a single AZ for **applications with persistent data needs**, such as Redis.  For stateless applications (e.g. typical microservice), the K8S compute nodes can be spread across multiple AZs and the pod replica count can be adjusted so that every pod is has replicas across the region's AZs.

Resources used in each region are as follows:

* (1) Virtual Private Cloud (VPC)
* (1) Internet Gateway
* (3) Elastic Load Balancers for: K8S API, Redis, spiped
* (1) Kubernetes master node EC2 instance (t2.medium)
* (3) Kubernetes compute node EC2 instances (t2.large)
* (1) Jumpbox EC2 instance (t2.small)
* (3) Kubernetes compute node EBS volumes (128 GiB)
* (1) Kubernetes master node EBS volume (64 GiB)
* (1) Redis data EBS volume (5 GiB, sized for demo)
* (1) Jumpbox EBS volume (8 GiB)
* (2) etcd EBS volumes (20 GiB)

![AWS Region Footprint](https://chrissnell.com/webflow/aws-vpc.png?2 "AWS Region Footprint")

# Kubernetes Architecture
The most basic unit of an application in Kubernetes is a pod.  By definition, a pod is an ephemeral instance of one or more interdependent applications and is constrained to a single node in the cluster.  When hardware failure or maintenance occurs, Kubernetes can rebuild a failed pod as a brand new pod on another node in the cluster.  When network-attached storage is used (such as AWS EBS), an application can be given a persistent disk volume that is attached to the pod and re-attached to a replacement pod, should the original pod become available.  

For this Redis infrastructure, I have built pods of a pair of Redis and spiped containers.  There is one pod per Kubernetes cluster and one Kubernetes cluster per AWS region.  These pods have an attached EBS volume that holds the Redis data store.  The pods are fronted by Kubernetes service objects (i.e. Elastic Load Balancers) to facilitate traffic ingress.

![Kubernetes Architecture](https://chrissnell.com/webflow/k8s-arch.png?2 "Kubernetes Architecture")

# Redis Architecture
This deployment uses a single instance of Redis per region, with the `us-east-1` region functioning as the master instance.  The instance in `eu-central-1` functions as a slave of the U.S.-based instance.  Standard master-slave replication is used and [spiped](https://github.com/Tarsnap/spiped) is used to encrypt this replication stream as it traverses regions.  

In `us-east-1`, spiped functions in "encrypt" mode, connecting to the Redis instance on localhost and proxying this traffic to a listener that authenticates connecting clients and then provides encrypted connections for their traffic.  In `eu-central-1`, spiped functions in "decrypt" mode, connecting to the spiped instance in `us-east-1`, authenticating, then decrypting the stream and making it available as an unencrypted socket connection on localhost.  The Redis slave in `eu-central-1` connects to this local proxy to initiate its slaving.

# Building this demo
1. Install the [AWS CLI client](https://aws.amazon.com/cli/), if you haven't already.
2. Set your `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` environment variables, as appropriate. 
3. Ensure that your IAM role can attach full-access policies for EC2, Route53, S3, IAM, and VPC services.
4. Install [kops](https://github.com/kubernetes/kops) via your preferred method.
5. Create an IAM group for kops:
```
aws iam create-group --group-name kops
```
6. Attach policies to this new group:
```
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonEC2FullAccess --group-name kops
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonRoute53FullAccess --group-name kops
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess --group-name kops
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/IAMFullAccess --group-name kops
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonVPCFullAccess --group-name kops
```
7. Create a kops IAM user and add them to the group:
```
aws iam create-user --user-name kops
aws iam add-user-to-group --user-name kops --group-name kops
```
8. Create an access key for the user.  Save these keys locally and set them in your shell environment.
```
aws iam create-access-key --user-name kops
```
9. Create a S3 bucket to hold kops's state:
```
aws s3api create-bucket --bucket kops-state-store --region us-east-1
aws s3api put-bucket-versioning --bucket kops-state-store --versioning-configuration Status=Enabled
```
10. Set the required `KOPS_STATE_STORE` environment variable, as appropriate:
```
export KOPS_STATE_STORE=s3://kops-state-store
```
11. Set some convenience variables in your environment as appropriate:
```
export NAME=us-east-2.k8s.local
export BUILD_REGION=us-east-2
export BUILD_ZONE=us-east-2c
```
12. Create the cluster manifest with kops:
```
kops create cluster \
    --ssh-public-key=~/.ssh/id_rsa_webflow.pub \
    --node-count 3 \
    --zones ${BUILD_ZONE} \
    --master-zones ${BUILD_ZONE} \
    --node-size t2.large \
    --master-size t2.medium \
    --topology private \
    --networking kopeio-vxlan \
    ${NAME}
```
13. Edit the cluster and make the following changes: 1. Change authorization to `rbac: {}`.  2.  Set `loadbalancer.type` to `Internal`:
```
kops edit cluster ${NAME}
```
14. Obtain the correct CoreOS Container Linux AMI (**HVM type**) by visiting this page:  https://coreos.com/os/docs/latest/booting-on-ec2.html
15. Obtain the `OwnerID` for that AMI by plugging in the AMI ID in this command and running it:
```
aws ec2 describe-images --region ${BUILD_REGION} --image-id <AMI ID FROM ABOVE> --output table
```
16. Get the ImageID by plugging `OwnerID` from the previous command into this command and running it.  The image ID should look something like this:  `595879546273/CoreOS-stable-1520.9.0-hvm`:
```
aws ec2 describe-images --region=${BUILD_REGION} --owner=<OWNER ID FROM ABOVE>     --filters "Name=virtualization-type,Values=hvm"  "Name=name,Values=CoreOS-stable*" --query 'sort_by(Images,&CreationDate)[-1].{id:ImageLocation}' --output table
```
17. Edit your kops instance group and change the image to what you obtained in the previous step:
```
kops edit ig --name=${NAME} nodes
```
18. Build the cluster from the manifest that you generated earlier:
```
kops update cluster ${NAME} --yes
```
19. Build a Virtual Private Gateway and a Customer Gateway to provide VPN access to your new Kubernetes cluster.
20. Alternatively, you can build a jumpbox using a generic EC2 instance.  Create a /24 subnet within the VPC for your jumpbox and add this subnet to your VPC's route table, giving it access to the K8S master and K8S node subnets.  Create a security group that allows inbound port 22 from anywhere.  Build a small instance in this subnet and security group and secure it appropriately.   Allocate an Elastic IP and associate it with the jumpbox's private IP.  Install [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/) and kops and copy the contents of your `${HOME}/.kube` and `${HOME}/.kops` directories to the jumpbox.  
21. Once you have a working VPN connection to the VPC, or a working jumpbox, it's time to build the Kubernetes objects.  Start by generating a base64-encoded secret for spiped:
```
dd if=/dev/urandom bs=32 count=1 |base64
```
22. Edit `kube/secret.yaml` put the output from the previous step as the value for `key:`.
23. Create the secret and verify:
```
kubectl create -f secret.yaml
kubectl get secrets
```
24. Create your persistent volume claim:
```
kubectl create -f persistentvolumeclaim.yaml
kubectl get pvc
```
25. Create your deployment and verify that the pods were deployed:
```
kubectl create -f deployment.yaml
kubectl get deploy
kubectl get pods
```
26. Create the service that fronts the pods and verify that it was created:
```
kubectl create -f service.yaml
kubectl get svc
```
27. Describe the spiped service to get the ELB name and port for your encrypted tunnel endpoint:
```
kubectl describe svc siped
```

