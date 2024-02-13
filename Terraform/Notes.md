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

#### variables type constraints
The type argument in a variable block allows to restrict the type of value that will be accepted as the value for a variable. If no type constraint is set then a value of any type will be accepted.

```terraform
variable "image_id" {
    type = string
}
```

Terraform supports the following data types

| Type    | Description                                            | Example                                |
| :-----: | ------------------------------------------------------ | -------------------------------------- |
| string  | sequence of unicode characters representing some text  | "hello"                                |
|  list   | sequential list of values identifies by their position | ["Rome", "Venice" ]                    |
|  map    | groups of values identified by named labels            | { "city": "Rome", "country": "Italy" } |
| number  | number value                                           | 200

### Local Values
Local values are helpful to avoid repeating the same values or expressions multiple times in a configuration. If overused they can also make a configuration hard to read by future maintainers by hiding the actual values used. Use local values in situations where a single value or result is used in many places and that value is likely to be changed in the future.

Local values can be used for multiple differente use-cases like havinga conditional expression.

```terraform
locals {
    name_prefix = "${var.name != "" ? var.name : var.default}"
}
```

### Terraform Functions
The terraform language includes a number of built-in functions to combine and transform values. The syntax is `function(arg_1, arg_2)`.

There are several terraform functions, here the some of the main ones:
- `file(filename)`
- `lookup(map, key, default_value)`
- `element(list, index)`
- `timestamp()`

Terraform does not support user defined functions.

### Terraform console
In Terraform, terraform console is a command-line tool that allows you to interactively evaluate expressions and query Terraform's state. It provides a simple environment where you can test expressions, functions, and data sources without modifying your Terraform configuration files or running actual deployments.

When you run terraform console, it starts an interactive session where you can input Terraform language expressions and see their evaluated results. This can be helpful for debugging, testing, or exploring Terraform configurations. Just run `terraform console`. It is very useful to get confident to the terraform functions. 

### Count Parameter
The count parameter on resources can simplify configurations and let scale resources by simply incrementing a number.

```terraform
resource "aws_instance" "ec2" {
    cunt = 5
    ami = "ami-08vds056vf8d6"
    instance_type = "t3.small"
    tags = {
        Name = "ec2-instance-${count.index}"
    }
}
```

In this way, the id of the resource is `aws_instance.ec2[index]`. 

In resource blocks where `count` is set, an additional count object is available in expressions, so you can modify the configuration of each instance. This object has one attribute, `count.index`, starting from `0`.

### Conditional Expressions
A conditional expression uses the value of a bool expression to select one of two values. The syntax is

```terraform
condition ? true_val : false_val
```

Together with the `count` parameter, we can conditionally create a resource

```terraform
resource "aws_instance" "ec2" {
    count = var.create_instance ? 1 : 0
    ami = "ami-08vds056vf8d6"
    instance_type = "t3.small"
}
```

### Data Sources
Data sources allow data to be fetched or computed for use elsewhere in Terraform configuration

...

### Debugging
Terraform has detailed logs which can be enabled by setting the `TF_LOG` environment variable to `TRACE`, `DEBUG`, `INFO`, `WARN` or `ERROR`. The `TRACE` value is the most verbose and it is the default if `TF_LOG` is set to something other than a log level name.

To redirect the output commands to a file, we can set the environment variable `TF_LOG_PATH`, for example

```terraform
export TF_LOG=/tmp/terraform.log
```

### Load order and Semantics
Terraform generally loads all the configuration files within the directory specified in alphabetical order. The files loaded must end in either `.tf` or `.tf.json` to specify the format that is in use. This allows to define resources to different files in the same directory to better organize our code. 

### Dynamic Blocks
In many of the the use-cases, there are repeatable nested blocks that needs to be defined. This can lead to a long code and it can be difficult to manage in a longer term. 
Dynamic blocks allows to dynamically construct repeatable nested blocks which is supported inside `resource`, `data`, `provider`, and `provisioner` blocks

```terraform
dynamic "ingress" {
    for_each = var.ingress_ports
    content {
        from_port   = ingress.value
        to_port     = ingress.value
        protocol    = "TCP"
        cidr_blocks = ["0.0.0.0/0"]
    }
}
```













