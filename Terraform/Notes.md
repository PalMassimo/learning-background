# Hashicorp Certified Terraform Associate 2023

Terraform is an immutable, declarative, Infrastructure as Code provisioning language based on Hashicorp Configuration Language, or optionally JSON. 

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

Instead of using `terraform refresh`, we can use `terraform apply -refresh-only`.

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
| number  | number value                                           | 200                                    |
|  list   | sequential list of values identifies by their position | ["Rome", "Venice" ]                    |
|  map    | groups of values identified by named labels            | { "city": "Rome", "country": "Italy" } |
|  set    | unordered muliple values with no duplicates            | { "apple", "banana", "mango"}          |

We can use the function `toset(["a", "b"])` to convert a list of values into a set.

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
    count = 5
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
In Terraform, `data` blocks are used to retrieve data from external sources, such as APIs or databases, and make that data available to your Terraform configuration. With data blocks, you can use information from external sources to drive your infrastructure as code, making it more dynamic and flexible.


### Debugging
Terraform has detailed logs which can be enabled by setting the `TF_LOG` environment variable to `TRACE`, `DEBUG`, `INFO`, `WARN` or `ERROR`. The `TRACE` value is the most verbose and it is the default if `TF_LOG` is set to something other than a log level name.

To redirect the output commands to a file, we can set the environment variable `TF_LOG_PATH`, for example

```terraform
export TF_LOG_PATH=terraform.log
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

The **iterator** argument (optional) sets the name of a temporary variable that represents the current element of the complex value. If omitted, the name of the variable deafults to the label of the dynamic block (e.g. "ingress" in the example above)

```terraform
dynamic "ingress" {
    for_each = var.ingress_ports
    iterator = port
    content {
        from_port = port.value
        to_port   = port.value
        protocol    = "TCP"
        cidr_blocks = ["0.0.0.0/0"]
    }
}
```

### Tainted Resources
Can happen that a resource created with terraform is manually modified by a user, so the configuration in the terraform state does not match no more with the current configuration of such resource. To address this problem we can
- import the changes to terraform
- delete and recreate the resource

For the second case we can mark the resource as `tainted`, with the command `terraform taint aws_instance.ec2`. The command will set the resource in the terraform state as tainted adding the property `status="tainted"`, forcing it to be destroyed and recreated on the next apply. 

The command `terraform taint` will not modify the infrastructure, but does modify the state file in order to mark a resource as tainted. Once a resource is marked as tainted, the next plan will show that the resource will be destroyed and recreated and the next apply will implement this change.

Note that tainting a resource may affect resources that depend on the newly tainted resource.

The `terraform taint` command is no longer used, because it is replaced by the `terraform apply -replace` command, that replace in one command the resource specified. 

```bash
terraform apply -replace aws_instance.web
```

### Spalat Expressions
Spalat expressions allow to get a list of all attributes. Suppose having three iam users created with the `count` parameter and defining as much as output variables as the iam users created. Then we can write

```terraform
resource "aws_iam_user" "user" {
    count = 3
    name  = "iam-user-${count.index}"
    path  = "system"
}

output "iam_user_arns" {
    value = aws_iam_user.user[*].arn
}
```

### Terraform Graph
The `terraform graph` command is used to generate a visual representation of either a configuration or execution plan. The output of terraform graph is in the DOT format, which can be easily be converted to an image.

```bash
terraform graph > graph.dot
```

The graphically visualize `graph.dot` we need an external tool, like `graphviz`


### Terraform Plan File
The generated terraform plan can be saved to a specific path. This plan can be used with `terraform apply` to be certain that only the changes shown in this plan are applied.

```bash
terraform plan -out=path/to/saved_plan
```

The file is a binary file, so its objective is to save the current plan in order to apply it in a second moment or to not lose it. To apply it just run `terraform apply saved_plan`

### Terraform Output
The terraform output is used to extract the value of an output variable from the state file

```bash
terraform output output_value_name 
```

Alternatively, we can simply inspect the terraform state. 


### Terraform Settings
The `terraform` configuration block type is used to configure some behaviors of Terraform itself, such as requiring a minimum Terraform version to apply the configuration. Terraform settings are gathered together into terraform blocks.

The `required_version` setting accepts a version contraint string, which specifies which versions of terraform can be used. If the running versions of Terraform does not match the constraints specified, Terraform will through an error and exit without taking any further actions. 

```terraform
terraform {
    required_version = "> 0.12.0"
    required_providers {
        aws = "~>5.0"
    }
}
```

### Challenges with larger infrastructure
When dealing with large infrastructure, we could face the issue related to API limits provided by the provider. To address this issue we can avoid updating the state of each resource with the flag `-refresh=false`, so the commands would be

```bash
terraform plan  -refresh=false
terraform apply -refresh=false
```

Another approach is to apply the command to a specific target using `terraform plan -target=aws_instance.ec2"`.

Both approaches however are discouraged from terraform: the best practise is to separate the configuration in multiple directories.


### Zipmap Function
The `zipmap` function constructs a map from a list of keys and corresponding list of values. The high level syntax is `zipmap(keyslist, valueslist)`

```
> zipmap(["a", "b"], [1, 2])
{
    "a"=1
    "b"=2
}
```

The `zipmap` function can be very useful when define output values

```terraform
output "combined" {
    value = zipmap(aws_iam_user.user[*].name, aws_iam_user.user[*].arn)
}
```

### Comments
To comment terraform code we can use `#` or `//` to comment a single line, for a block we have to use `\* *\` instead.


### For Each parameter
The `for_each` statement makes use of map/set as an index value of the created resource. In such blocks an additional `each` object is available, and it has two attributes

| each object  | description |
| :----------: | -------------------------------------------- |
| each.key     | the map value corresponding to this instance |

### Count and For Each meta arguments
When using `count`, each resource created is assigned to a particular `count.index`, so if this changes, terraform will see it as a different resource. Hence, if the count is based on a list of values and the order of the element changes, terraform will try to recreate all of them. 

In the case of `for_each` instead the meta argument is based on the `each.key`, so this is generally a more reliable approach because the order of the elements does not matter.

In both cases in the state terraform adds in each resource a field `index_key` field where it is a number in case of `count`, instead will be `each.key` if the resource is created with the `for_each` statement.

As a rule of thumb, we use `count` to create resources very similar to each others. In all other cases, generally `for_each` is better.


## Provisioners

Terraform can turn provisiones both at the time of resource creation as well as destruction. There are two types of `provisioners`:
- `local-exec`:  allow to invoke local executable after a resource is created
- `remote-exec`: invokes a script on a remote resource after it is created

### remote exec
We can use remote exec provisioner to run cli commands to an ec2 after is created. Suppose to install on it an NGinx server, we can define the ec2 resource as follows

```terraform
resource "aws_instance" "ec2" {
  ami           = "ami-0c7217cdde317cfec"
  instance_type = "t3.micro"
  key_name      = "ec2-keypair"

  connection {
    type        = "ssh"
    user        = "ec2-user"
    private_key = file("${path.module}/ssh/ec2-keypair.pem")
    host        = self.public_ip
  }

  provisioner "remote-exec" {
    inline = [
      "sudo amazon-linux-extras install -y nginx-1",
      "sudo systemctl start nginx"
    ]
  }
}
```

### local exec
It follows an example of local exec. The command runs after the apply

```terraform
resource "aws_instance" "ec2" {
    ami           = "ami-0c7217cdde317cfec"
    instance_type = "t3.micro"
    
    provisioner "local-exec" {
        command = "echo ${aws_instance.ec2.private_ip} >> ec2_privateip.txt"
    }
}
```

### Provisioner types
There are two primary types of provisioners

| Provisioner Type | description                           |
| :--------------: | ------------------------------------- |
| creation time    | runs **only** during creation         |
| destroy time     | runs before the resource is destroyed |

If a creation-time provisioner fails, the resource is marked as `tainted`. When not specified, the provisioner is of the type creation time. To mark it as destroy time type

```terraform
provisioner "remote-exec" {
    when   = destroy
    inline = [ "sudo yum -y remove nano" ]
```

### Provisioner failure behaviour
By default, provisioners that fail will also cause the terraform apply itself to fail. The `on_failure` setting can be used to change this.

| Values | description|
| :------: | ---------------------------------------------------------- |
| continue | ignore the error and continue with creation or destruction |
| fail     | raise an error and stop applying. If this is a creation provisioner, taint the resource |

An example below

```terraform
resource "aws_instance" "web" {
    # ...

    provisioner "local-exec" {
        command    = "echo the server ip is ${self.private_ip}"
        on_failure = continue
}
```

### Null Resource
The `null_resource` implements the standard resource lifecycle but takes no further action. The `triggers` argument allows specifying an arbitrary set of values that, when changes, will cause the resource to be replaced.


## Terraform Modules

Official terraform providers and modules are owned and maintained by HashiCorp.

### Commands
To download or update the modules we can use `terraform init` or `terraform get`.

### Referencing module outputs
To reference an output variable of a module use the syntax `module.module_name.output_variable_name`

### Terraform Registry
The **Terraform Registry** is a repository of modules written by the terraform community. We can find verified modules that are maintained by various third party vendors.

Verified modules are reviewed by HashiCorp and actively maintained by contributors to stay up to date and compatible with both terraform and their respective providers. Such modules are marked with a blue verification badge and can be written only by a small group of trusted HashiCorp partners. 

It follows an example of a module available in the terraform registry

```terraform
module "ec2-instance" {
    source  = "terraform-aws-modules/ec2-instance/aws"
    version = "2.13.0"
    ...
}
```

When using modules installed from a registry, HashiCorp recommends explicitly constraining the acceptable version numbers to avoid unexpected or unwanted changes. The version argument accepts a version constraint string. Terraform will use the newest installed version of the module that meets the constraint; if no acceptable versions are installed, it will download the newest version that meets the constraint.

A version number that meets every applicable constraint is considered acceptable.

Terraform consults version constraints to determine whether it has acceptable versions of itself, any required provider plugins, and any required modules. For plugins and modules, it will use the newest installed version that meets the applicable constraints.

#### Publish on Terraform Registry
Anyone can publish and share modules on the terraform registry. Published modules support versioning, automatically generate documentation, allow browsing version histories, show examples and readmes, and more. 

A module in order to be published on the terraform registry must follow the following conventions

| Requirement           | description  |
| :-------------------: | ---------------------------------------------- |
| GitHub                | the code must be on a github public repository                           |
| Naming convention     | the module name must follow the syntax `terraform-<provider>-<name>`     |
| repository convention | github repository description is used to populate the module description |
| module structure      | module structure must adhere to the standard module structure            |
| x.y.z release tags    | registry uses tags to identify module versions. Release tag names must be a semantic version, which can optionally be prefixed with a `v` (e.g. `v1.0.4`, `0.9.2`)

The module structure can be very simple

```bash
$ tree minimal-module/
.
|-- README.md
|-- main.tf
|-- variables.tf
|-- outputs.tf
```

or more structured

```bash
$ tree complex-module/
.
|-- README.md
|-- main.tf
|-- variables.tf
|-- output.tf
|-- ...
|-- modules/
|   |-- nestedA/
|   |   |-- README.md
|   |   |-- variables.tf
|   |   |-- main.tf
|   |   |-- outputs.tf
|   |-- nestedB/
|   |   |-- README.md
|   |   |-- variables.tf
|   |   |-- main.tf
|   |   |-- outputs.tf
|--examples/
|   |-- exampleA/
|   |   |-- main.tf
|   |-- exampleB/
|       |-- main.tf
```

### Terraform Private Registry
Terraform Cloud allows users to publish and mantain custom modules within their organization, providing a secure and controlled environment for sharing infrastructure configurations.

The source strings for private registry modules have source strings as public registry, but with an added hostname prefix `hostname/namespace/name/provider`

### Terraform Workspaces
Terraform allows to have multiple workspaces, with each of the workspace we can have different set of environment variable associated. The command to work with workspaces is `terraform workspace`. To show all possible commands associated, run `terraform workspace -h`

```bash
$ terraform workspace list       # list all workspaces
$ terraform workspace new dev    # creates dev workspace
$ terraform workspace show       # show the current workspace
$ terraform workspace select dev # change workspace
$ terraform workspace delete dev # delete the dev workspace
```

To dinamically use a variable value based on the workspace, see the example below

``` terraform
resource "aws_instance" "ec2" {
    ami          = "ami-8932nkldsf28"
    instance_type = lookup(var.instance_type, terraform.workspace)
}

variable "instance_type" {
    type = "map"

    default = {
        default = "t3.nano"
        dev     = "t3.micro"
        prod    = "t3.large"
    }
}
```


When using workspaces (after the apply) terraform will create the `terraform.tfstate.d` directory, in which will find one directory for every workspace.

```
.
|--terraform.tfstate.d/
|  |--dev/
|  |  |-- terraform.tfstate
|  |--test/
|  |  |-- terraform.tfstate
|  |--prod/
|  |  |-- terraform.tfstate
```

The `terraform.tfstate` for the `default` workspace is always in the root folder of the terraform project.


## Remote State Managment

### Sensitive data
Terraform when dealing with sensitive data will mark them as `<sensitive value>` when output them on the console, but these values will be saved in the `terraform.tfstate`.
For this security reason the `terraform.tfstate` must not be committed if working in a team, but should be saved in a remote configuration.

### Supported Module Sources
When defining a module, we have to specify the `source` argument, which tells terraform to find the source code.
The source can be in
- local path
- terraform registry
- GitHub
- Bitbucket
- Generic git, Mercurial repository
- HTTP Url
- S3 bucket
- GCS bucket

A local path must begin with either `.` or `..` to indicate that a local path is intended. 

Arbitrary git repositories can be used by prefixing the address with the special `git::` prefix. After it, any valid git URL can be specified to select one of the protocols supported by Git. 

```terraform
module "vpc" {
    source = "git::https://example.com/vpc.git
}

module "storage" {
    source = "git::ssh://username@example.com/storage.git"
}
```

By default, terraform will clone and use the default branch (referenced by HEAD) in the selected repository. We can override this using the `ref` argument

```terraform
module "vpc" {
    source = "git::https://example.com/vpc.git?ref=v1.2.0"
}
```

the value of `ref` argument can be any reference that would be accepted by the git checkout command, including branch and tag names.

### Terraform and .gitignore
Depending on the environments, it is recommended to avoid committing certain files to git

| Files to ignore     | description                                              |
| :-----------------: | -------------------------------------------------------- |
| `.terraform`        | cache directory, recreated with `terraform init`         |
| `terraform.tfvars`  | likely contains sensitive data like passwords            |
| `terraform.tfstate` | should be stored not in git repo but in the remote state |
| `crash.log`         | if terraform crashes, the logs are stored to this file   |

### Terraform Backend
Backends primarily determine where terraform stores its state. By default, terraform implicitly uses a backend called `local` to store state in a local file on disk.

```terraform
terraform {
  backend "local" {
    path = "relative/path/to/terraform.tfstate"
  }
}
```

To make a terraform project able to be handled by a team, we have to store the `terraform.tfstate` in a central backend, instead the code on a central git repository.
Terraform supports multiple backends that allows remote service related operations. Some of the most popular are
- s3
- consul
- azurerm
- kubernetes
- http
- etcd

Accessing state in a remote service generally requires some kind of access credentials. Some backends act like plain "remote disks" for state files, others instead support
locking the state while operations are being performed, which helps prevent conflicts and inconsistencies. 

Using `s3` bucket as a backend

```terraform
terraform {
    backend "s3" {
        bucket     = "mybucket"
        key        = "network/terraform.tfstate"
        region     = "us-east-1"
        access_key = "..."    # optional
        secret_key = "..."    # optional
     }
}
```

It's a common practice to divide the project in multiple terraform projects, where each `terraform.tfstate` is in a different path (e.g. in this configuration was `network/terraform.tfstate`).


### State Locking
Whenever a write operation is performed, terraform would lock the state file, in order to avoid concurrent writes. In particular, when using a local `backend` it creates a temporary file named 
`.terraform.tfstate.lock.info`, a json file with lock information (e.g. lock id).

State locking happens automatically on all operations that could write state. We won't see any message about it because all of this happens behind the scenes. If state locking fails, terraform stops.
Note that not all the backends support locking: the documentation for each backend includes details on whether it supports locking or not.

Terraform has a `force-unlock` command to manually unlock the state if unlocking failed. Use it carefully.

```bash
$ terraform force-unlock <lock-id>
```

### State Locking in S3
By default, s3 does not support **state locking** functionality: we need to use `DynamoDB` table, in order to store into it the state lock.
The table must have a partiotion key named `LockID` with type of `String`. The backend configuration will be the following

```terraform
terraform {
    backend "s3" {
        bucket     = "mybucket"
        key        = "network/terraform.tfstate"
        region     = "us-east-1"

        dynamodb_table = "terraform-state-locking"
     }
}
```

If the lock acquisition fails, we'll get an error message like this

```bash
$ terraform plan
Error message: ConditionalCheckFailedException: the conditional request failed
Lock Info:
    ID:        <uuid-string>
    Path:      s3_bucket/network/terraform.tfstate
    Operation: OperationTypeApply
    Who:       DESKTOP-LE83JS\Zeal Vora@DESKTOP-LE83JS
    Version:   1.1.9
    Created:   <timestamp>
    Info: 
```

These information about the lock is the value associated to the LockID key. Hence, in the dynamodb table will have an entry like this

| LockID                                |  Info                                                 |
| :-----------------------------------: | ----------------------------------------------------- |
| s3_bucket/network/terraform.tfstate   | {"ID": "...", "Path": "...", "Operation": "...", ...} |

This is true when a write operation is running. At the end of the operation terraform release the lock and the value associated to the value of the `LockID` will be a digest.

| LockID                                |  Digest                                               |
| :-----------------------------------: | ----------------------------------------------------- |
| s3_bucket/network/terraform.tfstate   | <alphanumeric_string> |

### Terraform State Managment
To manage the terraform state efficiently, terraform offers the following commands. An example is `terraform state list`

| State sub command | description                                              |
| :---------------: | -------------------------------------------------------- |
| `list`            | list resources within terraform state                    |
| `mv`              | moves items within terraform state                       |
| `pull`            | manually download and output the state from remote state |
| `push`            | manually upload a local state file to remote state       |
| `rm`              | remove items from the terraform state                    |
| `show`            | show the attributes of a single resource in the state    |

Note that the `terraform state rm aws_instance.ec2` remove the resource from the state but does not delete the resource from the actual infrastructure: simply terraform does not track it anymore. 

### Connecting Remote States
The `terraform_remote_state` data source retrieves the root module output values from some other terraform configuration, using the latest state snapshot from the remote backend.

In other words, `terraform_remote_state` allows us to fetch the output values of a terraform project and use them as input variables for another project without manually fetch them.

```terraform
data "terraform_remote_state" "vpc" {
    backend = "s3"
    # the config block value is the same as the backend configuration
    config = {
        bucket = "terraform-state-bucket"
        key    = "network/terraform.tfstate"
        region = "us-east-1"
    }
}

resource "aws_security_group" "allow_tls" {
    name = "allow-tls-sg"

    ingress {
        from_port   = 443
        to_port     = 443
        protocol    = "TCP"
        cidr_blocks = ["${data.terraform_remote_state.eip.outputs.eip_addr}/32"]
    }
}
```

### Terraform Import
Terraform is able to import existing infrastructure. This allows to take resources created manually to bring them under terraform managment.

The current implementation of terraform import can only import resources into the state, but it does not generate configuration. This means that these files must be created manually, but terraform has announced that in a future release terraform will support this feature. 

Let's assume we want to make terraform managing an ec2 instance created manually. We have to create define the resource manually in a `ec2.tf` file  with a basic configuration

```terraform
resource "aws_instance" "web" {
    ami           = "ami-156d64ds86g"
    instance_type = "t3.micro"
    key_name      = "ec2-keypair"
    tags = {
        Name = "ec2-instance"
    }
}
```

and a `provider` should be defined if it is not. Then we have to run `terraform init` and then run the `terraform import` command

```
$ terraform import aws_instance.ec2_instance <instance-id>
```

This will be populate the `terraform.tfstate` file with the detailed configuration about the ec2 instance imported. Now that the resource is managed by terraform, we can edit or destroy it using the `terraform apply` and `terraform destroy` commands.

Another equivalent method is to import resources with the `import` block 

```terraform
import {
  to = aws_instance.example
  id = "i-abcd1234"
}

resource "aws_instance" "example" {
  name = "hashi"
  # (other resource arguments...)
}

```


## Security Primer

### Deploy on multiple regions and accounts
If we have a terraform project with resources deployed on multiple accounts or regions we have to add on `providers.tf` another `provider` block

```terraform
provider "aws" {
    region = "us-east-1"
}

provider "aws" {
    alias   = "provider-2"
    region  = "ap-south-1"
    profile = "aws-account-2-profile"
}
```

Where the first `provider` will be used on defined resources if not specified otherwise. The `profile` field contains the profiles configured, retrieved running `aws configure list-profiles` command. To use the second provider:

```terraform
# will be used the "default" provider
resource "aws_eip" "first_eip"{
    vpc = "true"
}

resource "aws_eip" "first_eip"{
    vpc = "true"

    provider = "provider-2"
}
```

### HashiCorp Vault
**HashiCorp Vault** allows organizations to securely store secrets like tokens and certificates along with access management. It is able to generate secrets like ec2 key pairs or access keys for an iam user, or credentials for a MySQL database. It can rotate secrets or make it valid for only amount of time. 


Initially Vault can be used only through a CLI, but thanks to Hashicorp is now possible to use Vault through a GUI.

Hashicorp Vault can also be used to encrypt and decrypt any kind of data, generate random data and has other useful features.

### Vault Provider
The Vault provider allows terraform to read from, write to, and configure HashiCorp Vault. 

The following snippet shows how to use the secrets generated by Vault in Terraform

```terraform
provider "vault" {
    address = "http://127.0.0.1:8200
}

data "vault_generic_secret" "db_credentials" {
    path = "secret/db-creds"
}

output "vault_secret" {
    value = data.vault_generic_secret.db_credentials
    sensitive = "true"
}
```

Inspecting the section `output` of the state we will see 

```terraform
"outputs": {
    "vault_secret": {
        "value": "{\"admin\":\"password\"}",
        "type" : "string",
        "sensitive": true
    }
}
```


It's important to notice that secrets won't be encrypted in `terraform.tfstate` file. 


## Terraform Cloud
Terraform Cloud manages terraform runs in a consistent and reliable environment with various features like access controls, private registry for sharing modules, policy controls and others.


### Workspace
Terraform Cloud provides additional features, like cost estimation after a `terraform plan`, store in it the terraform state, see the terraform runs.

We can set one or more workflows, having as source a version control system like GitHub, GitLab, Bitbucket or Azure DevOps. The workflow can run commands like `terraform plan` or `terraform apply`, and the last one can require a manual approve after examining the desired state. 

We can set two types of variables: the terraform variables and the environment variables. One example of environment (or workspace) variables are aws access keys. 

We can have private modules in Terraform Cloud.

We can set a github repository as source from which we can trigger events 

### Workspace Sentinels
We can associate sentinels set policies to workspace. A sentinel policy is a rule that checks if resources that are going to be deployed satisfy such constraints. Rules depends on which provider the resources belong to.

An example of sentinel policy checks if a resource has a tag or not. We can set one of the following to the sentinel policy:
- `hard-mandatory`, cannot overide
- `soft-mandatory`, can override
- `advisory`, logging only

Sentinels do not prevent to manual changes from the console. Sentinels are a paid feature.

### Backend Operations Types
Terraform cloud supports `local` and `remote` backend operation types. The **remote backend** stores terraform state and may be used to run operations in the terraform cloud, instead with the **local backend**
Terraform cloud can also be used with local operations, in which case only state is stored in the terraform cloud backend

When using full remote operations (e.g. `terraform plan`) can be executed in terraform cloud's run environment, with log output streaming to the local terminal. 
The output shown in the local terminal will be the same as the one shown if the command was run in the cloud. 

In the `providers.tf` we have to set the backend as `remote`
```terraform
terraform {
    required_version = "~> 0.12.0"

    backend "remote" {}
}
```

And we have to create `backend.hcl` file

```hcl
workspaces { name = "demo-workspace" }
hostname     = "app.terraform.io"
organization = "my-demo-organization"
```

It is needed to have a token to connect to the terraform cloud: to get it run `terraform login`. It will ask to open the browser to a specific link, copied the token it provides and paste into the terminal. 
Only after logging in we can run `terraform init`. 

Notice that if the workspace is connected to a VCS - Version Control System we cannot run the commands from the terminal because we will have two repositories (the local and the remote),
so two source of truth. Hence, if we want to run the terraform commands form the local terminal is suggested to not use the **version control workflow** but to use the CLI-driven workflow instead. 
Indeed, when creating a workflow from the terraform cloud, we can select the option "version control flow" or "CLI-driven workflow". To use the last option we have to run `terraform login` to get the API token.

#### Remote Backend Configuration - deep dive
The reccomended way to set the remote configuration is 

```terraform
terraform {
    cloud {
        organization = "demo-organization"

        workspaces {
            name = "cli-driven-workspace"
        }
    }
}
``` 

### Understanding Concept of Air Gap
An air gap is a network security measure employed to ensure that a secure computer network is physically isolated from unsecured networks, such as the public internet.

Terraform Enterprise installs using either an online or air gapped method. The first method requires an internet connectivity, the secondo doesn't.

### Merge Request
After approving a merge request that modifies Terraform configurations in a GitHub repository linked to a Terraform Cloud workspace, the default action that can be expected to run automatically is a "speculative plan" operation.

When you merge a pull request or push changes to the main branch (or any branch you have configured as the trigger for the workspace), Terraform Cloud typically triggers a plan operation. During this plan phase, Terraform examines the proposed changes to your infrastructure and displays a list of actions it would take if applied. It's a way to preview the changes before actually making them.

The plan output shows what resources Terraform would create, modify, or delete, which allows you to review and validate the expected changes. After reviewing the plan, you can then manually apply the changes to your infrastructure through the Terraform Cloud workspace.

### Terraform Cloud Agents
Terraform Cloud agents are lightweight programs deployed within your target infrastructure environment. Their primary function is to receive Terraform plans from Terraform Cloud, execute those plans locally, and apply the desired infrastructure changes. This allows you to manage private or on-premises infrastructure with Terraform Cloud without opening your network to public ingress traffic.

## Extra

### Folder Structure

```bash
$ tree working_directory/
.
|-- .terraform/
    |-- providers/
    |-- environment
|-- terraform.tfstate.d/
    |-- development/
        |-- terraform.tfstate
    |-- test/
        |-- terraform.tfstate
|-- terraform.tfstate
|-- terraform.tfvars
|-- main.tf
|-- variables.tf
|-- outputs.tf
|-- other.tf
```

the `environment` file contains just the last workspace used (except the default one). 

### Terraform Migration
When migrating from Terraform Community (Free) to Terraform Cloud, the terraform version of the workspace just created is the same to the one used to perform the migration. This ensures compatibility and consistency with the existing state and configuration.

### Curiosities
Terraform by default provisions 10 resources concurrently during a `terraform apply` command to speed up the provisioning process and reduce the overall time taken.

### Find a place
Providers can be installed using multiple methods, including downloading from a Terraform public or private registry, the official HashiCorp releases page, a local plugins directory, or even from a plugin cache. Terraform cannot, however, install directly from the source code.











