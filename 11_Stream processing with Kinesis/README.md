# Module 11: Stream processing with Kinesis

## Process events in realtime with Kinesis and Lambda

**Goal:** Implement the order flow using Kinesis

<details>
<summary><b>Add the order_events Kinesis stream</b></summary><p>

1. In the `terraform` folder, add a new file called `kinesis.tf`.

2. Copy the following into the newly created `kinesis.tf` file:

```terraform
resource "aws_kinesis_stream" "orders_stream" {
  name        = "orders_${var.stage}_${var.my_name}"
  shard_count = 1
}
```

3. Now we need to output the stream name to SSM Parameter Store. **Replace** the `terraform/parameters.tf` file with the following:

```terraform
resource "aws_ssm_parameter" "table_name" {
  name = "/big-mouth-${var.my_name}/${var.stage}/table_name"
  type = "String"
  value = "${aws_dynamodb_table.restaurants_table.name}"
}

resource "aws_ssm_parameter" "url" {
  name = "/big-mouth-${var.my_name}/${var.stage}/url"
  type = "String"
  value = "${aws_api_gateway_deployment.api.invoke_url}"
}

resource "aws_ssm_parameter" "stream_name" {
  name = "/big-mouth-${var.my_name}/${var.stage}/stream_name"
  type = "String"
  value = "${aws_kinesis_stream.orders_stream.name}"
}
```

</p></details>

<details>
<summary><b>Add place-order function</b></summary><p>

1. In the `functions` folder, add a new file called `place-order.js`.

2. Copy the following into the newly created `place-order.js` file:

```javascript
const _          = require('lodash')
const AWS        = require('aws-sdk')
const kinesis    = new AWS.Kinesis()
const chance     = require('chance').Chance()
const streamName = process.env.order_events_stream

module.exports.handler = async (event, context) => {
  const restaurantName = JSON.parse(event.body).restaurantName

  const orderId = chance.guid()
  console.log(`placing order ID [${orderId}] to [${restaurantName}]`)

  const data = {
    orderId,
    restaurantName,
    eventType: 'order_placed'
  }

  const req = {
    Data: JSON.stringify(data), // the SDK would base64 encode this for us
    PartitionKey: orderId,
    StreamName: streamName
  }

  await kinesis.putRecord(req).promise()

  console.log(`published 'order_placed' event into Kinesis`)

  const response = {
    statusCode: 200,
    body: JSON.stringify({ orderId })
  }

  return response
}
```

3. Install `chance` as a dependency with `npm install --save chance`.

</p></details>

<details>
<summary><b>Add the Terraform scripts for place-order</b></summary><p>

1. In the `terraform` folder, add a file called `place-order.tf`.

2. Copy the following into `terraform/place-order.tf`

```terraform
resource "aws_lambda_function" "place_order" {
  function_name = "${local.function_prefix}-place-order"

  s3_bucket = "${local.deployment_bucket}"
  s3_key    = "${local.deployment_key}"

  handler = "functions/place-order.handler"
  runtime = "nodejs8.10"

  role = "${aws_iam_role.place_order_lambda_role.arn}"

  environment {
    variables = {
      order_events_stream = "${aws_kinesis_stream.orders_stream.name}"
    }
  }
}

# IAM role which dictates what other AWS services the hello function can access
resource "aws_iam_role" "place_order_lambda_role" {
  name = "${local.function_prefix}-place-order-role"

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

resource "aws_iam_role_policy_attachment" "place_order_lambda_role_policy" {
  role       = "${aws_iam_role.place_order_lambda_role.name}"
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole"
}

resource "aws_iam_policy" "place_order_lambda_kinesis_policy" {
  name = "place_order_kinesis"
  path = "/"
  policy = <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "kinesis:PutRecord",
      "Resource": "${aws_kinesis_stream.orders_stream.arn}"
    }
  ]
}
EOF
}

resource "aws_iam_role_policy_attachment" "place_order_lambda_kinesis_policy" {
  role       = "${aws_iam_role.place_order_lambda_role.name}"
  policy_arn = "${aws_iam_policy.place_order_lambda_kinesis_policy.arn}"
}
```

Notice that this new function references a Kinesis stream, whose name will be parameterised in SSM parameter store.

3. Open `terraform/apigateway.tf`, and **add** the following to the end of the file

```terraform
# PLACE-ORDER
resource "aws_api_gateway_resource" "place_order" {
  rest_api_id = "${aws_api_gateway_rest_api.api.id}"
  parent_id   = "${aws_api_gateway_rest_api.api.root_resource_id}"
  path_part   = "orders"
}

resource "aws_api_gateway_method" "place_order_post" {
  rest_api_id   = "${aws_api_gateway_rest_api.api.id}"
  resource_id   = "${aws_api_gateway_resource.place_order.id}"
  http_method   = "POST"
  authorization = "NONE"
}

resource "aws_api_gateway_integration" "place_order_lambda" {
  rest_api_id = "${aws_api_gateway_rest_api.api.id}"
  resource_id = "${aws_api_gateway_method.place_order_post.resource_id}"
  http_method = "${aws_api_gateway_method.place_order_post.http_method}"

  integration_http_method = "POST"
  type                    = "AWS_PROXY"
  uri                     = "${aws_lambda_function.place_order.invoke_arn}"
}

resource "aws_lambda_permission" "apigw_place_order" {
  statement_id  = "AllowAPIGatewayInvoke"
  action        = "lambda:InvokeFunction"
  function_name = "${aws_lambda_function.place_order.arn}"
  principal     = "apigateway.amazonaws.com"

  source_arn = "${aws_api_gateway_deployment.api.execution_arn}/*/*"
}
```

4. Staying in the `terraform/apigateway.tf` file, look for the resource `aws_api_gateway_deployment.api` and **replace** it with the following

```terraform
resource "aws_api_gateway_deployment" "api" {
  depends_on = [
    "aws_api_gateway_integration.get_index_lambda",
    "aws_api_gateway_integration.get_restaurants_lambda",
    "aws_api_gateway_integration.search_restaurants_lambda",
    "aws_api_gateway_integration.place_order_lambda"
  ]

  lifecycle {
    create_before_destroy = true
  }

  rest_api_id = "${aws_api_gateway_rest_api.api.id}"
  stage_name  = "${var.stage}"

  variables {
    deployed_at = "${timestamp()}"
  }
}
```

5. Redeploy the project by running the command `./build.sh deploy dev`

6. Once the deployment is done, curl the `/orders` endpoint. **Don't forget to change the url to the invoke URL from the Terraform output**

`curl -d '{"restaurantName":"Fangtasia"}' -H "Content-Type: application/json" -X POST https://xxx.execute-api.us-east-1.amazonaws.com/dev/orders`

and you should see a response like this

```json
{"orderId":"1c70ee8c-783c-53ea-ab91-f3c8d949f584"}
```

</p></details>

<details>
<summary><b>Add integration test for place-order function</b></summary><p>

1. Add a file `place-order.js` to `test_cases` folder

2. But, before we can write the test case, we need to get `stream_name` from SSM Parameter Store during initialization.

Open `tests/steps/init.js` and **Replace** the file with the following:

```javascript
const _ = require('lodash')
const { promisify } = require('util')
const awscred = require('awscred')
const { REGION, STAGE } = process.env
const AWS = require('aws-sdk')
AWS.config.region = REGION
const SSM = new AWS.SSM()

let initialized = false

const getParameters = async (keys) => {
  const prefix = `/big-mouth-yancui/${STAGE}/`
  const req = {
    Names: keys.map(key => `${prefix}${key}`)
  }
  const resp = await SSM.getParameters(req).promise()
  return _.reduce(resp.Parameters, function(obj, param) {
    obj[param.Name.substr(prefix.length)] = param.Value
    return obj
   }, {})
}

const init = async () => {
  if (initialized) {
    return
  }

  const params = await getParameters([
    'table_name',
    'url',
    'stream_name',
  ])

  console.log('SSM params loaded')

  process.env.TEST_ROOT = params.url
  process.env.restaurants_api = `${params.url}/restaurants`
  process.env.restaurants_table = params.table_name
  process.env.order_events_stream = params.stream_name
  process.env.AWS_REGION = REGION
  
  const { credentials } = await promisify(awscred.load)()
  
  process.env.AWS_ACCESS_KEY_ID = credentials.accessKeyId
  process.env.AWS_SECRET_ACCESS_KEY = credentials.secretAccessKey

  if (credentials.sessionToken) {
    process.env.AWS_SESSION_TOKEN = credentials.sessionToken
  }

  console.log('AWS credential loaded')

  initialized = true
}

module.exports = {
  init
}
```

3. It's rather difficult to verify that a message was indeed published to Kinesis, and not to mention that has unwanted side-effect of triggering any downstream functions too (during testing). So instead, we'll use mocks for this test case.

Install `mock-aws` as dev dependency

`npm install --save-dev mock-aws`

4. Now copy the following into the newly created `tests/test_cases/place-order.js` module

```javascript
const { expect } = require('chai')
const when = require('../steps/when')
const { init } = require('../steps/init')
const AWS = require('mock-aws')

describe(`When we invoke the POST /orders endpoint`, () => {
  let isEventPublished = false
  let resp

  before(async () => {
    await init()

    AWS.mock('Kinesis', 'putRecord', (req) => {
      isEventPublished = 
        req.StreamName === process.env.order_events_stream &&
        JSON.parse(req.Data).eventType === 'order_placed'

      return {
        promise: async () => {}
      }
    })

    resp = await when.we_invoke_place_order('Fangtasia')
  })

  after(() => AWS.restore('Kinesis', 'putRecord'))

  it(`Should return 200`, async () => {
    expect(resp.statusCode).to.equal(200)
  })

  it(`Should publish a message to Kinesis stream`, async () => {
    expect(isEventPublished).to.be.true
  })
})
```

This test cases depends on a new when step - `when.we_invoke_place_order`, so let's add that.

5. **Replace** the file `tests/steps/when.js` with the following, which adds a `we_invoke_place_order` function

```javascript
const APP_ROOT = '../../'
const _ = require('lodash')
const aws4 = require('aws4')
const URL = require('url')
const http = require('superagent-promise')(require('superagent'), Promise)
const mode = process.env.TEST_MODE

const respondFrom = async (httpRes) => {
  const contentType = _.get(httpRes, 'headers.content-type', 'application/json')
  const body = 
    contentType === 'application/json'
      ? httpRes.body
      : httpRes.text

  return { 
    statusCode: httpRes.status,
    body: body,
    headers: httpRes.headers
  }
}

const signHttpRequest = (url, httpReq) => {
  const urlData = URL.parse(url)
  const opts = {
    host: urlData.hostname, 
    path: urlData.pathname
  }

  aws4.sign(opts)

  httpReq
    .set('Host', opts.headers['Host'])
    .set('X-Amz-Date', opts.headers['X-Amz-Date'])
    .set('Authorization', opts.headers['Authorization'])

  if (opts.headers['X-Amz-Security-Token']) {
    httpReq.set('X-Amz-Security-Token', opts.headers['X-Amz-Security-Token'])
  }
}

const viaHttp = async (relPath, method, opts) => {
  const root = process.env.TEST_ROOT
  const url = `${root}/${relPath}`
  console.log(`invoking via HTTP ${method} ${url}`)

  try {
    const httpReq = http(method, url)

    const body = _.get(opts, "body")
    if (body) {      
      httpReq.send(body)
    }

    if (_.get(opts, "iam_auth", false) === true) {
      signHttpRequest(url, httpReq)
    }

    const authHeader = _.get(opts, "auth")
    if (authHeader) {
      httpReq.set('Authorization', authHeader)
    }

    const res = await httpReq
    return respondFrom(res)
  } catch (err) {
    if (err.status) {
      return {
        statusCode: err.status,
        headers: err.response.headers
      }
    } else {
      throw err
    }
  }
}

const viaHandler = async (event, functionName) => {
  const handler = require(`${APP_ROOT}/functions/${functionName}`).handler

  const context = {}
  const response = await handler(event, context)
  const contentType = _.get(response, 'headers.content-type', 'application/json');
  if (response.body && contentType === 'application/json') {
    response.body = JSON.parse(response.body);
  }
  return response
}

const we_invoke_get_index = async () => {
  const res = 
    mode === 'handler' 
      ? await viaHandler({}, 'get-index')
      : await viaHttp('', 'GET')

  return res
}

const we_invoke_get_restaurants = async () => {
  const res =
    mode === 'handler' 
      ? await viaHandler({}, 'get-restaurants')
      : await viaHttp('restaurants', 'GET', { iam_auth: true })

  return res
}

const we_invoke_search_restaurants = theme => {
  let event = {
    body: JSON.stringify({ theme })
  }
  
  const res = 
    mode === 'handler'
      ? viaHandler(event, 'search-restaurants')
      : viaHttp('restaurants/search', 'POST', event)

  return res
}

const we_invoke_place_order = async (restaurantName) => {
  const body = JSON.stringify({ restaurantName })
  const res = 
    mode === 'handler'
      ? await viaHandler({ body }, 'place-order')
      : await viaHttp('orders', 'POST', { body })
  
  return res
}

module.exports = {
  we_invoke_get_index,
  we_invoke_get_restaurants,
  we_invoke_search_restaurants,
  we_invoke_place_order,
}
```

6. Run integration tests

`STAGE=dev REGION=us-east-1 npm run test`

and see that all 5 tests are passing

```
  When we invoke the GET / endpoint
SSM params loaded
AWS credential loaded
loading index.html...
loaded
    ✓ Should return the index page with 8 restaurants (926ms)

  When we invoke the GET /restaurants endpoint
    ✓ Should return an array of 8 restaurants (2265ms)

  When we invoke the POST /orders endpoint
placing order ID [a94cd32e-b9e3-5a3d-aa4d-2d146f7c6103] to [Fangtasia]
published 'order_placed' event into Kinesis
    ✓ Should return 200
    ✓ Should publish a message to Kinesis stream

  When we invoke the POST /restaurants/search endpoint with theme 'cartoon'
    ✓ Should return an array of 4 restaurants (298ms)


  5 passing (4s)
```

</p></details>

<details>
<summary><b>Add acceptance test for place-order function</b></summary><p>

When executing the deployed `place-order` function via API Gateway, the function would publish an `order_placed` event to the real Kinesis stream.

To verify that the event is published as expected, you have some options:

* If events are streamed and backed up in S3 (e.g. via Kinesis Firehose, so all events are recorded in a persistent storage), then you can poll S3 for new events. However, this approach can be time-consuming depending on the Firehose configuration, if data are batched in 5 mins intervals then this approach becomes infeasible.

* If events are streamed to another BI platform, such as Google Big Query, in real time, then that is a far better option - to query Big Query for the expected event.

* You can use the AWS SDK to fetch Kinesis records with `kinesis.getRecords`, but this is clumsy as it's a multi-step process that requires you to describe shards and get shard iterator first, and when there are more than 1 shard in the stream it also becomes infeasible to keep polling every shard until you have found the expected event.

For this workshop, we'll take a short-cut and only validate Kinesis was called when executing as an integration test.

1. Modify `test_cases/place-order.js` so the test case no longer validates Kinesis event is published when running as an acceptance test

```javascript
it(`Should return 200`, async () => {
  expect(resp.statusCode).to.equal(200)
})

if (process.env.TEST_MODE === 'handler') {
  it(`Should publish a message to Kinesis stream`, async () => {
    expect(isEventPublished).to.be.true
  })
}
```

2. Run acceptance test

`STAGE=dev REGION=us-east-1 npm run acceptance`

and see that all 4 tests are passing

```
  When we invoke the GET / endpoint
SSM params loaded
AWS credential loaded
invoking via HTTP GET https://sr73zpk0el.execute-api.us-east-1.amazonaws.com/dev/
    ✓ Should return the index page with 8 restaurants (751ms)

  When we invoke the GET /restaurants endpoint
invoking via HTTP GET https://sr73zpk0el.execute-api.us-east-1.amazonaws.com/dev/restaurants
    ✓ Should return an array of 8 restaurants (380ms)

  When we invoke the POST /orders endpoint
invoking via HTTP POST https://sr73zpk0el.execute-api.us-east-1.amazonaws.com/dev/orders
    ✓ Should return 200

  When we invoke the POST /restaurants/search endpoint with theme 'cartoon'
invoking via HTTP POST https://sr73zpk0el.execute-api.us-east-1.amazonaws.com/dev/restaurants/search
    ✓ Should return an array of 4 restaurants (511ms)


  4 passing (3s)
```

</p></details>

<details>
<summary><b>Update web client to support placing order</b></summary><p>

1. Modify `static/index.html` to the following

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
        padding: 5px;
        margin: 5px;
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
      .row-container-left {
        list-style: none;
        display: flex;
        flex-flow: row;
        justify-content: flex-start;
      }
      .menu-text {
        font-family: Arial, Helvetica, sans-serif;
        font-size: 24px;
        font-weight: bold;
        color: white;
      }
      .text-trail-space {
        margin-right: 10px;
      }
      .hidden {
        display: none;
      }

      lable, button, input {
        display:block;
        font-family: Arial, Helvetica, sans-serif;
        font-size: 18px;
      }
      
      fieldset { 
        padding:0; 
        border:0; 
        margin-top:25px; 
      }

    </style>

    <script>
      const SEARCH_URL = '{{& searchUrl}}';
      const PLACE_ORDER_URL = '{{& placeOrderUrl}}';

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
                <ul class="column-container" onclick='placeOrder("${restaurant.name}")'>
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

      function placeOrder(restaurantName) {
        var xhr = new XMLHttpRequest();
        xhr.open('POST', PLACE_ORDER_URL, true);
        xhr.setRequestHeader("Content-Type", "application/json");
        xhr.send(JSON.stringify({ restaurantName }));

        xhr.onreadystatechange = function (e) {
          if (xhr.readyState === 4 && xhr.status === 200) {
            alert("your order has been placed, we'll let you know once it's been accepted by the restaurant!");
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
          <input id="theme" type="text" size="50" placeholder="enter a theme, eg. rick and morty"/>
          <button onclick="searchRestaurants()">Find Restaurants</button>
        </li>
        <li>
          <div class="restaurantsDiv column-container">
            <b class="dayOfWeek">{{dayOfWeek}}</b>
            <ul id="restaurantsUl" class="row-container">
              {{#restaurants}}
              <li class="restaurant">
                <ul class="column-container" onclick='placeOrder("{{name}}")'>
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

2. The `get-index` function also needs to change and pass the orders API to the template. **Replace** the file `functions/get-index.js` with the following

```javascript
const fs = require("fs")
const Mustache = require('mustache')
const http = require('superagent-promise')(require('superagent'), Promise)
const aws4 = require('aws4')
const URL = require('url')

const restaurantsApiRoot = process.env.restaurants_api
const days = ['Sunday', 'Monday', 'Tuesday', 'Wednesday', 'Thursday', 'Friday', 'Saturday']
const ordersApiRoot = process.env.orders_api

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
  const url = URL.parse(restaurantsApiRoot)
  const opts = {
    host: url.hostname,
    path: url.pathname
  }

  aws4.sign(opts)

  const httpReq = http
    .get(restaurantsApiRoot)
    .set('Host', opts.headers['Host'])
    .set('X-Amz-Date', opts.headers['X-Amz-Date'])
    .set('Authorization', opts.headers['Authorization'])

  if (opts.headers['X-Amz-Security-Token']) {
    httpReq.set('X-Amz-Security-Token', opts.headers['X-Amz-Security-Token'])
  }

  return (await httpReq).body
}

module.exports.handler = async (event, context) => {
  const template = loadHtml()
  const restaurants = await getRestaurants()
  const dayOfWeek = days[new Date().getDay()]
  const html = Mustache.render(template, {
    dayOfWeek,
    restaurants,
    searchUrl: `${restaurantsApiRoot}/search`,
    placeOrderUrl: `${ordersApiRoot}`
  })
  const response = {
    statusCode: 200,
    headers: {
      'content-type': 'text/html; charset=UTF-8'
    },
    body: html
  }

  return response
}
```

3. Open the `terraform/get-index.tf` file, look for the `aws_lambda_function.get_index` resource. Right now, it has only one environment variable.

```terraform
environment {
  variables = {
    restaurants_api = "https://${aws_api_gateway_rest_api.api.id}.execute-api.us-east-1.amazonaws.com/${var.stage}/restaurants"
  }
}
```

**Replace** this `environment` block with the following to add a second environment variable `orders_api`

```terraform
environment {
  variables = {
    restaurants_api = "https://${aws_api_gateway_rest_api.api.id}.execute-api.us-east-1.amazonaws.com/${var.stage}/restaurants",
    orders_api = "https://${aws_api_gateway_rest_api.api.id}.execute-api.us-east-1.amazonaws.com/${var.stage}/orders"
  }
}
```

4. Redeploy the project.

5. Load the landing page in the browser and click on one of the restaurants to order.

![](/images/mod11-001.png)

</p></details>

<details>
<summary><b>Add restaurant-notifications SNS topics</b></summary><p>

1. In the `terraform` folder, add a new file called `sns.tf`.

2. Copy the following into `sns.tf`:

```terraform
resource "aws_sns_topic" "restaurant_notification" {
  name = "restaurant-notificaton-${var.stage}-${var.my_name}"
}
```

3. Open `terraform/parameters.tf`, and **add** the following to the end fo the file.

```terraform
resource "aws_ssm_parameter" "restaurant_topic_name" {
  name = "/big-mouth-${var.my_name}/${var.stage}/restaurant_topic_name"
  type = "String"
  value = "${aws_sns_topic.restaurant_notification.name}"
}
```

This will create a new parameter for the topic name in SSM Parameter Store.

4. Redeploy the project, and you should see the topic in the SNS console.

![](/images/mod11-002.png)

</p></details>

<details>
<summary><b>Add notify-restaurant function</b></summary><p>

1. In the project root, add a new folder called `lib`.

2. In the `lib` folder, add a new file called `kinesis.js`.

3. Copy the following into `lib/kinesis.js`:

```javascript
function parsePayload (record) {
  const json = new Buffer(record.kinesis.data, 'base64').toString('utf8')
  return JSON.parse(json)
}

const getRecords = (event) => event.Records.map(parsePayload)

module.exports = {
  getRecords
}
```

Kinesis sends records to our function in batches (unlike other event sources such as API Gateway and SNS). And its payloads are base64 encoded. This helper module base64 decodes and JSON parses the Kinesis events for us.

4. In the `functions` folder, add a new file called `notify-restaurant.js`.

5. Copy the following into `notify-restaurant.js`:

```javascript
const _ = require('lodash')
const { getRecords } = require('../lib/kinesis')
const AWS = require('aws-sdk')
const kinesis = new AWS.Kinesis()
const sns = new AWS.SNS()

const streamName = process.env.order_events_stream
const topicArn = process.env.restaurant_notification_topic

module.exports.handler = async (event, context) => {
  const records = getRecords(event)
  const orderPlaced = records.filter(r => r.eventType === 'order_placed')

  for (let order of orderPlaced) {
    const snsReq = {
      Message: JSON.stringify(order),
      TopicArn: topicArn
    };
    await sns.publish(snsReq).promise()
    console.log(`notified restaurant [${order.restaurantName}] of order [${order.orderId}]`)

    const data = _.clone(order)
    data.eventType = 'restaurant_notified'

    const kinesisReq = {
      Data: JSON.stringify(data), // the SDK would base64 encode this for us
      PartitionKey: order.orderId,
      StreamName: streamName
    }
    await kinesis.putRecord(kinesisReq).promise()
    console.log(`published 'restaurant_notified' event to Kinesis`)
  }  
}
```

</p></details>

<details>
<summary><b>Add the Terraform scripts for notify-restaurant</b></summary><p>

1. In the `terraform` folder, add a new file called `notify-restaurant.js`.

2. Copy the following into `terraform/notify-restaurant.js`:

```terraform
resource "aws_lambda_function" "notify_restaurant" {
  function_name = "${local.function_prefix}-notify-restaurant"

  s3_bucket = "${local.deployment_bucket}"
  s3_key    = "${local.deployment_key}"

  handler = "functions/notify-restaurant.handler"
  runtime = "nodejs8.10"

  role = "${aws_iam_role.notify_restaurant_lambda_role.arn}"

  environment {
    variables = {
      order_events_stream = "${aws_kinesis_stream.orders_stream.name}",
      restaurant_notification_topic = "${aws_sns_topic.restaurant_notification.arn}"
    }
  }
}

resource "aws_iam_role" "notify_restaurant_lambda_role" {
  name = "${local.function_prefix}-notify-restaurant-role"

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

resource "aws_iam_role_policy_attachment" "notify_restaurant_lambda_role_policy" {
  role       = "${aws_iam_role.notify_restaurant_lambda_role.name}"
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole"
}

resource "aws_iam_policy" "notify_restaurant_lambda_policy" {
  name = "notify_restaurant"
  path = "/"
  policy = <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "kinesis:PutRecord",
        "kinesis:GetRecords",
        "kinesis:GetShardIterator",
        "kinesis:DescribeStream",
        "kinesis:ListStreams"
      ],
      "Resource": "${aws_kinesis_stream.orders_stream.arn}"
    },
    {
      "Effect": "Allow",
      "Action": "sns:Publish",
      "Resource": "${aws_sns_topic.restaurant_notification.arn}"
    }
  ]
}
EOF
}

resource "aws_iam_role_policy_attachment" "notify_restaurant_lambda_policy" {
  role       = "${aws_iam_role.notify_restaurant_lambda_role.name}"
  policy_arn = "${aws_iam_policy.notify_restaurant_lambda_policy.arn}"
}

resource "aws_lambda_event_source_mapping" "notify_restaurant_lambda_kinesis" {
  event_source_arn  = "${aws_kinesis_stream.orders_stream.arn}"
  function_name     = "${aws_lambda_function.notify_restaurant.arn}"
  starting_position = "LATEST"
  batch_size        = 10
}
```

3. Because we had added a new `lib` folder that needs to be included in the artefact, we need to update our build scripts to include it.

First, let's update our `build.sh` that we use locally, **replace** `build.sh` with the following:

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

  npm ci
  zip -r workshop.zip functions static node_modules lib

  MD5=$(md5 -q workshop.zip)
  aws s3 cp workshop.zip s3://ynap-production-ready-serverless-yancui/workshop/$MD5.zip
  
  cd terraform
  terraform apply --var "my_name=yancui" --var "file_name=$MD5"
else
  instruction
  exit 1
fi
```

4. Because we use a separate `build.js` in the CI/CD pipeline (remember, this is forced by the constraint of not having a custom Docker image), we also need to update it as well.

**Replace** `build.js` with the following:

```javascript
const fs = require('fs')
const archiver = require('archiver')
const md5File = require('md5-file')
const AWS = require('aws-sdk')
const S3 = new AWS.S3()

const output = fs.createWriteStream(__dirname + '/workshop.zip')
const archive = archiver('zip')

const [node, path, myName, ...rest] = process.argv
const Bucket = `ynap-production-ready-serverless-${myName}`

console.log(`deployment bucket is ${Bucket}`)

output.on('close', function () {
  console.log('deployment artefact created')

  md5File('workshop.zip', (err, md5) => {
    if (err) {
      throw err
    }

    const filename = `workshop/${md5}.zip`
    console.log(`uploading to S3 as ${filename}`)

    S3.upload({
      Bucket,
      Key: filename,
      Body: fs.createReadStream(__dirname + '/workshop.zip')
    }, (err, resp) => {
      if (err) {
        throw err
      }
      
      console.log('artefact has been uploaded to S3')
      
      fs.writeFileSync('workshop_md5.txt', md5)
    })
  })
})

archive.on('error', function(err){
  throw err
})

archive.pipe(output)

archive.directory('functions')
archive.directory('static')
archive.directory('node_modules')
archive.directory('lib')

archive.finalize()
```

5. Commit and push your changes, make sure the CICD pipeline still works.

</p></details>

<details>
<summary><b>Add integration test for notify-restaurant function</b></summary><p>

1. To test the `notify-restaurant` function, we need to know the `restaurant_topic_name` from SSM parameter store.

Open `tests/steps/init.js`, **replace** the file with the following and replace the `<PUT YOUR NAME HERE>` placeholder:

```javascript
const _ = require('lodash')
const { promisify } = require('util')
const awscred = require('awscred')
const { REGION, STAGE } = process.env
const AWS = require('aws-sdk')
AWS.config.region = REGION
const SSM = new AWS.SSM()

let initialized = false

const getParameters = async (keys) => {
  const prefix = `/big-mouth-<PUT YOUR NAME HERE>/${STAGE}/`
  const req = {
    Names: keys.map(key => `${prefix}${key}`)
  }
  const resp = await SSM.getParameters(req).promise()
  return _.reduce(resp.Parameters, function(obj, param) {
    obj[param.Name.substr(prefix.length)] = param.Value
    return obj
   }, {})
}

const init = async () => {
  if (initialized) {
    return
  }

  const params = await getParameters([
    'table_name',
    'url',
    'stream_name',
    'restaurant_topic_name',
  ])

  console.log('SSM params loaded')

  process.env.TEST_ROOT = params.url
  process.env.restaurants_api = `${params.url}/restaurants`
  process.env.restaurants_table = params.table_name
  process.env.order_events_stream = params.stream_name
  process.env.restaurant_notification_topic = params.restaurant_topic_name
  process.env.AWS_REGION = REGION
  
  const { credentials } = await promisify(awscred.load)()
  
  process.env.AWS_ACCESS_KEY_ID = credentials.accessKeyId
  process.env.AWS_SECRET_ACCESS_KEY = credentials.secretAccessKey

  if (credentials.sessionToken) {
    process.env.AWS_SESSION_TOKEN = credentials.sessionToken
  }

  console.log('AWS credential loaded')

  initialized = true
}

module.exports = {
  init
}
```

2. Open `tests/steps/when.js`, and add a `we_invoke_notify_restaurant` function along with its helper `toKinesisEvent` function:

```javascript
const toKinesisEvent = events => {
  const records = events.map(event => {
    const data = Buffer.from(JSON.stringify(event)).toString('base64')
    return {
      "eventID": "shardId-000000000000:49545115243490985018280067714973144582180062593244200961",
      "eventVersion": "1.0",
      "kinesis": {
        "approximateArrivalTimestamp": 1428537600,
        "partitionKey": "partitionKey-3",
        "data": data,
        "kinesisSchemaVersion": "1.0",
        "sequenceNumber": "49545115243490985018280067714973144582180062593244200961"
      },
      "invokeIdentityArn": "arn:aws:iam::EXAMPLE",
      "eventName": "aws:kinesis:record",
      "eventSourceARN": "arn:aws:kinesis:EXAMPLE",
      "eventSource": "aws:kinesis",
      "awsRegion": "us-east-1"
    }
  })

  return {
    Records: records
  }
}

const we_invoke_notify_restaurant = async (...events) => {
  if (mode === 'handler') {
    await viaHandler(toKinesisEvent(events), 'notify-restaurant')
  } else {
    throw new Error('not supported')
  }
}
```

and don't forget to export the `we_invoke_notify_restaurant` by adding it to `module.exports`.

Your `module.exports` should look like this:

```javascript
module.exports = {
  we_invoke_get_index,
  we_invoke_get_restaurants,
  we_invoke_search_restaurants,
  we_invoke_place_order,
  we_invoke_notify_restaurant,
}
```

3. In the `tests/test_cases` folder, add a new file called `notify-restaurant.js`.

4. Copy the following into `tests/test_cases/notify-restaurant.js`:

```javascript
const { expect } = require('chai')
const { init } = require('../steps/init')
const when = require('../steps/when')
const AWS = require('mock-aws')
const chance = require('chance').Chance()

describe(`When we invoke the notify-restaurant function`, () => {
  let isEventPublished = false
  let isNotified = false

  before(async () => {
    await init()

    AWS.mock('Kinesis', 'putRecord', (req) => {
      isEventPublished = 
        req.StreamName === process.env.order_events_stream &&
        JSON.parse(req.Data).eventType === 'restaurant_notified'

      return {
        promise: async () => {}
      }
    })

    AWS.mock('SNS', 'publish', (req) => {
      isNotified = 
        req.TopicArn === process.env.restaurant_notification_topic &&
        JSON.parse(req.Message).eventType === 'order_placed'

      return {
        promise: async () => {}
      }
    })

    const event = {
      orderId: chance.guid(),
      userEmail: chance.email(),
      restaurantName: 'Fangtasia',
      eventType: 'order_placed'
    }
    await when.we_invoke_notify_restaurant(event)
  })

  after(() => {
    AWS.restore('Kinesis', 'putRecord')
    AWS.restore('SNS', 'publish')
  })

  it(`Should publish message to SNS`, async () => {
    expect(isNotified).to.be.true
  })

  it(`Should publish event to Kinesis`, async () => {
    expect(isEventPublished).to.be.true
  })
})
```

5. Run integration tests

`STAGE=dev REGION=us-east-1 npm run test`

and see that the new test is failing:

```
1) When we invoke the notify-restaurant function
      "before all" hook for "Should publish message to SNS":
    TypeError: Cannot read property 'body' of undefined
    at viaHandler (tests/steps/when.js:83:16)
```

This is because our `notify-restaurant` doesn't return any response, because it doesn't need to.

6. Open `tests/steps/when.js`, we need to update the `viaHandler` function to handle this. **Replace** the `viaHandler` function with the following:

```javascript
const viaHandler = async (event, functionName) => {
  const handler = require(`${APP_ROOT}/functions/${functionName}`).handler
  console.log(`invoking via handler function ${functionName}`)

  const context = {}
  const response = await handler(event, context)
  const contentType = _.get(response, 'headers.content-type', 'application/json');
  if (_.get(response, 'body') && contentType === 'application/json') {
    response.body = JSON.parse(response.body);
  }
  return response
}
```

7. Rerun integration tests

`STAGE=dev REGION=us-east-1 npm run test`

and see that all tests are passing now

```
  When we invoke the GET / endpoint
SSM params loaded
AWS credential loaded
invoking via handler function get-index
loading index.html...
loaded
    ✓ Should return the index page with 8 restaurants (367ms)

  When we invoke the GET /restaurants endpoint
invoking via handler function get-restaurants
    ✓ Should return an array of 8 restaurants (532ms)

  When we invoke the notify-restaurant function
invoking via handler function notify-restaurant
notified restaurant [Fangtasia] of order [5e8f5bd3-234d-582c-b138-73d31afbb3fe]
published 'restaurant_notified' event to Kinesis
    ✓ Should publish message to SNS
    ✓ Should publish event to Kinesis

  Given an authenticated user
[test-Lina-Catarzi-0%sG^VPl] - user is created
[test-Lina-Catarzi-0%sG^VPl] - initialised auth flow
[test-Lina-Catarzi-0%sG^VPl] - responded to auth challenge
    When we invoke the POST /orders endpoint
invoking via handler function place-order
placing order ID [6ebe25d7-9d2d-5549-89c3-24b22766440f] to [Fangtasia] for user [test-Lina-Catarzi-0%sG^VPl@test.com]
published 'order_placed' event into Kinesis
      ✓ Should return 200
      ✓ Should publish a message to Kinesis stream
[test-Lina-Catarzi-0%sG^VPl] - user deleted

  Given an authenticated user
[test-Bradley-Kuiper-rJtlGV5T] - user is created
[test-Bradley-Kuiper-rJtlGV5T] - initialised auth flow
[test-Bradley-Kuiper-rJtlGV5T] - responded to auth challenge
    When we invoke the POST /restaurants/search endpoint with theme 'cartoon'
invoking via handler function search-restaurants
      ✓ Should return an array of 4 restaurants (253ms)
[test-Bradley-Kuiper-rJtlGV5T] - user deleted


  7 passing (5s)
```

</p></details>

<details>
<summary><b>Acceptance test for notify-restaurant function</b></summary><p>

We can publish a Kinesis event via the AWS SDK to execute the deployed `notify-restaurant` function. Since this function publishes to both SNS and Kinesis, we have the same conumdrum in verifying that it's producing the expected side-effects as the `place-order` function.

The same options we discussed earlier apply here, with regards to verifying the `restaurant_notified` event is published to Kinesis.

But how do we verify that SNS message has notified? And what if we had used SES as we intended initially?

To verify that an email was received, we could subscribe a test email address to the SNS topic (or whitelist it in the case of SES). Then we can programmatically (e.g. Gmail has an API which we can use to read our emails) check our inbox and see if we had received the notification email.

For this workshop, we'll take a short-cut and skip the test altogether.

1. We need to modify the `notify-restaurant` tests so they only run as integration test.

Open `tests/test_cases/notify-restaurant.js`, and wrap the whole `describe` block in a `if` block, like the following:

```javascript
if (process.env.TEST_MODE === 'handler') {
  describe(`When we invoke the notify-restaurant function`, () => {
    ...
  })
}
```

So your file should look like this in the end:

```javascript
const { expect } = require('chai')
const { init } = require('../steps/init')
const when = require('../steps/when')
const AWS = require('mock-aws')
const chance = require('chance').Chance()

if (process.env.TEST_MODE === 'handler') {
  describe(`When we invoke the notify-restaurant function`, () => {
    let isEventPublished = false
    let isNotified = false

    before(async () => {
      await init()

      AWS.mock('Kinesis', 'putRecord', (req) => {
        isEventPublished = 
          req.StreamName === process.env.order_events_stream &&
          JSON.parse(req.Data).eventType === 'restaurant_notified'

        return {
          promise: async () => {}
        }
      })

      AWS.mock('SNS', 'publish', (req) => {
        isNotified = 
          req.TopicArn === process.env.restaurant_notification_topic &&
          JSON.parse(req.Message).eventType === 'order_placed'

        return {
          promise: async () => {}
        }
      })

      const event = {
        orderId: chance.guid(),
        restaurantName: 'Fangtasia',
        eventType: 'order_placed'
      }
      await when.we_invoke_notify_restaurant(event)
    })

    after(() => {
      AWS.restore('Kinesis', 'putRecord')
      AWS.restore('SNS', 'publish')
    })

    it(`Should publish message to SNS`, async () => {
      expect(isNotified).to.be.true
    })

    it(`Should publish event to Kinesis`, async () => {
      expect(isEventPublished).to.be.true
    })
  })
}
```

2. Run acceptance test

`STAGE=dev REGION=us-east-1 npm run acceptance`

and see that all 4 tests are passing

```
  When we invoke the GET / endpoint
SSM params loaded
AWS credential loaded
invoking via HTTP GET https://sr73zpk0el.execute-api.us-east-1.amazonaws.com/dev/
    ✓ Should return the index page with 8 restaurants (644ms)

  When we invoke the GET /restaurants endpoint
invoking via HTTP GET https://sr73zpk0el.execute-api.us-east-1.amazonaws.com/dev/restaurants
    ✓ Should return an array of 8 restaurants (171ms)

  When we invoke the POST /orders endpoint
invoking via HTTP POST https://sr73zpk0el.execute-api.us-east-1.amazonaws.com/dev/orders
    ✓ Should return 200

  When we invoke the POST /restaurants/search endpoint with theme 'cartoon'
invoking via HTTP POST https://sr73zpk0el.execute-api.us-east-1.amazonaws.com/dev/restaurants/search
    ✓ Should return an array of 4 restaurants (512ms)


  4 passing (2s)
```

</p></details>

<details>
<summary><b>Subscribe yourself to the SNS topics</b></summary><p>

1. Go to SNS console

2. Find your restaurant notification topic

![](/images/mod11-002.png)

3. Click `Create subscription`

4. Choose `Email` for `Protocol` and enter your email

5. Click `Create subscription`

![](/images/mod11-003.png)

6. Check your email, and look for an email from `AWS Notification - Subscription Confirmation`

![](/images/mod11-004.png)

7. Click the `Confirm subscription` link

![](/images/mod11-005.png)

Ok, now you have subscribed yourself to the SNS topics, so we can see how the messages look when they have been processed.

8. Great! Reload the landing page in the browser and try to place a few orders.

You should receive emails from SNS like this:

![](/images/mod11-006.png)

</p></details>