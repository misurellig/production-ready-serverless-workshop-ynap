# Module 1: Deploying Lambda functions with Terraform

## Installation

Go to https://www.terraform.io and follow the instruction to install Terraform.

## Create a Serverless project

**Goal:** Create and deploy AWS Lambda handler code using Terraform.

<details>
<summary><b>HOW TO create and deploy a Lambda function with Terraform</b></summary><p>

1. Create a directory for your serverless project.

    ```
    mkdir workshop
    cd workshop
    ```

2. Initialise the project:

    `npm init -y`

3. Add a folder called `functions`

4. Add a file in the `functions` folder, call it `hello.js`

5. Copy the following into the `hello.js` module

```javascript
module.exports.handler = async (event) => {
  return {
    statusCode: 200,
    body: JSON.stringify({
      input: event
    })
  }
}
```

6. In the root of the project, add another folder, call it `terraform`

7. In the `terraform` folder, add a file called `provider.tf`

8. Copy the following into `provider.tf`

```terraform
provider "aws" {
  region = "us-east-1"
}
```

9. In the `terraform` folder, add another file called `hello.tf`

10. Copy the following into `hello.tf`

```terraform
resource "aws_lambda_function" "hello" {
  function_name = "hello-ynap-${var.my_name}"

  s3_bucket = "ynap-production-ready-serverless-${var.my_name}"
  s3_key    = "workshop.zip"

  # "main" is the file within the zip file above (functions/hello.js) 
  # "handler" is the name of the exported property in functions/hello.js
  handler = "functions/hello.handler"
  runtime = "nodejs8.10"

  role = "${aws_iam_role.hello_lambda_role.arn}"
}

# IAM role which dictates what other AWS services the hello function can access
resource "aws_iam_role" "hello_lambda_role" {
  name = "hello-lambda-role-${var.my_name}"

  assume_role_policy = <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Action": "sts:AssumeRole",
      "Principal": {
        "Service": "lambda.amazonaws.com"
      },
      "Effect": "Allow",
      "Sid": ""
    }
  ]
}
EOF
}

resource "aws_iam_role_policy_attachment" "hello_lambda_role_policy" {
  role       = "${aws_iam_role.hello_lambda_role.name}"
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole"
}
```

Here we are creating the `hello-ynap-<suffix>` (where `suffix` is your name) Lambda function, alongside the IAM role it'll use. One thing to note is that, it's pointing to a deployment artifact in a S3 bucket that is suffixed with your name.

Let's go ahead and create the variable for the suffix.

11. In the `terraform` folder, add a file called `variables.tf`

12. Copy the following into the `variables.tf` file:

```terraform
variable "my_name" {
  description = "The name of the student"
  type        = "string"
}
```

13. To create the deployment artifact, run the following command from the **root of the project**

`zip -r workshop.zip * -x terraform/*`

You should see something like this in the console:

```
  adding: functions/ (stored 0%)
  adding: functions/hello.js (deflated 15%)
  adding: package.json (deflated 32%)
  adding: terraform/ (stored 0%)
```

14. Now we need to create the S3 bucket itself, run the following command **don't forget to replace the suffix with your name**

`aws s3api create-bucket --bucket=ynap-production-ready-serverless-<suffix> --region=us-east-1`

15. To upload the deployment artifact `workshop.zip`, run the following command **don't forget to replace the suffix with your name**

`aws s3 cp workshop.zip s3://ynap-production-ready-serverless-<suffix>/workshop.zip`

16. And we're ready to deploy! Go to the `terraform` folder

`cd terraform`

and run the command

`terraform init`

You should see something along the lines of:

```
L01013552:terraform yan.cui$ terraform init

Initializing provider plugins...
- Checking for available provider plugins on https://releases.hashicorp.com...
- Downloading plugin for provider "aws" (2.12.0)...

The following providers do not have any version constraints in configuration,
so the latest version was installed.

To prevent automatic upgrades to new major versions that may contain breaking
changes, it is recommended to add version = "..." constraints to the
corresponding provider blocks in configuration, with the constraint strings
suggested below.

* provider.aws: version = "~> 2.12"

Terraform has been successfully initialized!

You may now begin working with Terraform. Try running "terraform plan" to see
any changes that are required for your infrastructure. All Terraform commands
should now work.

If you ever set or change modules or backend configuration for Terraform,
rerun this command to reinitialize your working directory. If you forget, other
commands will detect it and remind you to do so if necessary.
```

17. Now run `terraform apply -var 'my_name=xxx'` **replace xxx with your name**

e.g. `terraform apply -var 'my_name=yancui'`

You should see something along the lines of:

```
An execution plan has been generated and is shown below.
Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  + aws_iam_role.hello_lambda_role
      id:                             <computed>
      arn:                            <computed>
      assume_role_policy:             "{\n  \"Version\": \"2012-10-17\",\n  \"Statement\": [\n    {\n      \"Action\": \"sts:
AssumeRole\",\n      \"Principal\": {\n        \"Service\": \"lambda.amazonaws.com\"\n      },\n      \"Effect\": \"Allow\",\
n      \"Sid\": \"\"\n    }\n  ]\n}\n"
      create_date:                    <computed>
      force_detach_policies:          "false"
      max_session_duration:           "3600"
      name:                           "hello-lambda-role-yancui"
      path:                           "/"
      unique_id:                      <computed>

  + aws_iam_role_policy_attachment.hello_lambda_role_policy
      id:                             <computed>
      policy_arn:                     "arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole"
      role:                           "hello-lambda-role-yancui"

  + aws_lambda_function.hello
      id:                             <computed>
      arn:                            <computed>
      function_name:                  "hello-ynap-yancui"
      handler:                        "functions/hello.handler"
      invoke_arn:                     <computed>
      last_modified:                  <computed>
      memory_size:                    "128"
      publish:                        "false"
      qualified_arn:                  <computed>
      reserved_concurrent_executions: "-1"
      role:                           "${aws_iam_role.hello_lambda_role.arn}"
      runtime:                        "nodejs8.10"
      s3_bucket:                      "ynap-production-ready-serverless-yancui"
      s3_key:                         "workshop.zip"
      source_code_hash:               <computed>
      source_code_size:               <computed>
      timeout:                        "3"
      tracing_config.#:               <computed>
      version:                        <computed>


Plan: 3 to add, 0 to change, 0 to destroy.

Do you want to perform these actions?
  Terraform will perform the actions described above.
  Only 'yes' will be accepted to approve.

  Enter a value:
```

Enter `yes` to continue.

18. Now that your function is deployed. Let's invoke it, replace `xxx` with your name and run the following command

`lambda invoke --region=us-east-1 --function-name=hello-ynap-xxx output.txt`

You should see

```json
{
    "ExecutedVersion": "$LATEST",
    "StatusCode": 200
}
```

and there should be an `output.txt` file inside the `terraform` folder. Open it and it should look like this:

```json
{"statusCode":200,"body":"{\"input\":{}}"}
```

</p></details>