# Module 16: X-Ray

## Collect invocation traces in X-Ray

<details>
<summary><b>Enable X-Ray in API Gateway</b></summary>
<p>

1. Open `terraform/apigateway.tf` and look for the resource `aws_api_gateway_stage.stage`. **Replace** it with the following to set `xray_tracing_enabled` to `true`:

```terraform
resource "aws_api_gateway_stage" "stage" {
  stage_name           = "${var.stage}"
  rest_api_id          = "${aws_api_gateway_rest_api.api.id}"
  deployment_id        = "${aws_api_gateway_deployment.api.id}"
  xray_tracing_enabled = true
}
```

</p></details>

<details>
<summary><b>Enable X-Ray with Lambda</b></summary><p>

1. Open `terraform/get-index.tf` and look for the resource `aws_iam_policy.get_index_lambda_apigateway_policy`. We need to give the function permissions to write to X-Ray.

**Replace** the resource with the following:

```terraform
resource "aws_iam_policy" "get_index_lambda_apigateway_policy" {
  name = "apigateway_execute"
  path = "/"
  policy = <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "execute-api:Invoke",
      "Resource": "arn:aws:execute-api:us-east-1:${data.aws_caller_identity.current.account_id}:*/*/GET/restaurants"
    },
    {
      "Effect": "Allow",
      "Action": [
        "xray:PutTraceSegments",
        "xray:PutTelemetryRecords"
      ],
      "Resource": "*"
    }
  ]
}
EOF
}
```

2. Open `terraform/get-restaurants.tf` and look for the resource `aws_iam_policy.get_restaurants_lambda_dynamodb_policy`. We need to give the function permissions to write to X-Ray.

**Replace** the resource with the following:

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
    },
    {
      "Effect": "Allow",
      "Action": [
        "xray:PutTraceSegments",
        "xray:PutTelemetryRecords"
      ],
      "Resource": "*"
    }
  ]
}
EOF
}
```

3. Open `terraform/notify-restaurant.tf` and look for the resource `aws_iam_policy.notify_restaurant_lambda_policy`. We need to give the function permissions to write to X-Ray.

**Replace** the resource with the following:

```terraform
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
    },
    {
      "Effect": "Allow",
      "Action": [
        "xray:PutTraceSegments",
        "xray:PutTelemetryRecords"
      ],
      "Resource": "*"
    }
  ]
}
EOF
}
```

4. Open `terraform/place-order.tf` and look for the resource `aws_iam_policy.place_order_lambda_kinesis_policy`. We need to give the function permissions to write to X-Ray.

**Replace** the resource with the following:

```terraform
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
    },
    {
      "Effect": "Allow",
      "Action": [
        "xray:PutTraceSegments",
        "xray:PutTelemetryRecords"
      ],
      "Resource": "*"
    }
  ]
}
EOF
}
```

5. Open `terraform/search-restaurants.tf` and look for the resource `aws_iam_policy.search_restaurants_lambda_dynamodb_policy`. We need to give the function permissions to write to X-Ray.

**Replace** the resource with the following:

```terraform
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
    },
    {
      "Effect": "Allow",
      "Action": [
        "xray:PutTraceSegments",
        "xray:PutTelemetryRecords"
      ],
      "Resource": "*"
    }
  ]
}
EOF
}
```

6. Deploy the project.

7. After the deployment is done, load up the landing page, place an order. Then head to the X-Ray console and see what you get.

![](/images/mod16-001.png)

![](/images/mod16-002.png)

![](/images/mod16-003.png)

</p></details>

<details>
<summary><b>Instrumenting AWSSDK</b></summary><p>

At the moment we're not getting a lot of value out of X-Ray. We can get much more information about what's happening in our code if we instrument the various steps.

To begin with, we can instrument the AWS SDK so we track how long calls to DynamoDB and SNS takes in the traces.

1. From the project root, install `aws-xray-sdk-core` as dependency

`npm install --save aws-xray-sdk-core`

2. Open `functions/get-restaurants.js` and **replace** the line `const AWS = require('aws-sdk')` with the following:

```javascript
const AWSXRay = require('aws-xray-sdk-core')
const AWS = AWSXRay.captureAWS(require('aws-sdk'))
```

3. Repeat step 2 for

* `functions/notify-restaurant.js`
* `functions/place-order.js`
* `functions/search-restaurants.js`

4. Deploy the project

5. After the deployment is done, load up the landing page again and place an order. Then head to the X-Ray console and see what you get now.

![](/images/mod16-004.png)

![](/images/mod16-005.png)

![](/images/mod16-006.png)

</p></details>

<details>
<summary><b>Instrumenting HTTP calls</b></summary><p>

We can get even more value if we could see the traces for `get-index` function and the corresponding trace for the `get-restaurants` function in one screen.

![](/images/mod16-007.png)

Then it's proper distributed tracing! It's not very helpful if you're restricted to only what happens inside one function.

Fortunately, you can instrument the built-in `https` module with the X-Ray SDK, unfortunately, you have to use it instead of other HTTP clients..

1. Open `functions/get-index.js` and **replace** the line:

```javascript
const http = require('superagent-promise')(require('superagent'), Promise)
```

with the following:

```javascript
const AWSXRay = require('aws-xray-sdk-core')
const https = AWSXRay.captureHTTPs(require('https'))
```

2. Staying in `functions/get-index`. **Replace** the `getRestaurants` function with the following:

```javascript
const getRestaurants = () => {
  const url = URL.parse(restaurantsApiRoot)
  const opts = {
    host: url.hostname,
    path: url.pathname
  }

  aws4.sign(opts)

  return new Promise((resolve, reject) => {
    const options = {
      hostname: url.hostname,
      port: 443,
      path: url.pathname,
      method: 'GET',
      headers: opts.headers
    }

    const req = https.request(options, res => {
      res.on('data', buffer => {
        const body = buffer.toString('utf8')
        resolve(JSON.parse(body))
      })
    })

    req.on('error', err => reject(err))

    req.end()
  })
}
```

It uses the `https` module to make the HTTP request to the `/restaurants` endpoint instead.

3. Deploy the project.

4. After the deployment is done, load up the landing page again and place an order. Then head to the X-Ray console and now you can see the traces for `get-index` and `get-restaurants` function in one place.

![](/images/mod16-009.png)

</p></details>

<details>
<summary><b>Fix broken tests</b></summary><p>

If you run the integration tests:

`STAGE=dev REGION=us-east-1 npm run test`

then you'll see all the tests are broken...

This is because the X-Ray SDK expects some context and root segment to be provided by the Lambda service's runtime. Which we won't have when running locally.

1. Open `tests/steps/init.js`, we need to add this line along with the other environment variables

```javascript
process.env.AWS_XRAY_CONTEXT_MISSING = 'LOG_ERROR'
```

This stops the X-Ray SDK from erroring when it doesn't find the context

Rerun the integration tests, and the tests are still broken, with errors like this

```
  2) When we invoke the GET /restaurants endpoint
       Should return an array of 8 restaurants:
     TypeError: Service.prototype.customizeRequests is not a function
      at Object.captureAWS (node_modules/aws-xray-sdk-core/lib/patchers/aws_p.js:37:25)
      at Object.<anonymous> (functions/get-restaurants.js:5:21)
      at require (internal/module.js:11:18)
      at viaHandler (tests/steps/when.js:79:34)
      at Object.we_invoke_get_restaurants (tests/steps/when.js:103:15)
      at Context.it (tests/test_cases/get-restaurants.js:9:26)
```

2. The best bad way to work around this (except just giving up on the X-Ray SDK altogether) is to not use it when executing locally. When the function is running in the Lambda execution environment, it has a number of environment variables, including one called `LAMBDA_RUNTIME_DIR`

![](/images/mod16-010.png)

Go back to `functions/get-index.js` and **replace** the line

```javascript
const https = AWSXRay.captureHTTPs(require('https'))
```

with the following:

```javascript
const https = process.env.LAMBDA_RUNTIME_DIR
  ? AWSXRay.captureHTTPs(require('https'))
  : require('https')
```

3. Similarly, modify `functions/get-restaurants.js` and **replace** this line

```javascript
const AWS = AWSXRay.captureAWS(require('aws-sdk'))
```

with the following:

```javascript
const AWS = process.env.LAMBDA_RUNTIME_DIR
  ? AWSXRay.captureAWS(require('aws-sdk'))
  : require('aws-sdk')
```

4. Repeat step 3 with

* `functions/notify-restaurant.js`
* `functions/place-order.js`
* `functions/search-restaurants.js`

5. Rerun the integration tests

`STAGE=dev REGION=us-east-1 npm run test`

and see that all the tests should be passing now

</p></details>