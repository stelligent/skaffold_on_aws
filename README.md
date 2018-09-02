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

We'll need to install a few things for Skaffold to work

`kubectl`
This is used to interact with your EKS cluster.

    sudo apt-get update && sudo apt-get install -y apt-transport-https
    curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
    sudo touch /etc/apt/sources.list.d/kubernetes.list 
    echo "deb http://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee -a /etc/apt/sources.list.d/kubernetes.list
    sudo apt-get update
    sudo apt-get install -y kubectl

Confirm it's installed correctly with this command:

    kubectl version --short --client

`aws-am-authenticator`

This is used to get `kubectl` to authenticate to your EKS cluster.

    curl -o aws-iam-authenticator https://amazon-eks.s3-us-west-2.amazonaws.com/1.10.3/2018-07-26/bin/linux/amd64/aws-iam-authenticator
    chmod +x ./aws-iam-authenticator
    cp ./aws-iam-authenticator $HOME/bin/aws-iam-authenticator && export PATH=$HOME/bin:$PATH

Confirm it is installed correctly with this command:

    aws-iam-authenticator help

`skaffold`

And we'll actually need to install Skaffold

    curl -Lo skaffold https://storage.googleapis.com/skaffold/releases/latest/skaffold-linux-amd64 && chmod +x skaffold && sudo mv skaffold /usr/local/bin
