# Hashicorp Certified Terraform Associate 2023

## Deploy infrastructure

### Provider
Terraform supports a lot of providers like AWS, Azure, Google Cloud Platform and so on. To use those resources we have to download their plugins and define a configuration block. 

From terraform 0.12 version the block was defined as 

```tf
provider "aws" {
    region = "us-east-1"
}
```

However this approach is discouraged: use this block definition instead

```terraform
terraform {
    required_providers {
        digitalocean = {
            source = "digitalocean/digitalocean"
        }
    }
}

provider "digitalocean" {
    token = "your_token"
}
```

The first block definition can be used but it works only for hashicorp official plugins, for all the others the second block definition is mandatory. However, the best practice is to use always the second block definition.

Hence, the aws provider block should look something like this

```terraform 
terraform {
    required_providers {
        aws = {
            source = "hashicorp/aws"
            version = "~> 3.0"
        }
    }
}

provider "aws" {
    region = "us-east-1"
}
```

note that the `source` field reflect the url of the documentation: `https://registry.terraform.io/providers/hashicorp/aws/latest/docs`

To download the provider plugins, run

```terraform
terraform init
```

### Provider Versioning
Provider plugins are released separately from Terraform itself. During Terraform init, if version argument is not specified, the most recent provider will be downloaded during initialization. For production use, you should constrain the acceptable provider versions via configuration, to ensure that new versions with breaking changes will not be automatically installed.

There are multiple ways for specifying the version of a provider

| Version Number Arguments | Description                       |
| :----------------------: | --------------------------------- |
|         >= 1.0           | greater than equal to the version |
|         <= 1.0           | less than equal to the version    |
|         ~> 2.0           | any version in the 2.X range      |
|     >= 2.10, <=2.30      | any version between 2.10 and 2.30 |

The command `terraform init` generates `.terraform.lock.hcl`, that primarly helps to stick to a specific provider version constraint, even we changed the terraform_provider block.


### Terraform State
To know what resources have to be destroyed or created when we run `terraform apply` `terraform destroy` or `terraform plan`, terraform looks to its state file. This file is created by Terraform, in which it stores the state of the infrastructure that is being created from the tf files. This state allows terraform to map real world resource to our existing configuration. The file is called `terraform.tfstate`. 

With `terraform refresh` terraform tries to reconcile the state that Terraform knows (i.e. the one defined in the `terraform.tfstate`) with the current state (i.e. the deployed infrastructure).

## Read, generate and edit configuration

### Variables
To define a variable just create a terraform file and name it as `variables.tf` or `input.tf`, but this is just a convention.

```terraform
variable "instancetype" {}
```

Now, when running the `terraform` commands the cli will ask us to input variable values. However, there are other ways to define variables values:

- environment variables
- command line flags
- from a file
- default values

In case of command line flags

```terraform
terraform plan -var="ec2_instancetype=t3.micro"
```

To avoid input everytime from the cli the input variables values, we can define such values once in a file called `terraform.tfvars`:

```terraform
instancetype="t3.medium"
```

if we have named the file differently, we have to specify it when run the terraform commands, like `terraform plan -var-file=custom.tfvars`

We can assign a default value to a variable. In this case, if the value is not specified, the default one will be assumed

```terraform
variable "instancetype" {
    default = "t3.small"
}
```

The last approach is using the environment variables. In this approach we have to define an environment variable using the following convention:

```bash
export TF_VAR_instancetype=t3.small
```
```cmd
setx TF_VAR_instancetype=t3.small
```

In case of windows, we have to open another cmd instance to see the new environment variable.







