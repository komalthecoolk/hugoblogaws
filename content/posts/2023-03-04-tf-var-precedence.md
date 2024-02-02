+++
title = 'Order of precedence of variables in Terraform'
date = 2023-03-04
draft = false
tags = [
    "terraform",
    "iac",
    "devops"
    ]
toc = true
+++

## Variables in Terraform

Variables (more specifically input variables), as in most programming languages, are objects that can hold temporary values of different types that can passed to the program. They allow us to generate different outputs or functionality from a code while keeping the code consistent based on the values provided at the time of execution.

Since Terraform is a directory based infrastructure-as-code tool, it evaluates all of the relevant configuration files in a directory at the time of execution. Terraform configuration can be included in a single configuration file or split into multiple files for ease or organizing. This feature combined with the ability to call [modules](https://developer.hashicorp.com/terraform/language/modules) introduces a slight complexity with variables as a single variable can be declared from a host of locations.

This post is to understand how duplicate varibles are evaluated and what is the order of precedence when an input variable is provided at multiple locations.

![](/images/001-tf-var-precedence.png)

The above image summarizes the different locations we can input variables from and the order of precedence. Let's try this out using a very basic Terraform code that takes an input variable and outputs a formatted string that shows the location of the variable.

### Step 1: Current file structure and file content

To test this flow, I have some Terraform config files in a folder as shown below.

{{< highlight bash >}}
> tree
.
├── main.tf
├── prod.tfvars
├── secrets.auto.tfvars
├── shared.tfvars
├── terraform.tfvars
└── variables.tf

0 directories, 6 files
{{< /highlight >}}

Config files `'main.tf'` and `'variables.tf'` are populated with minimal code that takes in two variables called `source_of_var_1` and `source_of_var_2` and then prints them out in the Terraform outputs. We will input the same variables from multiple locations and the variables have values in a way to help us identify which file or location Terraform is picking the variable from. The rest of the files are initially empty.


{{< highlight hcl >}}
#main.tf

locals {
  finding_source_of_var_1 = var.source_of_var_1
  finding_source_of_var_2 = var.source_of_var_2
}

output "source_of_var_out_1" {
  value = "The source of the variable_1 is ${local.finding_source_of_var_1}"
}

output "source_of_var_out_2" {
  value = "The source of the variable_2 is ${local.finding_source_of_var_2}"
}
{{< /highlight >}}

{{< highlight hcl >}}
#variables.tf

variable "source_of_var_1" {
  type = string
  default = "file_variables.tf"
}

variable "source_of_var_2" {
  type = string
  default = "file_variables.tf"
}
{{< /highlight >}}


### Step 2: Inputting the variable as default value in definition in variables.tf.
To start off, I am providing default value for both of the input variables as part of the variabe definition in `'variables.tf'` as highlighted above.

When I execute the Terraform code, we can see that it picks up the and outputs the correct variable as defined.

{{< highlight bash >}}
> terraform plan

Changes to Outputs:
  + source_of_var_out_1 = "The source of the variable_1 is file_variables.tf"
  + source_of_var_out_2 = "The source of the variable_2 is file_variables.tf"
{{< /highlight >}}

### Step 3: Inputting the variable value via Environmental Variables
Terraform has options to  set different environmental variables in our operating system that controls the Terraform log debugging level, path to save logs etc along with some input variables that can be passed on to Terraform at the time of execution. This is helpful when we have dedicated machines to run Terraform for specific workspaces/environments. This allows these variables to be securely used for deployment while being excluded from the Terraform files.

First is an example of how we set the environment variables for Terraform and verify.

{{< highlight bash >}}
> export TF_VAR_source_of_var_1=environment_variables
> export TF_VAR_source_of_var_2=environment_variables
> echo $TF_VAR_source_of_var_1
environment_variables
> echo $TF_VAR_source_of_var_2
environment_variables
{{< /highlight >}}

Now when we run Terraform without any additional inputs, we can see that Terraform prefers the environmental variables over the default variable definitions.

{{< highlight bash >}}
> terraform plan

Changes to Outputs:
  + source_of_var_out_1 = "The source of the variable_1 is environment_variables"
  + source_of_var_out_2 = "The source of the variable_2 is environment_variables"
{{< /highlight >}}

### Step 4: Inputting the variable value via terraform.tfvars
Now we keep all configuration as it is but introduce the same variable `source_of_var_1` in the file `'terraform.tfvars'` but with a different value. This is a special file that Terraform automatically picks up when executed, as long as it is in the working directory.

Below are the contents of `'terraform.tfvars'`.

{{< highlight hcl >}}
#terraform.tfvars

source_of_var_1 = "file_terraform.tfvars"
{{< /highlight >}}

When we execute Terraform, we can see that for the common variables present in `'variables.tf'` and `'terraform.tfvars'`, the latter is preferred but for those that are not common, Terraform falls back to the default value in `'variables.tf'` as seen below.

{{< highlight bash >}}
> terraform plan

Changes to Outputs:
  + source_of_var_out_1 = "The source of the variable_1 is file_terraform.tfvars"
  + source_of_var_out_2 = "The source of the variable_2 is environment_variables"
{{< /highlight >}}

### Step 5: What about *.auto.tfvars files ?

For some use cases where we want to input senstive variables such as passwords, we can move just those variables into a different kind of vars files named `'*.auto.tfvars'`. These can then be excluded from being checked into version-control/Github. Similar to the `'terraform.tfvars'` file which is automatically loaded by Terraform, all of the `'*.auto.tfvars'` in the working directory are loaded for use by Terraform at the time of execution. The variables in the `'*.auto.tfvars'` files have higher precedence than `'terraform.tfvars'`.

I'm introducing a new value to the variable `source_of_var_1` via `'secrets.auto.tfvars'` which has been empty until now.

{{< highlight hcl >}}
#secrets.auto.tfvars

source_of_var_1 = "file_secrets.auto.tfvars"
{{< /highlight >}}

The result of this on the Terraform execution is as follows.

{{< highlight bash >}}
> terraform plan

Changes to Outputs:
  + source_of_var_out_1 = "The source of the variable_1 is file_secrets.auto.tfvars"
  + source_of_var_out_2 = "The source of the variable_2 is environment_variables"
{{< /highlight >}}

In cases of multiple `'*.auto.tfvars'` files, the common input variables are picked from the files have names that are alphabetically later/higher get more precedence. Follow the example below to understand. I'm introducing a new value to the same variable `source_of_var_1` via a new file `'top_secrets.auto.tfvars'`. Since the letter 't' is lexicographically later to the letter 's', this file shoule be more preferred than `'secrets.auto.tfvars'`.

{{< highlight hcl >}}
#top_secrets.auto.tfvars

source_of_var_1 = "file_top_secrets.auto.tfvars"
{{< /highlight >}}

{{< highlight bash >}}
> terraform plan

Changes to Outputs:
  + source_of_var_out_1 = "The source of the variable_1 is file_top_secrets.auto.tfvars"
  + source_of_var_out_2 = "The source of the variable_2 is environment_variables"
{{< /highlight >}}

As we can see, without even mentioning the names of the `'*.auto.tfvars'` anywhere, Terraform has picked up both the `'*.auto.tfvars'` files and preferred the input variable from `'top_secrets.auto.tfvars'` as explained above.


### Step 6: Inputting the variable values via custom variables file *.tfvars
Now we move forward and introduce the same variable `source_of_var_1` in a custom vars file called `'prod.tfvars'`. This is a custom file and is not automatically used by Terraform at the time of execution. Such files may typically be used when introducing a few variables that need to be introduced at the time deployment in some cases but not are not part of the core/base deployment. For example, when we want to separate variables belonging to different environments such as prod, dev, stage etc.

{{< highlight hcl >}}
#prod.tfvars

source_of_var_1 = "file_prod.tfvars"
{{< /highlight >}}

When we run Terraform, this value from `'prod.tfvars'` is not used by default and Terraform falls back to the next best preferences, that are `'*.auto.tfvars'` > `'terraform.tfvars'` > 'environmental variables' > 'default variable defintion', in that order.

{{< highlight bash >}}
> terraform plan

Changes to Outputs:
  + source_of_var_out_1 = "The source of the variable_1 is file_top_secrets.auto.tfvars"
  + source_of_var_out_2 = "The source of the variable_2 is environment_variables"
{{< /highlight >}}

When using custom vars file, we need to specify the file name as an arguement in the command line.

{{< highlight bash >}}
> terraform plan -var-file="prod.tfvars"

Changes to Outputs:
  + source_of_var_out_1 = "The source of the variable_1 is file_prod.tfvars"
  + source_of_var_out_2 = "The source of the variable_2 is environment_variables"
{{< /highlight >}}

### Step 7: What happens when there are multiple custom *.tfvars files ?

While specific custom `'*.tfvars` files are required to provide specific input variables for specific isolated environments like prod, dev, stage etc, sometimes we need variables that are common to all environments such as those from shared services (*DNS, DHCP, Security components etc*). For such use cases, we may have to input an additional custom vars file (`'shared.tfvars'` in our case) to contain all the shared variables.

Terraform allows us to pass names of multiple `'*.tfvars` files on the same CLI command. But what happens if there are duplicate values for some variables in some of these files? Let's test it out.

I'm adding new values to the variables in `'shared.tfvars'` and the contents of both the `'*.tfvars` are shown combined here for brevity.

{{< highlight hcl >}}
#prod.tfvars

source_of_var_1 = "file_prod.tfvars"

------------------------------------------------------------------------

#shared.tfvars

source_of_var_1 = "file_shared.tfvars"
source_of_var_2 = "file_shared.tfvars"

{{< /highlight >}}

 When we pass both files to Terraform as input arguments on the CLI, it uses all the variables from both files that are not common but for the common variables, it only picks the ones passed on later in the CLI command. Follow the two CLI command examples and their output to understand this.

{{< highlight bash >}}
# Example 1: prod.tfvars first and then shared.tfvars
> terraform plan -var-file="prod.tfvars" -var-file="shared.tfvars"

Changes to Outputs:
  + source_of_var_out_1 = "The source of the variable_1 is file_shared.tfvars"
  + source_of_var_out_2 = "The source of the variable_2 is file_shared.tfvars"

-------------------------------------------------------------------------
# Example 2: shared.tfvars first and then prod.tfvars

> terraform plan -var-file="shared.tfvars" -var-file="prod.tfvars"

Changes to Outputs:
  + source_of_var_out_1 = "The source of the variable_1 is file_prod.tfvars"
  + source_of_var_out_2 = "The source of the variable_2 is file_shared.tfvars"
{{< /highlight >}}



### Step 8: Inputting the variable values via CLI command

Even after organizing our different variables in the appropriate files, there may be use cases where we have to input variables via the command line invocation. This may be the case where we have just one or two variables and don't need to add them in a file or we need to input passwords at the time of execution instead of saving them in a file. Terraform allows us to do this via the `'-var'` argument. This takes a higher precedence over all the files we have discussed until now.

The current status as per the file structure we have created until now is as follows:
- Without any command line arguments we have `*.auto.tfvars` > `terraform.tfvars` > 'default variable definition'
- With CLI arguments we have `*.tfvars` > `*.auto.tfvars` > `terraform.tfvars` > 'default variable definition'

{{< highlight bash >}}
> terraform plan

Changes to Outputs:
  + source_of_var_out_1 = "The source of the variable_1 is file_top_secrets.auto.tfvars"
  + source_of_var_out_2 = "The source of the variable_2 is environment_variables"
{{< /highlight >}}

I am going to manually input the same variable `source_of_var_1` on the CLI as an argument.

{{< highlight bash >}}
> terraform plan -var="source_of_var_1=first_cli_input"

Changes to Outputs:
  + source_of_var_out_1 = "The source of the variable_1 is first_cli_input"
  + source_of_var_out_2 = "The source of the variable_2 is environment_variables"
{{< /highlight >}}

What if we have two manual inputs as CLI arguments"? Similar to the file inputs, the latter one takes the precedence.

{{< highlight bash >}}
> terraform plan -var="source_of_var_1=first_cli_input" -var="source_of_var_1=second_cli_input"

Changes to Outputs:
  + source_of_var_out_1 = "The source of the variable_1 is second_cli_input"
  + source_of_var_out_2 = "The source of the variable_2 is environment_variables"
{{< /highlight >}}

What if we have one manual input and one `'*.tfvars'` file as CLI arguments? The same logic applies as the latter input gets precedence. A few examples to show that are below.

{{< highlight bash >}}
> terraform plan -var="source_of_var_1=first_cli_input" -var-file="prod.tfvars"

Changes to Outputs:
  + source_of_var_out_1 = "The source of the variable_1 is file_prod.tfvars"
  + source_of_var_out_2 = "The source of the variable_2 is environment_variables"

-------------------------------------------------------------------------
> terraform plan -var-file="prod.tfvars" -var="source_of_var_1=first_cli_input"

Changes to Outputs:
  + source_of_var_out_1 = "The source of the variable_1 is first_cli_input"
  + source_of_var_out_2 = "The source of the variable_2 is environment_variables"
{{< /highlight >}}
