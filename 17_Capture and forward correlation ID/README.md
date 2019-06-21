# Module 17: Capture and forward correlation ID

## Capture and forward correlation IDs

Since we're using the `@perform/lambda-powertools-pattern-basic` and `@perform/lambda-powertools-logger`, we are already capturing incoming correlation IDs and including them in the logs.

However, we need to make sure the `get-index` function forwards the correlation IDs along, while still using the X-Ray instrumented `https` module.

The `dazn-lambda-powertools` project also has Kinesis and SNS clients that will automatically forward captured correlation IDs. We can use them as stand-in replacements for the Kinesis and SNS clients from the AWS SDK.

<details>
<summary><b>Forward correlation IDs from get-index</b></summary><p>

You can access the auto-captured correlation IDs using the `@perform/lambda-powertools-correlation-ids` package. From here, we can include them as HTTP headers.

1. At the project root, run `npm install --save @perform/lambda-powertools-correlation-ids`.

2. Open `functions/get-index.js`, **replace** it with the following:

```javascript
const fs = require("fs")
const Mustache = require('mustache')
const AWSXRay = require('aws-xray-sdk-core')
const https = process.env.LAMBDA_RUNTIME_DIR
  ? AWSXRay.captureHTTPs(require('https'))
  : require('https')
const aws4 = require('aws4')
const URL = require('url')
const Log = require('@perform/lambda-powertools-logger')
const wrap = require('@perform/lambda-powertools-pattern-basic')
const CorrelationIds = require('@perform/lambda-powertools-correlation-ids')

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
      headers: Object.assign({}, CorrelationIds.get(), opts.headers)
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

module.exports.handler = wrap(async (event, context) => {
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
})
```

In this change, we brought in the correlation IDs package on ln11:

```javascript
const CorrelationIds = require('@perform/lambda-powertools-correlation-ids')
```

We then included the correlation IDs in the headers on ln44:

```javascript
const options = {
  hostname: url.hostname,
  port: 443,
  path: url.pathname,
  method: 'GET',
  headers: Object.assign({}, CorrelationIds.get(), opts.headers)
}
```

These changes are enough to ensure correlation IDs are included in the HTTP headers in the request to the `GET /restaurants` endpoint.

3. To see the correlation IDs are forwarded along, and included in the logs in the `get-restaurants` function. Let's also update the `get-restaurants` function.

Open `functions/get-restaurants.js`, and **replace** it with the following:

```javascript
const AWSXRay = require('aws-xray-sdk-core')
const AWS = process.env.LAMBDA_RUNTIME_DIR
  ? AWSXRay.captureAWS(require('aws-sdk'))
  : require('aws-sdk')
const dynamodb = new AWS.DynamoDB.DocumentClient()
const wrap = require('@perform/lambda-powertools-pattern-basic')
const Log = require('@perform/lambda-powertools-logger')

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

module.exports.handler = wrap(async (event, context) => {
  const restaurants = await getRestaurants(defaultResults)
  Log.debug('found restaurants', { count: restaurants.length })
  const response = {
    statusCode: 200,
    body: JSON.stringify(restaurants)
  }

  return response
})
```

4. Open `tests/step/when.js`, and **replace** the `viaHandler` function with the following:

```javascript
const viaHandler = async (event, functionName) => {
  const handler = util.promisify(require(`${APP_ROOT}/functions/${functionName}`).handler)
  console.log(`invoking via handler function ${functionName}`)

  const context = { awsRequestId: 'test' }
  const response = await handler(event, context)
  const contentType = _.get(response, 'headers.content-type', 'application/json');
  if (_.get(response, 'body') && contentType === 'application/json') {
    response.body = JSON.parse(response.body);
  }
  return response
}
```

5. Deploy the project.

6. Once the deployment is done, load the page. And then open the X-Ray console to make sure that the X-Ray tracing is still working.

![](/images/mod17-001.png)

7. Open the CloudWatch console to check the logs for both `get-index` and `get-restaurants`. You should see that the same correlation ID is included in both logs.

In the below example, you can see the correlation ID `d08d3858-9b4b-488a-b502-88b6870f53e5` in both functions' logs. You can also see the `debug-log-enabled` decision is passed along as well.

![](/images/mod17-002.png)

![](/images/mod17-003.png)

</p></details>

<details>
<summary><b>Forward correlation IDs from Kinesis functions</b></summary><p>

1. Open `functions/place-order.js`, and **replace** the following line (ln6):

```javascript
const kinesis = new AWS.Kinesis()
```

with the following:

```javascript
const kinesis = require('@perform/lambda-powertools-kinesis-client')
```

This would be enough for the `place-order` function to forward the current set of correlation IDs to the Kinesis stream.

2. Open `functions/notify-restaurant.js`, and **replace** the file with the following:

```javascript
const _ = require('lodash')
const AWSXRay = require('aws-xray-sdk-core')
const AWS = process.env.LAMBDA_RUNTIME_DIR
  ? AWSXRay.captureAWS(require('aws-sdk'))
  : require('aws-sdk')
const kinesis = require('@perform/lambda-powertools-kinesis-client')
const sns = require('@perform/lambda-powertools-sns-client')
const Log = require('@perform/lambda-powertools-logger')
const wrap = require('@perform/lambda-powertools-pattern-basic')

const streamName = process.env.order_events_stream
const topicArn = process.env.restaurant_notification_topic

module.exports.handler = wrap(async (event, context) => {
  const events = context.parsedKinesisEvents
  Log.debug('processing order events', { count: events.length })

  const promises = events
    .filter(evt => evt.eventType === 'order_placed')
    .map(async order => {
      order.logger.debug(
        'notified restaurant of order', 
        { restaurantName: order.restaurantName, orderId: order.orderId})

      const snsReq = {
        Message: JSON.stringify(order),
        TopicArn: topicArn
      };
      await sns.publishWithCorrelationIds(order.correlationIds, snsReq).promise()

      const data = _.clone(order)
      data.eventType = 'restaurant_notified'

      const kinesisReq = {
        Data: JSON.stringify(data), // the SDK would base64 encode this for us
        PartitionKey: order.orderId,
        StreamName: streamName
      }
      await kinesis.putRecordWithCorrelationIds(order.correlationIds, kinesisReq).promise()
      order.logger.debug(`published 'restaurant_notified' event to Kinesis`)
    })

  await Promise.all(promises)
})
```

Couple of things to note in this change:

* the `context` object already contains an array of the based64 decoded, and JSON parsed events in `parsedKinesisEvents`
* every parsed event has its own logger and set of correlation IDs
* we use the SNS and Kinesis client's override functions to include the correlation IDs specific to each event
* we can process the parsed events safely in parallel

The reason for these special behaviour is that Kinesis and SQS are batch-based event sources. Every record in the batch has its own set of correlation IDs and should be used when processing those events. For more detail on the rationale for this design approach, please read [this](https://github.com/getndazn/dazn-lambda-powertools/tree/master/packages/lambda-powertools-middleware-correlation-ids#logging-and-forwarding-correlation-ids-for-kinesis-and-sqs-events).

3. We no longer need the `lib/kinesis.js` module anymore. Delete it.

4. Deploy the project.

5. Once the deployment is done, reload the page and place an order. Go to the X-Ray console, and make sure that the `notify-restaurant` function's tracing still works.

![](/images/mod17-004.png)

6. Open the CloudWatch console, and look for the logs for the `place-order` and `notify-restaurant` functions. You should see the same correlation IDs span across these two functions' logs.

In the following example, you can see the correlation ID `b4a6cacc-0cbd-4d77-8bab-cbca27b914ad` in the logs for both functions. You can also see the `debug-log-enabled` decision is passed along as well.

![](/images/mod17-005.png)

![](/images/mod17-006.png)

</p></details>
