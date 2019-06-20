# Module 2: Building API with API Gateway and Lambda

## Returning HTML from Lambda

**Goal:** Create and deploy an API that returns HTML instead of JSON

<details>
<summary><b>Add an API function</b></summary><p>

1. In the `functions` folder, add a file called `get-index.js`

2. Copy the following into `get-index.js`:

```javascript
const fs = require("fs")

let html

function loadHtml () {
  if (!html) {
    console.log('loading index.html...')
    html = fs.readFileSync('static/index.html', 'utf-8')
    console.log('loaded')
  }
  
  return html
}

module.exports.handler = async (event, context) => {
  const html = loadHtml()
  const response = {
    statusCode: 200,
    headers: {
      'Content-Type': 'text/html; charset=UTF-8'
    },
    body: html
  }

  return response
}
```

Notice that this function returns `statusCode`, `body` and `headers`. API Gateway would turn these into a HTTP response to the caller and return the content type as HTML.

Also note that the variable `html` is declared outside the `handler` function. As a global variable it persists between invocations. Once it's initialized during cold start, we wouldn't need to perform the extra IO to load the static `index.html` file on every invocation. This is a common (and recommended by the Lambda team) practice for taking advantage of container reuse so you don't have to load static configurations and templates on every invocation.

</p></details>

<details>
<summary><b>Add the static HTML</b></summary><p>

1. Add a folder in the root, called `static`.

2. Add a file `index.html` under the newly created `static` folder

3. Modify the `index.html` file to the following:

```xml
<!DOCTYPE html>
<html>
  <head>
    <meta charset="UTF-8">
    <title>Big Mouth</title>
    
    <style>
      .fullscreenDiv {
        background-color: #05bafd;
        width: 100%;
        height: auto;
        bottom: 0px;
        top: 0px;
        left: 0;
        position: absolute;
      }

      .column-container {
        padding: 0;
        margin: 0;
        list-style: none;
        display: -webkit-box;
        display: -moz-box;
        display: -ms-flexbox;
        display: -webkit-flex;
        display: flex;
        flex-flow: column;
        justify-content: center;
      }

      .item {
        padding: 5px;
        height: auto;
        margin-top: 10px;
        display: flex;
        flex-flow: row;
        justify-content: center;
      }

      input {
        font-family: Arial, Helvetica, sans-serif;
        font-size: 18px;
      }

      button {
        font-family: Arial, Helvetica, sans-serif;
        font-size: 18px;
      }
    </style>

    <script>
    </script>
  </head>

  <body>
    <div class="fullscreenDiv">
      <ul class="column-container">
        <li class="item">
          <img id="logo" src="https://d2qt42rcwzspd6.cloudfront.net/manning/big-mouth.png">
        </li>
        <li class="item">
          <input id="theme" type="text" size="50" placeholder="enter a theme, eg. cartoon"/>
          <button onclick="search()">Find Restaurants</button>
        </li>
      </ul>
  </div>
  </body>

</html>
```

Your folder structure should look like this at this point:

```
functions
  |-- hello.js
  |-- get-index.js
terraform
  |-- hello.tf
  |-- provider.tf
  |-- variables.tf
static
  |-- index.html
package.json
```
</p></details>

<details>
<summary><b>Add the Terraform scripts</b></summary><p>

1. In the `terraform` folder, add a file called `locals.tf`. We will define recurring patterns and names here, such as a prefix for all our functions.

2. Copy the following into `locals.tf`:

```terraform
locals {
  function_prefix = "${var.service_name}-${var.stage}-${var.my_name}"
  deployment_bucket = "ynap-production-ready-serverless-${var.my_name}"
  deployment_key = "workshop/${var.file_name}.zip"
}
```

We have specified the deployment bucket and key where we will expect the deployment artifacts to be. We will deal with these in a moment.

Also note that the `function_prefix` local variable references two new input variables. Let's add them.

3. Open the `terraform/variables.tf` file and **add the following to the end of the file**:

```terraform
variable "service_name" {
  description = "The name of the student"
  type        = "string"
  default     = "production-ready-serverless"
}

variable "stage" {
  description = "The name of the stage, e.g. dev, staging, prod"
  type        = "string"
  default     = "dev"
}

variable "file_name" {
  description = "The name of the deployment package"
  type        = "string"
}
```

Notice that we also added a `file_name` input variable. This is where we tell Terraform where the deployment artifact for our function would be.

4. In the `terraform` folder, add another file called `get-index.tf`.

5. Copy the following into `get-index.tf`:

```terraform
resource "aws_lambda_function" "get_index" {
  function_name = "${local.function_prefix}-get-index"

  s3_bucket = "${local.deployment_bucket}"
  s3_key    = "${local.deployment_key}"

  handler = "functions/get-index.handler"
  runtime = "nodejs8.10"

  role = "${aws_iam_role.get_index_lambda_role.arn}"
}

# IAM role which dictates what other AWS services the hello function can access
resource "aws_iam_role" "get_index_lambda_role" {
  name = "${local.function_prefix}-get-index-lambda-role"

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

resource "aws_iam_role_policy_attachment" "get_index_lambda_role_policy" {
  role       = "${aws_iam_role.get_index_lambda_role.name}"
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole"
}
```

6. We still need to associate the new function with the `/` endpoint of our API. But first, let's remove the old `hello` endpoint from our API. Open the `terraform/apigateway.tf` file and **remove** the following lines from the file:

```terraform
resource "aws_api_gateway_resource" "hello" {
  rest_api_id = "${aws_api_gateway_rest_api.api.id}"
  parent_id   = "${aws_api_gateway_rest_api.api.root_resource_id}"
  path_part   = "hello"
}

resource "aws_api_gateway_method" "hello-get" {
  rest_api_id   = "${aws_api_gateway_rest_api.api.id}"
  resource_id   = "${aws_api_gateway_resource.hello.id}"
  http_method   = "GET"
  authorization = "NONE"
}

resource "aws_api_gateway_integration" "hello-lambda" {
  rest_api_id = "${aws_api_gateway_rest_api.api.id}"
  resource_id = "${aws_api_gateway_method.hello-get.resource_id}"
  http_method = "${aws_api_gateway_method.hello-get.http_method}"

  integration_http_method = "POST"
  type                    = "AWS_PROXY"
  uri                     = "${aws_lambda_function.hello.invoke_arn}"
}
```

and **remove** this line as well:

```terraform
resource "aws_lambda_permission" "apigw" {
  statement_id  = "AllowAPIGatewayInvoke"
  action        = "lambda:InvokeFunction"
  function_name = "${aws_lambda_function.hello.arn}"
  principal     = "apigateway.amazonaws.com"

  # The /*/* portion grants access from any method on any resource
  # within the API Gateway "REST API".
  source_arn = "${aws_api_gateway_stage.stage.execution_arn}/*/*"
}
```

7. Staying in the `terraform/apigateway.tf` file, **add the following to the end of the file**:

```terraform
# GET-INDEX
resource "aws_api_gateway_method" "get_index_get" {
  rest_api_id   = "${aws_api_gateway_rest_api.api.id}"
  resource_id   = "${aws_api_gateway_rest_api.api.root_resource_id}"
  http_method   = "GET"
  authorization = "NONE"
}

resource "aws_api_gateway_integration" "get_index_lambda" {
  rest_api_id = "${aws_api_gateway_rest_api.api.id}"
  resource_id = "${aws_api_gateway_method.get_index_get.resource_id}"
  http_method = "${aws_api_gateway_method.get_index_get.http_method}"

  integration_http_method = "POST"
  type                    = "AWS_PROXY"
  uri                     = "${aws_lambda_function.get_index.invoke_arn}"
}

resource "aws_lambda_permission" "apigw_get_index" {
  statement_id  = "AllowAPIGatewayInvoke"
  action        = "lambda:InvokeFunction"
  function_name = "${aws_lambda_function.get_index.arn}"
  principal     = "apigateway.amazonaws.com"

  # The /*/* portion grants access from any method on any resource
  # within the API Gateway "REST API".
  source_arn = "${aws_api_gateway_stage.stage.execution_arn}/*/*"
}
```

and look for the resource `aws_api_gateway_deployment.api`, where we still have a dependency on the now deleted `hello-lambda` resource. **Replace** this resource definition with the following

```terraform
resource "aws_api_gateway_deployment" "api" {
  depends_on = [
    "aws_api_gateway_integration.get_index_lambda"
  ]

  lifecycle {
    create_before_destroy = true
  }

  rest_api_id = "${aws_api_gateway_rest_api.api.id}"
  stage_name  = ""

  variables {
    deployed_at = "${timestamp()}"
  }
}
```

8. In the `terraform` folder, add another file `outputs.tf` so we can the invoke URL for our API and so on.

9. Copy the following into `terraform/outputs.tf`

```terraform
data "aws_caller_identity" "current" {}

output "account_id" {
  value = "${data.aws_caller_identity.current.account_id}"
}

output "invoke_url" {
  value = "${aws_api_gateway_stage.stage.invoke_url}"
}
```

</p></details>

<details>
<summary><b>Deploy the serverless project</b></summary><p>

1. Instead of running commands manually, let's automate it with a simple script. In the root of the project, add a file called `build.sh`.

2. Copy the following into `build.sh` and **replace** the two occurrances of `<your_name>`.

```bash
#!/bin/bash
set -e
set -o pipefail

instruction()
{
  echo "usage: ./build.sh deploy <stage>"
  echo ""
  echo "stage: eg. dev, staging, prod, ..."
  echo ""
  echo "for example: ./deploy.sh dev"
}

if [ $# -eq 0 ]; then
  instruction
  exit 1
elif [ "$1" = "deploy" ] && [ $# -eq 2 ]; then
  STAGE=$2

  npm install
  zip -r workshop.zip functions static node_modules

  MD5=$(md5 -q workshop.zip)
  aws s3 cp workshop.zip s3://ynap-production-ready-serverless-<your_name>/workshop/$MD5.zip
  
  cd terraform
  terraform apply --var "my_name=<your_name>" --var "file_name=$MD5"
else
  instruction
  exit 1
fi
```

3. Run the command `chmod +x build.sh`

4. Now we're ready to deploy our project with `./build.sh dev` and deploy, answer `yes` to confirm the deployment.

Once the deployment is finished, you should be able to go to the root URL of project and see this.

![](/images/mod02-001.png)

</p></details>

## Creating the restaurants API

**Goal:** Create and deploy the `/restaurants` endpoint

<details>
<summary><b>Add a function to return restaurants</b></summary><p>

1. Add a file `get-restaurants.js` file to the `functions` folder

2. Install the `aws-sdk` package from the **project root**

`npm install --save aws-sdk`

3. Copy the following into the `get-restaurants.js`

```javascript
const AWS = require('aws-sdk')
const dynamodb = new AWS.DynamoDB.DocumentClient()

const defaultResults = process.env.defaultResults || 8
const tableName = process.env.restaurants_table

const getRestaurants = async (count) => {
  const req = {
    TableName: tableName,
    Limit: count
  }

  const resp = await dynamodb.scan(req).promise()
  return resp.Items
}

module.exports.handler = async (event, context) => {
  const restaurants = await getRestaurants(defaultResults)
  const response = {
    statusCode: 200,
    body: JSON.stringify(restaurants)
  }

  return response
}
```

This function depends on two environment variables:

* `defaultResults` [optional] : how many restaurants to return

* `restaurants_table` [required] : name of the restaurants DynamoDB table

</p>
</details>

<details>
<summary><b>Update the Terraform scripts</b></summary><p>

1. In the `terraform` folder, add a new file, called `get-restaurants.tf`

2. Copy the following into `get-restaurants.tf`

```terraform
resource "aws_lambda_function" "get_restaurants" {
  function_name = "${local.function_prefix}-get-restaurants"

  s3_bucket = "${local.deployment_bucket}"
  s3_key    = "${local.deployment_key}"

  handler = "functions/get-restaurants.handler"
  runtime = "nodejs8.10"

  role = "${aws_iam_role.get_restaurants_lambda_role.arn}"

  environment {
    variables = {
      restaurants_table = "${aws_dynamodb_table.restaurants_table.name}"
    }
  }
}

# IAM role which dictates what other AWS services the hello function can access
resource "aws_iam_role" "get_restaurants_lambda_role" {
  name = "${local.function_prefix}-get-restaurants-role"

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

resource "aws_iam_role_policy_attachment" "get_restaurants_lambda_role_policy" {
  role       = "${aws_iam_role.get_restaurants_lambda_role.name}"
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole"
}
```

Note that the new function declares an environment variable `restaurants_table`. We will need to create a DynamoDB table with a matching name later. But for now, let's first update the API Gateway configuration.

3. **Add** the following to **the end** of `terraform/apigateway.tf`

```terraform
# GET-RESTAURANTS
resource "aws_api_gateway_resource" "get_restaurants" {
  rest_api_id = "${aws_api_gateway_rest_api.api.id}"
  parent_id   = "${aws_api_gateway_rest_api.api.root_resource_id}"
  path_part   = "restaurants"
}

resource "aws_api_gateway_method" "get_restaurants_get" {
  rest_api_id   = "${aws_api_gateway_rest_api.api.id}"
  resource_id   = "${aws_api_gateway_resource.get_restaurants.id}"
  http_method   = "GET"
  authorization = "NONE"
}

resource "aws_api_gateway_integration" "get_restaurants_lambda" {
  rest_api_id = "${aws_api_gateway_rest_api.api.id}"
  resource_id = "${aws_api_gateway_method.get_restaurants_get.resource_id}"
  http_method = "${aws_api_gateway_method.get_restaurants_get.http_method}"

  integration_http_method = "POST"
  type                    = "AWS_PROXY"
  uri                     = "${aws_lambda_function.get_restaurants.invoke_arn}"
}

resource "aws_lambda_permission" "apigw_get_restaurants" {
  statement_id  = "AllowAPIGatewayInvoke"
  action        = "lambda:InvokeFunction"
  function_name = "${aws_lambda_function.get_restaurants.arn}"
  principal     = "apigateway.amazonaws.com"

  source_arn = "${aws_api_gateway_stage.stage.execution_arn}/*/*"
}
```

4. Staying in `terraform/apigateway.tf`, we need to **update** the `aws_api_gateway_deployment.api` resource to add the new `/restaurants` endpoint to its dependencies.

```terraform
resource "aws_api_gateway_deployment" "api" {
  depends_on = [
    "aws_api_gateway_integration.get_index_lambda",
    "aws_api_gateway_integration.get_restaurants_lambda"
  ]

  lifecycle {
    create_before_destroy = true
  }

  rest_api_id = "${aws_api_gateway_rest_api.api.id}"
  stage_name  = ""

  variables {
    deployed_at = "${timestamp()}"
  }
}
```

5. In the `terraform` folder, add a folder called `dynamodb.tf`.

6. Copy the following into `dynamodb.tf`

```terraform
resource "aws_dynamodb_table" "restaurants_table" {
  name           = "restaurants_${var.stage}_${var.my_name}"
  billing_mode   = "PAY_PER_REQUEST"  
  hash_key       = "name"

  attribute {
    name = "name"
    type = "S"
  }
}
```

7. And now we need to make sure our `get-restaurants` function has the permission to scan the restaurants DynamoDB table. Open the `terraform/get-restaurants.tf` file and **add the following to the end of the file**.

```terraform
resource "aws_iam_policy" "get_restaurants_lambda_dynamodb_policy" {
  name = "get_restaurants_dynamodb_scan"
  path = "/"
  policy = <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "dynamodb:scan",
      "Resource": "${aws_dynamodb_table.restaurants_table.arn}"
    }
  ]
}
EOF
}

resource "aws_iam_role_policy_attachment" "get_restaurants_lambda_dynamodb_policy" {
  role       = "${aws_iam_role.get_restaurants_lambda_role.name}"
  policy_arn = "${aws_iam_policy.get_restaurants_lambda_dynamodb_policy.arn}"
}
```

</p>
</details>

<details>
<summary><b>Deploy the project</b></summary><p>

1. Run the command `./build.sh deploy dev` and answer `yes` to confirm the deployment.

</p>
</details>

<details>
<summary><b>Seed the DynamoDB table</b></summary><p>

1. Add a file `seed-restaurants.js` to the project root

2. Modify `seed-restaurants.js` to the following (make sure you change `restaurants_dev_<suffix>` to your table name):

```javascript
const AWS = require('aws-sdk')
AWS.config.region = 'us-east-1'
const dynamodb = new AWS.DynamoDB.DocumentClient()

let restaurants = [
  {
    name: "Fangtasia",
    image: "https://d2qt42rcwzspd6.cloudfront.net/manning/fangtasia.png",
    themes: ["true blood"]
  },
  { 
    name: "Shoney's", 
    image: "https://d2qt42rcwzspd6.cloudfront.net/manning/shoney's.png", 
    themes: ["cartoon", "rick and morty"] 
  },
  { 
    name: "Freddy's BBQ Joint", 
    image: "https://d2qt42rcwzspd6.cloudfront.net/manning/freddy's+bbq+joint.png", 
    themes: ["netflix", "house of cards"] 
  },
  { 
    name: "Pizza Planet", 
    image: "https://d2qt42rcwzspd6.cloudfront.net/manning/pizza+planet.png", 
    themes: ["netflix", "toy story"] 
  },
  { 
    name: "Leaky Cauldron", 
    image: "https://d2qt42rcwzspd6.cloudfront.net/manning/leaky+cauldron.png", 
    themes: ["movie", "harry potter"] 
  },
  { 
    name: "Lil' Bits", 
    image: "https://d2qt42rcwzspd6.cloudfront.net/manning/lil+bits.png", 
    themes: ["cartoon", "rick and morty"] 
  },
  { 
    name: "Fancy Eats", 
    image: "https://d2qt42rcwzspd6.cloudfront.net/manning/fancy+eats.png", 
    themes: ["cartoon", "rick and morty"] 
  },
  { 
    name: "Don Cuco", 
    image: "https://d2qt42rcwzspd6.cloudfront.net/manning/don%20cuco.png", 
    themes: ["cartoon", "rick and morty"] 
  },
];

let putReqs = restaurants.map(x => ({
  PutRequest: {
    Item: x
  }
}))

let req = {
  RequestItems: {
    'restaurants_dev_<suffix>': putReqs
  }
}
dynamodb.batchWrite(req).promise().then(() => console.log("all done"))
```

3. Run the `seed-restaurants.js` script

`node seed-restaurants.js`

</p></details>

## Displaying restaurants in the landing page

**Goal:** Displays the restaurants in `index.html`

<details>
<summary><b>Update index.html to allow templating with Mustache</b></summary><p>

1. Modify `index.html` to the following:

```html
<!DOCTYPE html>
<html>
  <head>
    <meta charset="UTF-8">
    <title>Big Mouth</title>
    
    <style>
      .fullscreenDiv {
        background-color: #05bafd;
        width: 100%;
        height: auto;
        bottom: 0px;
        top: 0px;
        left: 0;
        position: absolute;
      }
      .restaurantsDiv {
        background-color: #ffffff;
        width: 100%;
        height: auto;
      }
      .dayOfWeek {
        font-family: Arial, Helvetica, sans-serif;
        font-size: 32px;
        padding: 10px;
        height: auto;
        display: flex;
        justify-content: center;
      }
      .column-container {
        padding: 0;
        margin: 0;
        list-style: none;
        display: flex;
        flex-flow: column;
        flex-wrap: wrap;
        justify-content: center;
      }
      .row-container {
        padding: 0;
        margin: 0;
        list-style: none;
        display: flex;
        flex-flow: row;
        flex-wrap: wrap;
        justify-content: center;
      }
      .item {
        padding: 5px;
        height: auto;
        margin-top: 10px;
        display: flex;
        flex-flow: row;
        flex-wrap: wrap;
        justify-content: center;
      }
      .restaurant {
        background-color: #00a8f7;
        border-radius: 10px;
        padding: 5px;
        height: auto;
        width: auto;
        margin-left: 40px;
        margin-right: 40px;
        margin-top: 15px;
        margin-bottom: 0px;
        display: flex;
        justify-content: center;
      }
      .restaurant-name {
        font-size: 24px;
        font-family:Arial, Helvetica, sans-serif;
        color: #ffffff;
        padding: 10px;
        margin: 0px;
      }
      .restaurant-image {
        padding-top: 0px;
        margin-top: 0px;
      }
      input {
        font-family: Arial, Helvetica, sans-serif;
        font-size: 18px;
      }
      button {
        font-family: Arial, Helvetica, sans-serif;
        font-size: 18px;
      }
    </style>

    <script>
    </script>
  </head>

  <body>
    <div class="fullscreenDiv">
      <ul class="column-container">
        <li class="item">
          <img id="logo" src="https://d2qt42rcwzspd6.cloudfront.net/manning/big-mouth.png">
        </li>
        <li class="item">
          <input id="theme" type="text" size="50" placeholder="enter a theme, eg. cartoon"/>
          <button onclick="search()">Find Restaurants</button>
        </li>
        <li>
          <div class="restaurantsDiv column-container">
            <b class="dayOfWeek">{{dayOfWeek}}</b>
            <ul class="row-container">
              {{#restaurants}}
              <li class="restaurant">
                <ul class="column-container">
                    <li class="item restaurant-name">{{name}}</li>
                    <li class="item restaurant-image">
                      <img src="{{image}}">
                    </li>
                </ul>
              </li>
              {{/restaurants}}
            </ul>
          </div>
        </li>
      </ul>
  </div>
  </body>

</html>
```

</p></details>

<details>
<summary><b>Render the index.html with restaurants</b></summary><p>

1. Go back to the project root and install `mustache` as dependency

`npm install --save mustache`

2. Install `superagent` as dependency

`npm install --save superagent`

3. Install `superagent-promise` as dependency

`npm install --save superagent-promise`

4. **Replace** `get-index.js` with the following:

```javascript
const fs = require("fs")
const Mustache = require('mustache')
const http = require('superagent-promise')(require('superagent'), Promise)

const restaurantsApiRoot = process.env.restaurants_api
const days = ['Sunday', 'Monday', 'Tuesday', 'Wednesday', 'Thursday', 'Friday', 'Saturday']

let html

function loadHtml () {
  if (!html) {
    console.log('loading index.html...')
    html = fs.readFileSync('static/index.html', 'utf-8')
    console.log('loaded')
  }
  
  return html
}

const getRestaurants = async () => {
  return (await http.get(restaurantsApiRoot)).body
}

module.exports.handler = async (event, context) => {
  const template = loadHtml()
  const restaurants = await getRestaurants()
  const dayOfWeek = days[new Date().getDay()]
  const html = Mustache.render(template, { dayOfWeek, restaurants })
  const response = {
    statusCode: 200,
    headers: {
      'Content-Type': 'text/html; charset=UTF-8'
    },
    body: html
  }

  return response
}
```

After this change, the `get-index` function needs the `restaurants_api` environment variable to know where the `/restaurants` endpoint is.

5. Modify the `terraform/get-index.tf` file to add an environment variable to the `get-index` function. Look for the `aws_lambda_function.get_index` resource and **replace** it with the following

```terraform
resource "aws_lambda_function" "get_index" {
  function_name = "${local.function_prefix}-get-index"

  s3_bucket = "${local.deployment_bucket}"
  s3_key    = "${local.deployment_key}"

  handler = "functions/get-index.handler"
  runtime = "nodejs8.10"

  role = "${aws_iam_role.get_index_lambda_role.arn}"

  environment {
    variables = {
      restaurants_api = "https://${aws_api_gateway_rest_api.api.id}.execute-api.us-east-1.amazonaws.com/${var.stage}/restaurants"
    }
  }
}
```

</p></details>

<details>
<summary><b>Deploy the serverless project</b></summary><p>

1. It's time to deploy everything again. Run the command `./build.sh deploy dev` and answer `yes` to confirm the deployment.

Once the deployment is finished, you should be able to go to the root URL of project and see some restaurants.

![](/images/mod02-002.png)

</p>
</details>

## Create POST /restaurants/search endpoint

**Goal:** Set up a `/restaurants/search` endpoint for searching restaurants by theme.

<details>
<summary><b>Add search-restaurants function</b></summary><p>

1. In the `functions` folder, add a new file called `search-restaurants.js`.

2. Copy the following into `functions/search-restaurants.js`

```javascript
const AWS = require('aws-sdk')
const dynamodb = new AWS.DynamoDB.DocumentClient()

const defaultResults = process.env.defaultResults || 8
const tableName = process.env.restaurants_table

const findRestaurantsByTheme = async (theme, count) => {
  const req = {
    TableName: tableName,
    Limit: count,
    FilterExpression: "contains(themes, :theme)",
    ExpressionAttributeValues: { ":theme": theme }
  }

  const resp = await dynamodb.scan(req).promise()
  return resp.Items
}

module.exports.handler = async (event, context) => {
  const req = JSON.parse(event.body)
  const theme = req.theme
  const restaurants = await findRestaurantsByTheme(theme, defaultResults)
  const response = {
    statusCode: 200,
    body: JSON.stringify(restaurants)
  }

  return response
}
```

</p></details>

<details>
<summary><b>Add the Terraform scripts</b></summary><p>

1. In the `terraform` folder, add a file called `search-restaurants.tf`

2. Copy the following into `terraform/search-restaurants.tf`

```terraform
resource "aws_lambda_function" "search_restaurants" {
  function_name = "${local.function_prefix}-search-restaurants"

  s3_bucket = "${local.deployment_bucket}"
  s3_key    = "${local.deployment_key}"

  handler = "functions/search-restaurants.handler"
  runtime = "nodejs8.10"

  role = "${aws_iam_role.search_restaurants_lambda_role.arn}"

  environment {
    variables = {
      restaurants_table = "${aws_dynamodb_table.restaurants_table.name}"
    }
  }
}

# IAM role which dictates what other AWS services the hello function can access
resource "aws_iam_role" "search_restaurants_lambda_role" {
  name = "${local.function_prefix}-search-restaurants-role"

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

resource "aws_iam_role_policy_attachment" "search_restaurants_lambda_role_policy" {
  role       = "${aws_iam_role.search_restaurants_lambda_role.name}"
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole"
}

resource "aws_iam_policy" "search_restaurants_lambda_dynamodb_policy" {
  name = "search_restaurants_dynamodb_scan"
  path = "/"
  policy = <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "dynamodb:scan",
      "Resource": "${aws_dynamodb_table.restaurants_table.arn}"
    }
  ]
}
EOF
}

resource "aws_iam_role_policy_attachment" "search_restaurants_lambda_dynamodb_policy" {
  role       = "${aws_iam_role.search_restaurants_lambda_role.name}"
  policy_arn = "${aws_iam_policy.search_restaurants_lambda_dynamodb_policy.arn}"
}
```

3. Open `terraform/apigateway.tf`, and **add** the following to the end of the file

```terraform
# SEARCH-RESTAURANTS
resource "aws_api_gateway_resource" "search_restaurants" {
  rest_api_id = "${aws_api_gateway_rest_api.api.id}"
  parent_id   = "${aws_api_gateway_resource.get_restaurants.id}"
  path_part   = "search"
}

resource "aws_api_gateway_method" "search_restaurants_post" {
  rest_api_id   = "${aws_api_gateway_rest_api.api.id}"
  resource_id   = "${aws_api_gateway_resource.search_restaurants.id}"
  http_method   = "POST"
  authorization = "NONE"
}

resource "aws_api_gateway_integration" "search_restaurants_lambda" {
  rest_api_id = "${aws_api_gateway_rest_api.api.id}"
  resource_id = "${aws_api_gateway_method.search_restaurants_post.resource_id}"
  http_method = "${aws_api_gateway_method.search_restaurants_post.http_method}"

  integration_http_method = "POST"
  type                    = "AWS_PROXY"
  uri                     = "${aws_lambda_function.search_restaurants.invoke_arn}"
}

resource "aws_lambda_permission" "apigw_search_restaurants" {
  statement_id  = "AllowAPIGatewayInvoke"
  action        = "lambda:InvokeFunction"
  function_name = "${aws_lambda_function.search_restaurants.arn}"
  principal     = "apigateway.amazonaws.com"

  source_arn = "${aws_api_gateway_stage.stage.execution_arn}/*/*"
}
```

4. Staying in the `terraform/apigateway.tf` file, look for the resource `aws_api_gateway_deployment.api` and **replace** it with the following

```terraform
resource "aws_api_gateway_deployment" "api" {
  depends_on = [
    "aws_api_gateway_integration.get_index_lambda",
    "aws_api_gateway_integration.get_restaurants_lambda",
    "aws_api_gateway_integration.search_restaurants_lambda"
  ]

  lifecycle {
    create_before_destroy = true
  }

  rest_api_id = "${aws_api_gateway_rest_api.api.id}"
  stage_name  = ""

  variables {
    deployed_at = "${timestamp()}"
  }
}
```

5. Redeploy the project by running the command `./build.sh deploy dev`

6. Once the deployment is done, curl the `/restaurants/search` endpoint for the `cartoon` theme. **Don't forget to change the url to invoke URL from the Terraform output**

`curl -d '{"theme":"cartoon"}' -H "Content-Type: application/json" -X POST https://xxx-api.us-east-1.amazonaws.com/dev/restaurants/search`

and you should see that the response is

```json
[
  {
    "name": "Shoney's",
    "image": "https:\/\/d2qt42rcwzspd6.cloudfront.net\/manning\/shoney's.png",
    "themes": [
      "cartoon",
      "rick and morty"
    ]
  },
  {
    "name": "Lil' Bits",
    "image": "https:\/\/d2qt42rcwzspd6.cloudfront.net\/manning\/lil+bits.png",
    "themes": [
      "cartoon",
      "rick and morty"
    ]
  },
  {
    "name": "Fancy Eats",
    "image": "https:\/\/d2qt42rcwzspd6.cloudfront.net\/manning\/fancy+eats.png",
    "themes": [
      "cartoon",
      "rick and morty"
    ]
  },
  {
    "name": "Don Cuco",
    "image": "https:\/\/d2qt42rcwzspd6.cloudfront.net\/manning\/don%20cuco.png",
    "themes": [
      "cartoon",
      "rick and morty"
    ]
  }
]
```

</p></details>

<details>
<summary><b>Integrate the index.html with the /restaurants/search endpoint</b></summary><p>

1. **Replace** the content of `static/index.html` with the following

```html
<!DOCTYPE html>
<html>
  <head>
    <meta charset="UTF-8">
    <title>Big Mouth</title>

    <script src="https://code.jquery.com/jquery-3.2.1.min.js" 
            integrity="sha256-hwg4gsxgFZhOsEEamdOYGBf13FyQuiTwlAQgxVSNgt4="
            crossorigin="anonymous"></script>
    <script src="https://code.jquery.com/ui/1.12.1/jquery-ui.min.js" 
            integrity="sha384-Dziy8F2VlJQLMShA6FHWNul/veM9bCkRUaLqr199K94ntO5QUrLJBEbYegdSkkqX" 
            crossorigin="anonymous"></script>
    <link rel="stylesheet" href="https://code.jquery.com/ui/1.12.1/themes/base/jquery-ui.css">
    
    <style>
      .fullscreenDiv {
        background-color: #05bafd;
        width: 100%;
        height: auto;
        bottom: 0px;
        top: 0px;
        left: 0;
        position: absolute;
      }
      .restaurantsDiv {
        background-color: #ffffff;
        width: 100%;
        height: auto;
      }
      .dayOfWeek {
        font-family: Arial, Helvetica, sans-serif;
        font-size: 32px;
        padding: 10px;
        height: auto;
        display: flex;
        justify-content: center;
      }
      .column-container {
        padding: 0;
        margin: 0;
        list-style: none;
        display: flex;
        flex-flow: column;
        flex-wrap: wrap;
        justify-content: center;
      }
      .row-container {
        padding: 0;
        margin: 0;
        list-style: none;
        display: flex;
        flex-flow: row;
        flex-wrap: wrap;
        justify-content: center;
      }
      .item {
        padding: 5px;
        height: auto;
        margin-top: 10px;
        display: flex;
        flex-flow: row;
        flex-wrap: wrap;
        justify-content: center;
      }
      .restaurant {
        background-color: #00a8f7;
        border-radius: 10px;
        padding: 5px;
        height: auto;
        width: auto;
        margin-left: 40px;
        margin-right: 40px;
        margin-top: 15px;
        margin-bottom: 0px;
        display: flex;
        justify-content: center;
      }
      .restaurant-name {
        font-size: 24px;
        font-family:Arial, Helvetica, sans-serif;
        color: #ffffff;
        padding: 10px;
        margin: 0px;
      }
      .restaurant-image {
        padding-top: 0px;
        margin-top: 0px;
      }
      input {
        font-family: Arial, Helvetica, sans-serif;
        font-size: 18px;
      }
      button {
        font-family: Arial, Helvetica, sans-serif;
        font-size: 18px;
      }
    </style>

    <script>
      const SEARCH_URL = '{{& searchUrl}}';

      function searchRestaurants() {
        var theme = $("#theme")[0].value;

        var xhr = new XMLHttpRequest();
        xhr.open('POST', SEARCH_URL, true);
        xhr.setRequestHeader("Content-Type", "application/json");
        xhr.send(JSON.stringify({ theme }));

        xhr.onreadystatechange = function (e) {
          if (xhr.readyState === 4 && xhr.status === 200) {
            var restaurants = JSON.parse(xhr.responseText);
            var restaurantsList = $("#restaurantsUl");
            restaurantsList.empty();

            for (var restaurant of restaurants) {
              restaurantsList.append(`
              <li class="restaurant">
                <ul class="column-container">
                    <li class="item restaurant-name">${restaurant.name}</li>
                    <li class="item restaurant-image">
                      <img src="${restaurant.image}">
                    </li>
                </ul>
              </li>
              `);
            }

          } else if (xhr.readyState === 4) {
            alert(xhr.responseText);
          }
        };
      }
    </script>
  </head>

  <body>
    <div class="fullscreenDiv">
      <ul class="column-container">
        <li class="item">
          <img id="logo" src="https://d2qt42rcwzspd6.cloudfront.net/manning/big-mouth.png">
        </li>
        <li class="item">
          <input id="theme" type="text" size="50" placeholder="enter a theme, eg. cartoon"/>
          <button onclick="searchRestaurants()">Find Restaurants</button>
        </li>
        <li>
          <div class="restaurantsDiv column-container">
            <b class="dayOfWeek">{{dayOfWeek}}</b>
            <ul id="restaurantsUl" class="row-container">
              {{#restaurants}}
              <li class="restaurant">
                <ul class="column-container">
                    <li class="item restaurant-name">{{name}}</li>
                    <li class="item restaurant-image">
                      <img src="{{image}}">
                    </li>
                </ul>
              </li>
              {{/restaurants}}
            </ul> 
          </div>
        </li>
      </ul>
  </div>
  </body>

</html>
```

This new version of `index.html` expects the URL to the search endpoint to be passed in via the `moustache` template. So we need to update the `get-index` function to pass it in.

2. Open `functions/get-index.js` and replace the exported `handler` property with the following

```javascript
module.exports.handler = async (event, context) => {
  const template = loadHtml()
  const restaurants = await getRestaurants()
  const dayOfWeek = days[new Date().getDay()]
  const html = Mustache.render(template, {
    dayOfWeek,
    restaurants,
    searchUrl: `${restaurantsApiRoot}/search`
  })
  const response = {
    statusCode: 200,
    headers: {
      'Content-Type': 'text/html; charset=UTF-8'
    },
    body: html
  }

  return response
}
```

3. Redeploy the project by running `./build.sh deploy dev`

4. Once deployed, refresh the page and enter `cartoon` in the search box and click `Find Restaurants`, and see that the results are returned

![](/images/mod02-003.png)

</p></details>