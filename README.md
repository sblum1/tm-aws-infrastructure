# tm-aws-infrastructure
This repository is meant to serve as a template for creating the components necessary to host Vault on AWS via Terraform, which includes a VPC, 2 public subnets, 2 private subnets, 2 NAT gateways, an RDS Postgres instance, an EKS cluster with two nodes, and an internet gateway across 2 availability zones. A diagram illustrating this can be found below:

![image](https://user-images.githubusercontent.com/119435702/211435286-1959b993-4f4a-41ae-81fc-2fd336a25b68.jpeg)

This does not take into account provisioning an MSK cluster (for Kafka implementation), but steps can be found on the attached document 'Setting up a Kafka Cluster'.

## Sources

This repository follows the architecture used by Mark Maglana in his series on designing a 3-tier app on AWS EKS. His repository can be found [here](https://github.com/relaxdiego/system-design), as well as his videos covering the topic, which can be found [here](https://relaxdiego.com/2021/07/system-design-webapp-on-aws.html). Notes covering the architecture discussed in the videos can be found in the attached document, 'EKS Web App Notes'.

## Prerequisites: 
1. An AWS account 
2. AWS CLI; see the attached document 'AWS CLI Instructions'
3. An IAM user with admin access; instructions can be found [here](https://dev.to/aws-builders/creating-your-first-iam-admin-user-and-user-group-in-your-aws-account-machine-learning-part-1-3cne).
4. The configuration of an IAM user with admin access to the AWS CLI; instructions can be found [here](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-files.html).
5. Terraform CLI; download [here](https://developer.hashicorp.com/terraform/downloads).
6. Kubectl; download [here](https://kubernetes.io/docs/tasks/tools/#kubectl).
7. Eksctl; download [here](https://docs.aws.amazon.com/eks/latest/userguide/eksctl.html).
8. Helm; download [here](https://helm.sh/docs/intro/install/).

## Steps:

#### 1. Initialize Terraform Directory
```
cd <PROJECT-ROOT>

terraform -chdir=terraform init
```

#### 2. Create Environment-Specific tfvars File
```
cp terraform/example.tfvars terraform/terraform.tfvars
```

#### 3. Create and download an EC2 key-pair 
The steps to create an EC2 key-pair are as follows:
1. Navigate to the EC2 page from the AWS console.
2. Click on 'Key Pairs' which can be found on the left-hand side of the page.
3. Create a key pair, with a type of RSA and key file format of .pem
4. Download the key pair, and navigate to file path (alternatively, this can be dragged into an environment within an IDE).
5. Run the following command to extract the public key material:
``` 
ssh-keygen -y -f /path_to_key_pair/my-key-pair.pem
```
If the command fails, run the following command before rerunning the previous:
```
chmod 400 key-pair-name.pem
```
6. Make note of the public key (which will be used in the tfvars file). The key should begin 'ssh-rsa' followed by an alphanumerical sequence.
7. Delete the EC2 key-pair that was created via the console (this avoids any duplicate key name errors).

#### 3. Fill Out Environment Variables
Example IP CIDR's can be found in the example.tfvars file. Note that node instances cannot be ran on the T2 instace class. The public key material from the previous step should be input for the variable 'ec2_key'. 

#### 4. Create the DB Credentials Secret in AWS
```
mkdir -p secrets_dir
cd secrets_dir
secrets_dir=$(pwd)
chmod 0700 $secrets_dir
cd ..

aws_profile=$(grep -E ' *aws_profile *=' terraform/terraform.tfvars | sed -E 's/ *aws_profile *= *"(.*)"/\1/g')
aws_region=$(grep -E ' *aws_region *=' terraform/terraform.tfvars | sed -E 's/ *aws_region *= *"(.*)"/\1/g')
cluster_name=$(grep -E ' *cluster_name *=' terraform/terraform.tfvars | sed -E 's/ *cluster_name *= *"(.*)"/\1/g')
db_creds_secret_name=${cluster_name}-db-creds
db_creds_secret_file=${secrets_dir}/${cluster_name}-db-creds.json

cat > $db_creds_secret_file <<EOF
{
    "db_user": "SU_$(uuidgen | tr -d '-')",
    "db_pass": "$(uuidgen)"
}
EOF
chmod 0600 $db_creds_secret_file

aws secretsmanager create-secret \
  --profile "$aws_profile" \
  --name "$db_creds_secret_name" \
  --description "DB credentials for ${cluster_name}" \
  --secret-string file://$db_creds_secret_file
```

After running these commands, a JSON object with the keys 'ARN', 'Name', and 'VersionID' should appear on the console.

#### 5. Deploy to AWS 
The infrastructure can now be deployed using the following command:
```
terraform -chdir=[name-of-tfvars-file] apply
```
