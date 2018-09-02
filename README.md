# skaffold_on_aws
stelligent (probably) won't pay for my gcp account, so let's get skaffold running on AWS!

# assumptions
* you've got the aws cli configured

# things to do
if you wish to create an applie pie from scratch, you must first create the universe. So let's create a eks cluster, and everything you need for that, starting with the VPC.

    export stack_name=skaffold-on-aws-$(date +%Y%m%d%H%M%S)
    aws cloudformation create-stack \
      --stack-name $stack_name \
      --template-body file://provisioning/eks.yml \
      --capabilities CAPABILITY_IAM \
      --parameters \
        ParameterKey="KeyName",ParameterValue="jonny-labs" \
        ParameterKey="NodeImageId",ParameterValue="ami-0b2ae3c6bda8b5c06" \

* install and configure kubectl
  * kubectl install
  * aws-iam-authenticator install
* install skaffold

