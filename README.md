# Automating Infrastructure Creation With IAC (Terraform)


- Create an IAM user, name it terraform (ensure that the user has only programatic access to your AWS account) and grant this user AdministratorAccess permissions.

![](./Images/Screenshot%202024-04-30%20094511.png)

![](.//Images/Screenshot%202024-04-30%20094530.png)

- Copy the secret access key and access key ID. Save them in a notepad temporarily.
- Configurepython --version

```
pip install boto3

pip install boto3[crt]
```

![](./images/boto3.png)

![](./images/boto3_2.png)


If you have the AWS CLI installed, then you can use the aws configure command to configure your credentials file:

```
aws configure
```
Alternatively, you can create the credentials file yourself. By default, its location is ~/.aws/credentials. At a minimum, the credentials file should specify the access key and secret access key. In this example, the key and secret key for the account are specified in the default profile:

```
aws_access_key_id = YOUR_ACCESS_KEY
aws_secret_access_key = YOUR_SECRET_KEY
```
![](./images//AWS_configure.png)

![](./Images/Bucket.png)
You may also want to add a default region to the AWS configuration file, which is located by default at ~/.aws/config:

```
region=us-east-1
```
Alternatively, you can pass a region_name when creating clients and resources.


# VPC | SUBNETS | SECURITY GROUPS

Let us create a directory structure

Open your Visual Studio Code and:

- Create a folder called `PBL`
- Create a file in the folder, name it `main.tf`
- 
![](./Images/apply_001.png)

![](./images/001.png)


# Provider and VPC resource section

- Set up Terraform CLI as per this instruction.

- Add AWS as a provider, and a resource to create a VPC in the main.tf file.

- Provider block informs Terraform that we intend to build infrastructure within AWS.

- Resource block will create a VPC.

Note: You can change the configuration above to create your VPC in other region that is closer to you. The same applies to all configuration snippets that will follow.

- The next thing we need to do, is to download necessary plugins for Terraform to work. These plugins are used by providers and provisioners. At this stage, we only have provider in our main.tf file. So, Terraform will just download plugin for AWS provider.



- Notice that a new directory has been created: .terraform.... This is where Terraform keeps plugins. Generally, it is safe to delete this folder. It just means that you must execute terraform init again, to download them.

- Moving on, let us create the only resource we just defined. aws_vpc. But before we do that, we should check to see what terraform intends to create before we tell it to go ahead and create it.

- Run terraform plan

- Then, if you are happy with changes planned, execute terraform apply

![](./Images/apply_001.png)



According to our architectural design, we require 6 subnets:

- 2 public
- 2 private for webservers
- 2 private for data layer
- 
Let us create the first 2 public subnets.

Add below configuration to the main.tf file:

```
# Create public subnets1
   resource "aws_subnet" "public_subnet_1" {
    vpc_id                  = aws_vpc.main.id
    cidr_block              = "172.16.1.0/24"
    availability_zone       = "us-east-1a"
    map_public_ip_on_launch = true
}
# Create public subnets2

resource "aws_subnet" "public_subnet_2" {
    vpc_id                  = aws_vpc.main.id
    cidr_block              = "172.16.2.0/24"
    availability_zone       = "us-east-1b"
    map_public_ip_on_launch = true
}
```

We are creating 2 subnets, therefore declaring 2 resource blocks – one for each of the subnets. We are using the vpc_id argument to interpolate the value of the VPC id by setting it to aws_vpc.main.id. This way, Terraform knows inside which VPC to create the subnet. Run terraform plan and terraform apply

Observations:

Hard coded values: Remember our best practice hint from the beginning? Both the availability_zone and cidr_block arguments are hard coded. We should always endeavour to make our work dynamic. Multiple Resource Blocks: Notice that we have declared multiple resource blocks for each subnet in the code. This is bad coding practice. We need to create a single resource block that can dynamically create resources without specifying multiple blocks. Now let us improve our code by refactoring it.

First, destroy the current infrastructure. Since we are still in development, this is totally fine. Otherwise, DO NOT DESTROY an infrastructure that has been deployed to production.

To destroy whatever has been created run terraform destroy command, and type yes after evaluating the plan.


![](./Images/destroy.png)


# FIXING THE PROBLEMS BY CODE REFACTORING

Starting with the provider block, declare a variable named region, give it a default value, and update the provider section by referring to the declared variable.

```
    variable "region" {
        default = "us-east-1"
    }

    provider "aws" {
        region = var.region
    }
Do the same to cidr value in the vpc block, and all the other arguments.
    variable "region" {
        default = "us-east-1"
    }

    variable "vpc_cidr" {
        default = "172.16.0.0/16"
    }

    variable "enable_dns_support" {
        default = "true"
    }

    variable "enable_dns_hostnames" {
        default ="true" 
    }

    variable "enable_classiclink" {
        default = "false"
    }

    variable "enable_classiclink_dns_support" {
        default = "false"
    }

    provider "aws" {
    region = var.region
    }

    # Create VPC
    resource "aws_vpc" "main" {
    cidr_block                     = var.vpc_cidr
    enable_dns_support             = var.enable_dns_support 
    enable_dns_hostnames           = var.enable_dns_support
    enable_classiclink             = var.enable_classiclink
    enable_classiclink_dns_support = var.enable_classiclink

    }
```

We are just going to introduce some interesting concepts. Loops & Data sources

Terraform has a functionality that allows us to pull data which exposes information to us. Let us fetch Availability zones from AWS, and replace the hard coded value in the subnet’s availability_zone section.

```
# Get list of availability zones
data "aws_availability_zones" "available" {
  state = "available"
}
```
To make use of this new data resource, we will need to introduce a count argument in the subnet block: Something like this.

```
# Create public subnet1
resource "aws_subnet" "public" { 
    count                   = 2
    vpc_id                  = aws_vpc.main.id
    cidr_block              = "172.16.1.0/24"
    map_public_ip_on_launch = true
    availability_zone       = data.aws_availability_zones.available.names[count.index]

}
```

But we still have a problem. If we run Terraform with this configuration, it may succeed for the first time, but by the time it goes into the second loop, it will fail because we still have cidr_block hard coded. The same cidr_block cannot be created twice within the same VPC. So, we have a little more work to do.

We will introduce a function cidrsubnet() to make this happen. It accepts 3 parameters.

```
  # Create public subnet1
  resource "aws_subnet" "public" { 
      count                   = 2
      vpc_id                  = aws_vpc.main.id
      cidr_block              = cidrsubnet(var.vpc_cidr, 4 , count.index)
      map_public_ip_on_launch = true
      availability_zone       = data.aws_availability_zones.available.names[count.index]

  }
```

We can introuduce length() function, which basically determines the length of a given list, map, or string.

Since data.aws_availability_zones.available.names returns a list like ["eu-central-1a", "eu-central-1b", "eu-central-1c"] we can pass it into a lenght function and get number of the AZs.

Now we can simply update the public subnet block like this

# Create public subnet1

```
resource "aws_subnet" "public" { 
    count                   = length(data.aws_availability_zones.available.names)
    vpc_id                  = aws_vpc.main.id
    cidr_block              = cidrsubnet(var.vpc_cidr, 4 , count.index)
    map_public_ip_on_launch = true
    availability_zone       = data.aws_availability_zones.available.names[count.index]

}
```

Observations:

What we have now, is sufficient to create the subnet resource required. But if you observe, it is not satisfying our business requirement of just 2 subnets. The length function will return number 3 to the count argument, but what we actually need is 2. Now, let us fix this.

Declare a variable to store the desired number of public subnets, and set the default value


```
variable "preferred_number_of_public_subnets" {
  default = 2
}
```

Next, update the count argument with a condition. Terraform needs to check first if there is a desired number of subnets. Otherwise, use the data returned by the lenght function. See how that is presented below.



# Create public subnets


```
# Create public subnets
resource "aws_subnet" "public" {
  count  = var.preferred_number_of_public_subnets == null ? length(data.aws_availability_zones.available.names) : var.preferred_number_of_public_subnets   
  vpc_id = aws_vpc.main.id
  cidr_block              = cidrsubnet(var.vpc_cidr, 4 , count.index)
  map_public_ip_on_launch = true
  availability_zone       = data.aws_availability_zones.available.names[count.index]

}
```

Now lets break it down:

- The first part var.preferred_number_of_public_subnets == null checks if the value of the variable is set to null or has some value defined.
- The second part ? and length(data.aws_availability_zones.available.names) means, if the first part is true, then use this. In other words, if preferred number of public subnets is null (Or not known) then set the value to the data returned by lenght function.
- The third part : and var.preferred_number_of_public_subnets means, if the first condition is false, i.e preferred number of public subnets is not null then set the value to whatever is definied in var.preferred_number_of_public_subnets
  
Now the entire configuration should now look like this

```
# Get list of availability zones
data "aws_availability_zones" "available" {
state = "available"
}

variable "region" {
      default = "us-east-1"
}

variable "vpc_cidr" {
    default = "172.16.0.0/16"
}

variable "enable_dns_support" {
    default = "true"
}

variable "enable_dns_hostnames" {
    default ="true" 
}

variable "enable_classiclink" {
    default = "false"
}

variable "enable_classiclink_dns_support" {
    default = "false"
}

  variable "preferred_number_of_public_subnets" {
      default = 2
}

provider "aws" {
  region = var.region
}

# Create VPC
resource "aws_vpc" "main" {
  cidr_block                     = var.vpc_cidr
  enable_dns_support             = var.enable_dns_support 
  enable_dns_hostnames           = var.enable_dns_support
  enable_classiclink             = var.enable_classiclink
  enable_classiclink_dns_support = var.enable_classiclink

}

# Create public subnets
resource "aws_subnet" "public" {
  count  = var.preferred_number_of_public_subnets == null ? length(data.aws_availability_zones.available.names) : var.preferred_number_of_public_subnets   
  vpc_id = aws_vpc.main.id
  cidr_block              = cidrsubnet(var.vpc_cidr, 4 , count.index)
  map_public_ip_on_launch = true
  availability_zone       = data.aws_availability_zones.available.names[count.index]

}
```


# Introducing variables.tf & terraform.tfvars

We will put all variable declarations in a separate file

And provide non default values to each of them

- Create a new file and name it variables.tf
- Copy all the variable declarations into the new file.
- Create another file, name it terraform.tfvars
- Set values for each of the variables.

## Maint.tf

```bash

 
provider "aws" {
    region = var.region

}
# Create VPC

resource "aws_vpc" "my_vpc" {
    cidr_block           = var.cidr
    enable_dns_support   = var.enable_dns_support
    enable_dns_hostnames = var.enable_dns_hostname

    # Remove the line below
    # enable_classiclink   = var.enable_classiclink

    # Remove the line below
    # enable_classiclink_dns_support = var.enable_classiclink_dns_support

}

# Create public subnets

resource "aws_subnet" "public_subnet" {
    count             = length(var.availability_zones)
    vpc_id            = aws_vpc.my_vpc.id
    cidr_block        = cidrsubnet(var.cidr, 8, count.index)
    map_public_ip_on_launch = true
    availability_zone = element(var.availability_zones, count.index)

}
```

## variables.tf

```bash
# Define variables

variable "region" {
    description = "AWS region"
    default     = "us-east-1"

}

variable "cidr" {
    description = "VPC CIDR block"
    default     = "172.16.0.0/16"

}


variable "availability_zones" {
    description = "List of availability zones"
    default     = ["us-east-1a", "us-east-1b"]

}


variable "enable_dns_support" {
    description = "Enable DNS support"
    default     = true

}


variable "enable_dns_hostname" {
    description = "Enable DNS hostname"
    default     = true

}

variable "enable_classiclink" {
    description = "Enable ClassicLink"
    default     = true

}

 

variable "enable_classiclink_dns_support" {
    description = "Enable ClassicLink DNS support"
    default     = true

}

variable "preferred_number_of_public_subnets" {
    description = "Preferred number of public subnets"
    default     = 2
  
}
```

## terraform.tfvars

```bash
region = "us-east-1"

cidr = "172.16.0.0/16" 

enable_dns_support = "true" 

enable_dns_hostname = "true"  

enable_classiclink = "false" 

enable_classiclink_dns_support = "false" 

preferred_number_of_public_subnets = 2

```
![](./Images/tfvars.png)
![](./Images/variable.png)
![](./Images/main.png)

![](./Images/file_structure.png)

![](./Images/final_plan.png)
![](./Images/created.png)
![](./Images/destroy_final.png)
