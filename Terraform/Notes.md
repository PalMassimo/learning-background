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




