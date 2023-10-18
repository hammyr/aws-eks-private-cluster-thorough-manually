# Create a private Kubernetes Cluster on AWS EKS
- taken from YT 'Ajit Inamdar Tech'
	- url: https://youtu.be/M3j0mln3jBo?si=pkdgWQZqXo3aidRd
- here we are Creating EKS Cluster manually through console.



### Create VPC and its required services
```
- aws vpc >> Create vpc >> 
	-Resources to create:  VPC and more
	-Name tag auto-generation:  untick this,   #we named tags in last, see 2nd last one
	-IPv4 CIDR block:  10.0.0.0/16  (default)
	-Number of Availability Zones (AZs):  2
	-Number of public subnets:  2
	-Number of private subnets:  4
	-NAT gateways:  In 1 AZ
	-VPC endpoints:  None
	-vpc tags:
		-vpc:  eks-demo-vpc
		-Subnets:
			-eks-demo-web-public-subnet-1a
			-eks-demo-app-private-subnet-1a
			-eks-demo-db-private-subnet-1a
			-eks-demo-web-public-subnet-1b
			-eks-demo-app-private-subnet-1b
			-eks-demo-db-private-subnet-1b
		-Route tables:
			-eks-demo-web-public-route-table
			-eks-demo-app-private-route-table-1a
			-eks-demo-app-private-route-table-1b
			-eks-demo-db-private-route-table-1a
			-eks-demo-db-private-route-table-1b
		-Network connections:
			-eks-demo-internet-gateway
			-eks-demo-nat-gateway
	-Create vpc >> done.
```



### create Role for `Cluster service role` for `EKS Cluster`
```
- IAM >> Roles >> Create Roles >>
	-Trusted entity type:  aws services
	-Use case:  EKS - Cluster 
		#(Allows access to other AWS service resources that are required to operate clusters managed by EKS.)
	-next >> 
		-policy name: AmazonEKSClusterPolicy
	-next >> 
	-Role name: eks-demo-cluster-role
	-Create role >> done.
```



### Create EKS Cluster
```
- aws eks >> add Cluster >> Create >> 
	-name:  eks-demo-cluster
	-Kubernetes version:  1.25
	-Cluster service role:  select (eks-demo-cluster-role)
	-tags: Name: eks-demo-cluster
	-next >>
	-Networking:
		-VPC:  select 'eks-demo-vpc'
		-subnets:  select
			-eks-demo-app-private-subnet-1a
			-eks-demo-app-private-subnet-1b
		-Security groups: leave as default.
		
		-Cluster endpoint access: private
	-next >>
	-Control plane logging: leave as default
	-next >>
	-Amazon EKS add-ons:  leave as default
	-next >>
	-Configure selected add-ons settings:  leave as default
	-next >>
	-Review and create:  Create >> done.
		#-Noted: Cluster creation process will take arount 10 minutes.
```



### create Role for `Node IAM role` for `node group`
```
- IAM >> Roles >> Create Roles >>
	-Trusted entity type:  aws services
	-usecase:  ec2
	-next >>
	-Permissions policies: search 'eks' and 'container' & select below policies:
		-AmazonEKSWorkerNodePolicy
		-AmazonEC2ContainerRegistryReadOnly			#this is for pull the images from ECR.
	-next >>
	-Role name: eks-demo-node-group-role
	-remaining all:  keep as it is
	-Create role >> done.
```



### Create key pair for node group
```
- name: eks-demo-node-group-key
	-tags: Name: eks-demo-node-group-key
	-Create >> done.
```


### Create Security group for node group
```
	-name: eks-demo-node-group-security-group
	-description: eks-demo-node-group-security-group
	-vpc:  eks-demo-vpc
	-inbound rule: not add any rule as of now.
		#-Noted: for this demo we dont need any access to worker Node.
	-Create >> done.
```



### Create compute capacity: Node groups
```
- eks >> Clusters >> eks-demo-cluster >> compute >> node groups >> add Node groups >>
	-name: eks-demo-node-group
	-node IAM Role: select- 'eks-demo-node-group-role'
	-tags: Name: eks-demo-node-group
	-next >>
	-Node group compute configuration: go with all default option
		-AMI type:  Amazon Linux 2 (AL2_x86_64)
		-Capacity type:  On-Demand
		-Instance types:  t3.medium
		-disk size:  20
	-Node group scaling configuration: go with all default option
		-Desired size:  2
		-Minimum size:  2
		-Maximum size:  2
	-Node group update configuration: default
	-next >>
	-Node group network configuration
		-Subnets: select
			-eks-demo-app-private-subnet-1a
			-eks-demo-app-private-subnet-1b
		-Configure remote access to nodes: enable it
			-EC2 Key Pair:  eks-demo-node-group-key
			-Allow remote access from:  Selected security groups
			-Security groups:  eks-demo-node-group-security-group
	-next >>
	-Review and create >> Create >> done.
		#-Noted: it takes arount 2 minutes, same time as normal Instance creation takes.
	
	-next >> go to ec2 -instance >> see provisioning is started or not >>
```

---8:40.........


### Provision Jump Server
```
- ec2 >> Create one Instance with these details:
	-Name:  eks-demo-jump-server
	-ami:  Amazon-linux-2023
	-Instance type:  t2.micro
	-Key pair:  eks-demo-jump-server-key
		-Create new key with this name:  eks-demo-jump-server-key
	-VPC:  eks-demo-vpc
	-subnet:  eks-demo-web-public-subnet-1a
	-Auto-assign public IP:  enable
	-Security group:  Create new sg
		-name: eks-demo-jump-server-security-group
		-description: eks-demo-jump-server-security-group
		-inbound rule: allow ssh from anywhere.
	-launch Instance >> done.
```



### connect jump-server via ssh and install, configure required tools
- ssh jump-server
- install kubectl tool from aws
	- docs page: https://docs.aws.amazon.com/eks/latest/userguide/install-kubectl.html
- abc 
```
chmod 400 eks-demo-jump-server-key.pem
ssh -i "eks-demo-jump-server-key.pem" ec2-user@ec2-13-214-193-23.ap-southeast-1.compute.amazonaws.com



#-docs page: https://docs.aws.amazon.com/eks/latest/userguide/install-kubectl.html


# install kubectl tool & configure
curl -O https://s3.us-west-2.amazonaws.com/amazon-eks/1.25.14/2023-10-12/bin/linux/amd64/kubectl

# verify binary (optional)
curl -O https://s3.us-west-2.amazonaws.com/amazon-eks/1.25.14/2023-10-12/bin/linux/amd64/kubectl.sha256

# Check the SHA-256 checksum
sha256sum -c kubectl.sha256

# this ssh command give sha256 string
openssl sha1 -sha256 kubectl

# Apply execute permissions to the binary.
chmod +x ./kubectl

# Copy the binary to a folder in your PATH.
mkdir -p $HOME/bin && cp ./kubectl $HOME/bin/kubectl && export PATH=$HOME/bin:$PATH

# (Optional) Add the $HOME/bin path to your shell initialization file so that it is configured when you open a shell.
echo 'export PATH=$HOME/bin:$PATH' >> ~/.bashrc

# After you install kubectl, you can verify its version.
kubectl version --client
WARNING: This version information is deprecated and will be replaced with the output from kubectl version --short.  Use --output=yaml|json to get the full version.
Client Version: version.Info{Major:"1", Minor:"25+", GitVersion:"v1.25.14-eks-fd9b4c5", GitCommit:"4854490b8fa7537a7ba1913c0de10342a333e893", GitTreeState:"clean", BuildDate:"2023-10-04T17:30:24Z", GoVersion:"go1.20.8", Compiler:"gc", Platform:"linux/amd64"}
Kustomize Version: v4.5.7
```

-12:45...

### configure iam credentials in jump-server
```
aws configure
AWS Access Key ID [None]: ***********
AWS Secret Access Key [None]: **************************
Default region name [None]: ap-southeast-1
Default output format [None]: json 
```



### To create your kubeconfig file with the AWS CLI
- To create your kubeconfig file with the AWS CLI
- update config kubectl
- allow 443 port in eks-clsuter Security group from Jump server Security group
- validate cubeconfig......
- try to Access the Cluster from jump Server........
```
# now our AWS configuration is done
# we need to configure access to the eks
# cluster from our jump server for that
# let's search for "update config kubectl"
# open the official documentation link
	# url: https://docs.aws.amazon.com/eks/latest/userguide/create-kubeconfig.html


# To create your kubeconfig file with the AWS CLI
[ec2-user@ip-10-0-7-56 ~]$ aws eks update-kubeconfig --region ap-southeast-1 --name eks-demo-cluster
Added new context arn:aws:eks:ap-southeast-1:214262210418:cluster/eks-demo-cluster to /home/ec2-user/.kube/config


# now let's validate if it has been added as expected we can do that using the command kubectl config view
[ec2-user@ip-10-0-7-56 ~]$ kubectl config view
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: DATA+OMITTED
    server: https://410BF9726BF05ADE08D173BF900AE5C9.gr7.ap-southeast-1.eks.amazonaws.com
  name: arn:aws:eks:ap-southeast-1:214262210418:cluster/eks-demo-cluster
contexts:
- context:
    cluster: arn:aws:eks:ap-southeast-1:214262210418:cluster/eks-demo-cluster
    user: arn:aws:eks:ap-southeast-1:214262210418:cluster/eks-demo-cluster
  name: arn:aws:eks:ap-southeast-1:214262210418:cluster/eks-demo-cluster
current-context: arn:aws:eks:ap-southeast-1:214262210418:cluster/eks-demo-cluster
kind: Config
preferences: {}
users:
- name: arn:aws:eks:ap-southeast-1:214262210418:cluster/eks-demo-cluster
  user:
    exec:
      apiVersion: client.authentication.k8s.io/v1beta1
      args:
      - --region
      - ap-southeast-1
      - eks
      - get-token
      - --cluster-name
      - eks-demo-cluster
      command: aws
      env: null
      interactiveMode: IfAvailable
      provideClusterInfo: false

  
# now let's try to access the cluster using the command kubectl cluster-info
[ec2-user@ip-10-0-7-56 ~]$ kubectl cluster-info
^C

	# Noted: here we see the request is not going
	# through
	# it's because we have added the context
	# to the jump server, configure the iam
	# credentials as well, but our eks cluster
	# has to allow connectivity from
	# the jump server, for that we will have to
	# open 443 on the eks cluster Security
	# Group from jump server Security Group


# allow 443 port in eks-clsuter Security group from Jump server Security group
- go to eks >> clsuter >> eks-demo-cluster >> Networking >> Cluster security group >>
	-add port 443 from 'eks-demo-jump-server-security-group'
		-Description: allow 443 port in eks-clsuter Security group from Jump server Security group
	-save rules >> done.


# now head on to the terminal and rerun the cluster info command again
[ec2-user@ip-10-0-7-56 ~]$ kubectl cluster-info
Kubernetes control plane is running at https://410BF9726BF05ADE08D173BF900AE5C9.gr7.ap-southeast-1.eks.amazonaws.com
CoreDNS is running at https://410BF9726BF05ADE08D173BF900AE5C9.gr7.ap-southeast-1.eks.amazonaws.com/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy


# so finally our cluster is accessible from our jump server, let's try kubectl get nodes command as well
[ec2-user@ip-10-0-7-56 ~]$ kubectl get nodes
NAME                                              STATUS   ROLES    AGE     VERSION
ip-10-0-129-89.ap-southeast-1.compute.internal    Ready    <none>   3h43m   v1.25.13-eks-43840fb
ip-10-0-147-243.ap-southeast-1.compute.internal   Ready    <none>   3h43m   v1.25.13-eks-43840fb


# we are now able to access the cluster,
# however it's not a best practice to
# access it using IAM credentials instead
# it should be done via IAM role if we are
# accessing from an ac2 instance


# when you do a configure list we see list
# of IAM identities configured in the
# server so these are the credentials of
# iam user
[ec2-user@ip-10-0-7-56 ~]$ aws configure list
      Name                    Value             Type    Location
      ----                    -----             ----    --------
   profile                <not set>             None    None
access_key     ****************U3T5 shared-credentials-file
secret_key     ****************nmxu shared-credentials-file
    region           ap-southeast-1      config-file    ~/.aws/config


# and when you do AWS STS get
# caller identity it tells you who is
# running the AWS command again here we
# see my user is being used
[ec2-user@ip-10-0-7-56 ~]$ aws sts get-caller-identity
{
    "UserId": "AIDATDYYF35ZOAEZGUO2L",
    "Account": "214262210418",
    "Arn": "arn:aws:iam::214262210418:user/cloud-admin"
}
```



### so let's configure our ec2 instance to use a iam role instead of an iam user credentials
```
# Create iam role
- aws iam >> role >> create role >>
	-Trusted entity type:  AWS service
	-Use case:  EC2
	-next >>
	-Permissions policies: leave it, see explanation in below:
		# here we are not selecting any policies
		# as we need only kubernetes access to the
		# ec2 user for now
		# and for that configuration no policy is
		# required, in future if you want to access
		# other services or perform some
		# activities in the eks cluster you can
		# provide those permissions
		# but for accessing the kubernetes cluster
		# nothing is required so ignore the
		# policies and click next
	-next >>
	-Role name:  eks-demo-jump-server-role
	-rest:  default
	-Create >> done.


# attach this role into jump-server
	-ec2 instance >> eks-demo-jump-server >> action >> security >> modify iam role >>
		-select: eks-demo-jump-server-role & update iam role >> done.
```




### configure our IAM role in the kubernetes cluster
```

-17:30...

# go back to the jump server terminal and
# check the configure list
# we still see the configured user and get
# caller identity is still showing iam
# user as Ajit but that's okay for now
# I'll show how do we clear that in some
# time 
[ec2-user@ip-10-0-7-56 ~]$ aws configure list
      Name                    Value             Type    Location
      ----                    -----             ----    --------
   profile                <not set>             None    None
access_key     ****************U3T5 shared-credentials-file
secret_key     ****************nmxu shared-credentials-file
    region           ap-southeast-1      config-file    ~/.aws/config
[ec2-user@ip-10-0-7-56 ~]$ aws sts get-caller-identity
{
    "UserId": "AIDATDYYF35ZOAEZGUO2L",
    "Account": "214262210418",
    "Arn": "arn:aws:iam::214262210418:user/cloud-admin"
}




# but before that we have to
# configure our IAM role in the kubernetes
# cluster
# go to the web browser and search for "AWS
# eks edit configmap"
# open the link which says enable IAM
# principle access to your cluster
	#url: https://docs.aws.amazon.com/eks/latest/userguide/add-user-role.html


# we have to make changes to AWS auth
# config map from the cube system
# namespace
# the describe command will give you the
# details about this particular config map
[ec2-user@ip-10-0-7-56 ~]$ kubectl describe configmap aws-auth  -n kube-system
Name:         aws-auth
Namespace:    kube-system
Labels:       <none>
Annotations:  <none>

Data
====
mapRoles:
----
- groups:
  - system:bootstrappers
  - system:nodes
  rolearn: arn:aws:iam::214262210418:role/eks-demo-node-group-role
  username: system:node:{{EC2PrivateDNSName}}


BinaryData
====

Events:  <none>



# but before we make any changes let's take a backup of the config map
[ec2-user@ip-10-0-7-56 ~]$ kubectl get cm aws-auth -n kube-system -o yaml > aws-auth.yaml



# now update/edit the configmap
[ec2-user@ip-10-0-7-56 ~]$ kubectl edit cm aws-auth -n kube-system
configmap/aws-auth edited


# lets see validate whether configmap is updated
[ec2-user@ip-10-0-7-56 ~]$ kubectl describe configmap aws-auth  -n kube-system
Name:         aws-auth
Namespace:    kube-system
Labels:       <none>
Annotations:  <none>

Data
====
mapRoles:
----
- groups:
  - system:bootstrappers
  - system:nodes
  rolearn: arn:aws:iam::214262210418:role/eks-demo-node-group-role
  username: system:node:{{EC2PrivateDNSName}}
- groups:
  - system:masters
  rolearn: arn:aws:iam::214262210418:role/eks-demo-jump-server-role
  username: eks-demo-jump-server-role


BinaryData
====

Events:  <none>
```




### here in jump-server iam-user credentials & iam role both are Configure
```
[ec2-user@ip-10-0-7-56 ~]$ aws configure list
      Name                    Value             Type    Location
      ----                    -----             ----    --------
   profile                <not set>             None    None
access_key     ****************U3T5 shared-credentials-file
secret_key     ****************nmxu shared-credentials-file
    region           ap-southeast-1      config-file    ~/.aws/config


[ec2-user@ip-10-0-7-56 ~]$ aws sts get-caller-identity
{
    "UserId": "AIDATDYYF35ZOAEZGUO2L",
    "Account": "214262210418",
    "Arn": "arn:aws:iam::214262210418:user/cloud-admin"
}



# you already know that even with iam
# role configured just now, but all the request to AWS
# are done via IAM user credentials only, so to reset that (iam-user-credentials)
[ec2-user@ip-10-0-7-56 ~]$ > ~/.aws/config
[ec2-user@ip-10-0-7-56 ~]$ > ~/.aws/credentials



# now rerun the AWS STS get caller
# identity command you will notice that it
# is picking up the IAM role and not the IAM
# user now
[ec2-user@ip-10-0-7-56 ~]$ aws sts get-caller-identity
{
    "UserId": "AROATDYYF35ZEHZAIJTXB:i-03655a19c5c8ffe7f",
    "Account": "214262210418",
    "Arn": "arn:aws:sts::214262210418:assumed-role/eks-demo-jump-server-role/i-03655a19c5c8ffe7f"
}


# when you do an AWS configure list you will see Dynamic credentials generated by the IAM role
[ec2-user@ip-10-0-7-56 ~]$ aws configure list
      Name                    Value             Type    Location
      ----                    -----             ----    --------
   profile                <not set>             None    None
access_key     ****************LLW2         iam-role
secret_key     ****************1PKJ         iam-role
    region           ap-southeast-1             imds



# let's do a 'kubectl get nodes' to check if our cluster is accessible via iam role
[ec2-user@ip-10-0-7-56 ~]$ kubectl get nodes
NAME                                              STATUS   ROLES    AGE     VERSION
ip-10-0-129-89.ap-southeast-1.compute.internal    Ready    <none>   4h50m   v1.25.13-eks-43840fb
ip-10-0-147-243.ap-southeast-1.compute.internal   Ready    <none>   4h50m   v1.25.13-eks-43840fb
```


####### ==========> End <================== ######


