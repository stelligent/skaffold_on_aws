# skaffold_on_aws
stelligent (probably) won't pay for my gcp account, so let's get [Skaffold](https://github.com/GoogleContainerTools/skaffold) running on AWS!

# assumptions
* [you've got the aws cli configured](https://docs.aws.amazon.com/cli/latest/userguide/installing.html)


# things to do
if you wish to create an applie pie from scratch, you must first create the universe. So let's create a eks cluster, and everything you need for that, starting with the VPC.

    export stack_name=skaffold-on-aws-$(date +%Y%m%d%H%M%S)
    aws cloudformation create-stack \
      --stack-name $stack_name \
      --template-body file://provisioning/eks.yml \
      --capabilities CAPABILITY_IAM \
      --parameters \
        ParameterKey="KeyName",ParameterValue="jonny-labs" \
        ParameterKey="NodeImageId",ParameterValue="ami-0b2ae3c6bda8b5c06"

We'll need to install a few things for everything to work

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

You can confirm everything is working with this command:

    kubectl get svc

If that doesn't give you back some cluster info, running it in verbose mode should help you figure out why:

    kubectl get svc --v=10

## skaffold

And we'll actually need to install Skaffold

    curl -Lo skaffold https://storage.googleapis.com/skaffold/releases/latest/skaffold-linux-amd64 && chmod +x skaffold && sudo mv skaffold /usr/local/bin

And now configure it to work with EKS/ECR:

    export ecr=$(aws cloudformation describe-stack-resources --stack-name $stack_name --query StackResources[?ResourceType==\'AWS::ECR::Repository\'].PhysicalResourceId --output text)
    export repository_uri=$(aws ecr describe-repositories --repository-names $ecr --query repositories[*].repositoryUri --output text)

    cat << SKFLDCFG > skaffold.yml
    apiVersion: skaffold/v1alpha2
    kind: Config
    build:
      artifacts:
      - imageName: ${repository_uri}/skaffold-example
    deploy:
      kubectl:
        manifests:
          - k8s-*
    SKFLDCFG








