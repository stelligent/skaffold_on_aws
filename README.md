# skaffold_on_aws
stelligent (probably) won't pay for my gcp account, so let's get [Skaffold](https://github.com/GoogleContainerTools/skaffold) running on AWS!

# assumptions
* [you've got the aws cli install and configured](https://docs.aws.amazon.com/cli/latest/userguide/installing.html)
* [you've got docker installed and configured](https://docs.docker.com/install/)

If you'd rather do this on an EC2 instance instead of your laptop, I got you fam:

    aws cloudformation create-stack \
      --stack-name workstation-$(date +%Y%m%d%H%M%S) \
      --template-body file://provisioning/workstation.yml \
      --capabilities CAPABILITY_IAM \
      --parameters \
        ParameterKey="KeyName",ParameterValue="your-key-pair"

Tho, if you do that, it might be hard to make changes to your code and see them autodeploy.

# things to do
If you wish to create an apple pie from scratch, you must first create the universe. So let's create an EKS cluster, and everything you need for that, starting with the VPC. *KeyName* should be an ec2 keypair you have in the account; *NodeImageId* should be either `ami-08cab282f9979fc7a` for us-west-2, or `ami-0b2ae3c6bda8b5c06` for us-east-1.

    export stack_name=skaffold-on-aws-$(date +%Y%m%d%H%M%S)
    aws cloudformation create-stack \
      --stack-name $stack_name \
      --template-body file://provisioning/eks.yml \
      --capabilities CAPABILITY_IAM \
      --parameters \
        ParameterKey="KeyName",ParameterValue="jonny-labs" \
        ParameterKey="NodeImageId",ParameterValue="ami-0b2ae3c6bda8b5c06"

That stack is gonna take about 15m to come up. In the meantime, we'll need to install a few things for everything to work

## aws-am-authenticator

This is used to get `kubectl` to authenticate to your EKS cluster.

    curl -o aws-iam-authenticator https://amazon-eks.s3-us-west-2.amazonaws.com/1.10.3/2018-07-26/bin/linux/amd64/aws-iam-authenticator
    chmod +x ./aws-iam-authenticator
    mkdir -p $HOME/bin # ...just in case
    cp ./aws-iam-authenticator $HOME/bin/aws-iam-authenticator && export PATH=$HOME/bin:$PATH

Confirm it is installed correctly with this command:

    aws-iam-authenticator help

## kubectl

This is used to interact with your EKS cluster.

    sudo apt-get update && sudo apt-get install -y apt-transport-https
    curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
    sudo touch /etc/apt/sources.list.d/kubernetes.list 
    echo "deb http://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee -a /etc/apt/sources.list.d/kubernetes.list
    sudo apt-get update
    sudo apt-get install -y kubectl

Confirm it's installed correctly with this command:

    kubectl version --short --client

Now we'll need to create a `kubeconfig` file. 

    cluster_name=$(aws cloudformation describe-stack-resources --stack-name $stack_name --query StackResources[?ResourceType==\'AWS::EKS::Cluster\'].PhysicalResourceId --output text)
    endpoint_url=$(aws eks describe-cluster --name $cluster_name  --query cluster.endpoint --output text)
    ca_cert=$(aws eks describe-cluster --name $cluster_name  --query cluster.certificateAuthority.data --output text)
    mkdir -p ~/.kube
    cat << KBCFG > ~/.kube/config-${cluster_name}
    apiVersion: v1
    clusters:
    - cluster:
        server: $endpoint_url
        certificate-authority-data: $ca_cert
      name: kubernetes
    contexts:
    - context:
        cluster: kubernetes
        user: aws
      name: aws
    current-context: aws
    kind: Config
    preferences: {}
    users:
    - name: aws
      user:
        exec:
          apiVersion: client.authentication.k8s.io/v1alpha1
          command: aws-iam-authenticator
          args:
            - "token"
            - "-i"
            - "$cluster_name"
    KBCFG
    export KUBECONFIG=$KUBECONFIG:~/.kube/config-${cluster_name}

You can confirm everything is working with this command (assuming the cloudformation stack above has completed!:

    kubectl get svc

If that doesn't give you back some cluster info, running it in verbose mode should help you figure out why:

    kubectl get svc --v=10

With `kubectl` installed, we need to use it to add our nodes to the EKS Cluster.

    node_group_role_arn=$(aws cloudformation describe-stacks --stack-name $stack_name --query Stacks[*].Outputs[?OutputKey==\'NodeGroupRoleArn\'].OutputValue --output text)
    cat << AUTHCFG > aws-auth-cm.yaml
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: aws-auth
      namespace: kube-system
    data:
      mapRoles: |
        - rolearn: $node_group_role_arn
          username: system:node:{{EC2PrivateDNSName}}
          groups:
            - system:bootstrappers
            - system:nodes
    AUTHCFG
    kubectl apply -f aws-auth-cm.yaml

It will take a few moments for the nodes to be added. You can check their progress with this command:

     kubectl get nodes

 You're looking for the `STATUS` field to be `Ready` for all nodes.

## skaffold

And we'll actually need to install Skaffold

    curl -Lo skaffold https://storage.googleapis.com/skaffold/releases/latest/skaffold-linux-amd64
    chmod +x skaffold
    sudo mv skaffold $HOME/bin

And now configure it to work with EKS/ECR:

    # login to ECR
    $(aws ecr get-login --no-include-email)

    # lookup all relevant information
    export ecr=$(aws cloudformation describe-stack-resources --stack-name $stack_name --query StackResources[?ResourceType==\'AWS::ECR::Repository\'].PhysicalResourceId --output text)
    export repository_uri=$(aws ecr describe-repositories --repository-names $ecr --query repositories[*].repositoryUri --output text)
    export region=us-east-1
    export secret=${region}-ecr-registry
    export password=$(aws ecr get-authorization-token --output text --query authorizationData[].authorizationToken | base64 -d | cut -d: -f2)
    email=user@host.com

    # configure kubes to log into ECR
    kubectl delete secret --ignore-not-found $secret
    kubectl create secret docker-registry $secret \
     --docker-server=https://${repository_uri} \
     --docker-username=AWS \
     --docker-password="${password}" \
     --docker-email="${email}"

    # set up skaffold and k8s configuration files
    cat << SKFLDCFG > skaffold.yaml
    apiVersion: skaffold/v1alpha2
    kind: Config
    build:
      artifacts:
      - imageName: ${repository_uri}
    deploy:
      kubectl:
        manifests:
          - k8s-*
    SKFLDCFG

    cat << PODCFG > k8s-pod.yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: getting-started
    spec:
      hostNetwork: true
      containers:
      - name: getting-started
        image: ${repository_uri}
        ports:
        - containerPort: 80
      imagePullSecrets:
      - name: us-east-1-ecr-registry  
    PODCFG

Alright, at this point you should be able to run skaffold and have it deploy

    skaffold dev

And then in another window if you run

    kubectl describe pods

You should see all the information about your running container! If you make any changes in that directory, Skaffold will detect it, rebuild the container, and push it to your Kubernetes cluster.



