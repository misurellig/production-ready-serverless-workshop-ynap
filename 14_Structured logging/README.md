# Module 14: Structured logging

## Apply structured logging to the project

<details>
<summary><b>Using a simple logger</b></summary><p>

I built a suite of tools to help folks build production-ready serverless applications while I was at DAZN. It's now open source: [dazn-lambda-powertools](https://github.com/getndazn/dazn-lambda-powertools).

One of the tools available is a very simple logger that supports structured logging (amongst other things).

So, first, let's install the logger for our project.

1. At the project root, run the command `npm install --save @perform/lambda-powertools-logger` to install the logger.

Now we need to change all the places where we're using `console.log`.

2. Open `functions/get-index.js`, and replace `console.log` with use of the logger.

**Replace** `functions/get-index.js` with the following:

```javascript
const fs = require("fs")
const Mustache = require('mustache')
const http = require('superagent-promise')(require('superagent'), Promise)
const aws4 = require('aws4')
const URL = require('url')
const Log = require('@perform/lambda-powertools-logger')

const restaurantsApiRoot = process.env.restaurants_api
const days = ['Sunday', 'Monday', 'Tuesday', 'Wednesday', 'Thursday', 'Friday', 'Saturday']
const ordersApiRoot = process.env.orders_api

let html

function loadHtml () {
  if (!html) {
    Log.debug('loading index.html...')
    html = fs.readFileSync('static/index.html', 'utf-8')
    Log.debug('loaded')
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
  Log.debug('received restaurants', { count: restaurants.length })

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

3. Open `functions/place-order.js`, and replace `console.log` with use of the logger.

**Replace** `functions/place-order.js` with the following:

```javascript
const _ = require('lodash')
const AWS = require('aws-sdk')
const kinesis = new AWS.Kinesis()
const chance = require('chance').Chance()
const streamName = process.env.order_events_stream
const Log = require('@perform/lambda-powertools-logger')

module.exports.handler = async (event, context) => {
  const restaurantName = JSON.parse(event.body).restaurantName

  const orderId = chance.guid()
  Log.debug('placing order', { orderId, restaurantName })

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

  Log.debug(`published 'order_placed' event into Kinesis`)

  const response = {
    statusCode: 200,
    body: JSON.stringify({ orderId })
  }

  return response
}
```

4. Open `functions/notify-restaurant.js`, and replace `console.log` with use of the logger.

**Replace** `functions/notify-restaurant.js` with the following:

```javascript
const _ = require('lodash')
const { getRecords } = require('../lib/kinesis')
const AWS = require('aws-sdk')
const kinesis = new AWS.Kinesis()
const sns = new AWS.SNS()
const Log = require('@perform/lambda-powertools-logger')

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
    Log.debug(
      'notified restaurant of order', 
      { restaurantName: order.restaurantName, orderId: order.orderId})

    const data = _.clone(order)
    data.eventType = 'restaurant_notified'

    const kinesisReq = {
      Data: JSON.stringify(data), // the SDK would base64 encode this for us
      PartitionKey: order.orderId,
      StreamName: streamName
    }
    await kinesis.putRecord(kinesisReq).promise()
    Log.debug(`published 'restaurant_notified' event to Kinesis`)
  }  
}
```

5. Run the integration tests

`STAGE=dev REGION=us-east-1 npm run test`

and see that the functions are now logging in JSON

```
  When we invoke the GET / endpoint
SSM params loaded
AWS credential loaded
invoking via handler function get-index
{"message":"loading index.html...","awsRegion":"us-east-1","environment":"dev","level":20,"sLevel":"DEBUG"}
{"message":"loaded","awsRegion":"us-east-1","environment":"dev","level":20,"sLevel":"DEBUG"}
{"message":"received restaurants","count":8,"awsRegion":"us-east-1","environment":"dev","level":20,"sLevel":"DEBUG"}
    ✓ Should return the index page with 8 restaurants (763ms)

  When we invoke the GET /restaurants endpoint
invoking via handler function get-restaurants
    ✓ Should return an array of 8 restaurants (1236ms)

  When we invoke the notify-restaurant function
invoking via handler function notify-restaurant
{"message":"notified restaurant of order","restaurantName":"Fangtasia","orderId":"ba847fbd-7de3-5ed2-b41f-60ec1eabce37","awsRegion":"us-east-1","environment":"dev","level":20,"sLevel":"DEBUG"}
{"message":"published 'restaurant_notified' event to Kinesis","awsRegion":"us-east-1","environment":"dev","level":20,"sLevel":"DEBUG"}
(node:76670) [DEP0005] DeprecationWarning: Buffer() is deprecated due to security and usability issues. Please use the Buffer.alloc(), Buffer.allocUnsafe(), or Buffer.from() methods instead.
    ✓ Should publish message to SNS
    ✓ Should publish event to Kinesis

  When we invoke the POST /orders endpoint
invoking via handler function place-order
{"message":"placing order","orderId":"739d98ee-1e14-5367-b089-a9eb9798c3f6","restaurantName":"Fangtasia","awsRegion":"us-east-1","environment":"dev","level":20,"sLevel":"DEBUG"}
{"message":"published 'order_placed' event into Kinesis","awsRegion":"us-east-1","environment":"dev","level":20,"sLevel":"DEBUG"}
    ✓ Should return 200
    ✓ Should publish a message to Kinesis stream

  When we invoke the POST /restaurants/search endpoint with theme 'cartoon'
invoking via handler function search-restaurants
    ✓ Should return an array of 4 restaurants (487ms)


  7 passing (3s)
```

</p></details>

<details>
<summary><b>Disable debug logging in production</b></summary><p>

This logger allows you to control the default log level via the `LOG_LEVEL` environment variable.

1. In the `terraform` folder, open `variables.tf`.

2. **Add** the following to the end of the file:

```terraform
variable "log_level" {
  description = "The level functions should log at"
  type        = "string"
  default     = "DEBUG"
}
```

This allows us to override the log level via a Terraform variable.

3. In the `terraform` folder, open `get-index.tf`.

4. Find the resource `aws_lambda_function.get_index`, and **replace** it with the following:

```terraform
resource "aws_lambda_function" "get_index" {
  function_name = "${local.function_prefix}-get-index"

  s3_bucket = "${local.deployment_bucket}"
  s3_key    = "${local.deployment_key}"

  handler = "functions/get-index.handler"
  runtime = "nodejs8.10"

  role = "${aws_iam_role.get_index_lambda_role.arn}"
  timeout = 6

  environment {
    variables = {
      restaurants_api = "https://${aws_api_gateway_rest_api.api.id}.execute-api.us-east-1.amazonaws.com/${var.stage}/restaurants"
      orders_api = "https://${aws_api_gateway_rest_api.api.id}.execute-api.us-east-1.amazonaws.com/${var.stage}/orders"
      LOG_LEVEL = "${var.log_level}"
    }
  }
}
```

5. In the `terraform` folder, open `place-order.tf`.

6. Find the resource `aws_lambda_function.place_order`, and **replace** it with the following:

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
      LOG_LEVEL = "${var.log_level}"
    }
  }
}
```

7. In the `terraform` folder, open `search-restaurants.tf`.

8. Find the resource `aws_lambda_function.search_restaurants`, and **replace** it with the following:

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
      LOG_LEVEL = "${var.log_level}"
    }
  }
}
```

9. In the `terraform` folder, open `notify-restaurant.tf`.

10. Find the resource `aws_lambda_function.notify_restaurant`, and **replace** it with the following:

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
      order_events_stream = "${aws_kinesis_stream.orders_stream.name}"
      restaurant_notification_topic = "${aws_sns_topic.restaurant_notification.arn}"
      LOG_LEVEL = "${var.log_level}"
    }
  }
}
```

11. Redeploy the project.

12. Run the acceptance tests to make sure it still works.

`STAGE=dev REGION=us-east-1 npm run acceptance`

13. After running the tests, check the Lambda function logs. You should see that they're in JSON format, and that the debug logs are still coming through.

14. Now, as an experiment, trying changing the default log level (by updating `terraform/variables.tf`) to `INFO` and redeploy. You should see that none of the debug log messages are coming through.

</p></details>