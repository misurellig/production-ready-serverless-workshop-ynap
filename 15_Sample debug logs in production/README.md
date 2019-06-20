# Module 15: Sample debug logs in production

## Sample 1% of debug logs in production

<details>
<summary><b>Install lambda-powertools-pattern-basic</b></summary><p>

1. At the project root, run the command `npm install --save @perform/lambda-powertools-pattern-basic`.

This package gives you a simple wrapper which applies a couple of [middy](https://github.com/middyjs/middy) middlewares for your function:

* `@perform/lambda-powertools-middleware-sample-logging`: which supports sampling debug logs. The wrapper configures this sample logging middleware to sample debug logs for 1% of invocations.

* `@perform/lambda-powertools-middleware-correlation-ids`: which extracts correlation IDs from the invocation event and makes them available for the logger. It also supports a special correlation ID `debug-log-enabled`, which enables sampling debug logs at the user transaction (a chain of Lambda invocations) level.

* `@perform/lambda-powertools-middleware-log-timeout`: which emits an error message for when a function times out. Normally, when a Lambda function times out, you don't get an error message from the application, which makes debugging time out errors difficult.

Now we need to apply it to all of our functions.

2. Open `functions/get-index.js`, and **replace** it with the following:

```javascript
const fs = require("fs")
const Mustache = require('mustache')
const http = require('superagent-promise')(require('superagent'), Promise)
const aws4 = require('aws4')
const URL = require('url')
const Log = require('@perform/lambda-powertools-logger')
const wrap = require('@perform/lambda-powertools-pattern-basic')

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

3. Open `functions/get-restaurants.js`, and **replace** it with the following:

```javascript
const AWS = require('aws-sdk')
const dynamodb = new AWS.DynamoDB.DocumentClient()
const wrap = require('@perform/lambda-powertools-pattern-basic')

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
  const response = {
    statusCode: 200,
    body: JSON.stringify(restaurants)
  }

  return response
})
```

4. Open `functions/place-order.js`, and **replace** it with the following:

```javascript
const _ = require('lodash')
const AWS = require('aws-sdk')
const kinesis = new AWS.Kinesis()
const chance = require('chance').Chance()
const streamName = process.env.order_events_stream
const Log = require('@perform/lambda-powertools-logger')
const wrap = require('@perform/lambda-powertools-pattern-basic')

module.exports.handler = wrap(async (event, context) => {
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
})
```

5. Open `functions/search-restaurants.js`, and **replace** it with the following:

```javascript
const AWS = require('aws-sdk')
const dynamodb = new AWS.DynamoDB.DocumentClient()
const wrap = require('@perform/lambda-powertools-pattern-basic')

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

module.exports.handler = wrap(async (event, context) => {
  const req = JSON.parse(event.body)
  const theme = req.theme
  const restaurants = await findRestaurantsByTheme(theme, defaultResults)
  const response = {
    statusCode: 200,
    body: JSON.stringify(restaurants)
  }

  return response
})
```

6. Open `functions/notify-restaurant.js`, and **replace** it with the following:

```javascript
const _ = require('lodash')
const { getRecords } = require('../lib/kinesis')
const AWS = require('aws-sdk')
const kinesis = new AWS.Kinesis()
const sns = new AWS.SNS()
const Log = require('@perform/lambda-powertools-logger')
const wrap = require('@perform/lambda-powertools-pattern-basic')

const streamName = process.env.order_events_stream
const topicArn = process.env.restaurant_notification_topic

module.exports.handler = wrap(async (event, context) => {
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
})
```

7. Run integration test

`STAGE=dev REGION=us-east-1 npm run test`

and see that tests are failing with error messages like this:

```
1) When we invoke the GET / endpoint
       Should return the index page with 8 restaurants:
     TypeError: callback is not a function
      at terminate (node_modules/middy/src/middy.js:152:16)
      at runNext (node_modules/middy/src/middy.js:126:14)
      at runErrorMiddlewares (node_modules/middy/src/middy.js:130:3)
      at errorHandler (node_modules/middy/src/middy.js:160:14)
      at runMiddlewares (node_modules/middy/src/middy.js:164:23)
      at runNext (node_modules/middy/src/middy.js:87:14)
      at runMiddlewares (node_modules/middy/src/middy.js:91:3)
      at instance (node_modules/middy/src/middy.js:163:5)
      at viaHandler (tests/steps/when.js:82:26)
      at Object.we_invoke_get_index (tests/steps/when.js:93:15)
      at Context.it (tests/test_cases/get-index.js:10:28)
      at process.topLevelDomainCallback (domain.js:120:23)
```

This is because, `middy` turns our functions into callback style functions so that it's backward compatible with Node 6.10 as well.

So we need to update `tests/steps/when.js` to match this. We'll use `util.promisify` to turn the handler function back to async function.

8. Open `tests/steps/when.js`, and **replace** it with the following:

```javascript
const APP_ROOT = '../../'
const _ = require('lodash')
const aws4 = require('aws4')
const URL = require('url')
const http = require('superagent-promise')(require('superagent'), Promise)
const util = require('util')
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
  const handler = util.promisify(require(`${APP_ROOT}/functions/${functionName}`).handler)
  console.log(`invoking via handler function ${functionName}`)

  const context = {}
  const response = await handler(event, context)
  const contentType = _.get(response, 'headers.content-type', 'application/json');
  if (_.get(response, 'body') && contentType === 'application/json') {
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

module.exports = {
  we_invoke_get_index,
  we_invoke_get_restaurants,
  we_invoke_search_restaurants,
  we_invoke_place_order,
  we_invoke_notify_restaurant,
}
```

The change is mainly in the `viaHandler` function, where we use `util.promisify` to turn the handler back to an `async` function.

```javascript
const handler = util.promisify(require(`${APP_ROOT}/functions/${functionName}`).handler)
```

9. Rerun integration tests again

`STAGE=dev REGION=us-east-1 npm run test`

and see that all the tests are now passing

```
  When we invoke the GET / endpoint
SSM params loaded
AWS credential loaded
invoking via handler function get-index
{"message":"loading index.html...","awsRegion":"us-east-1","environment":"dev","debug-log-enabled":"false","call-chain-length":1,"level":20,"sLevel":"DEBUG"}
{"message":"loaded","awsRegion":"us-east-1","environment":"dev","debug-log-enabled":"false","call-chain-length":1,"level":20,"sLevel":"DEBUG"}
{"message":"received restaurants","count":8,"awsRegion":"us-east-1","environment":"dev","debug-log-enabled":"false","call-chain-length":1,"level":20,"sLevel":"DEBUG"}
    ✓ Should return the index page with 8 restaurants (779ms)

  When we invoke the GET /restaurants endpoint
invoking via handler function get-restaurants
    ✓ Should return an array of 8 restaurants (1261ms)

  When we invoke the notify-restaurant function
invoking via handler function notify-restaurant
{"message":"notified restaurant of order","restaurantName":"Fangtasia","orderId":"89b64e29-920a-567d-a41f-a6ac37923f8f","awsRegion":"us-east-1","environment":"dev","debug-log-enabled":"false","level":20,"sLevel":"DEBUG"}
{"message":"published 'restaurant_notified' event to Kinesis","awsRegion":"us-east-1","environment":"dev","debug-log-enabled":"false","level":20,"sLevel":"DEBUG"}
(node:79374) [DEP0005] DeprecationWarning: Buffer() is deprecated due to security and usability issues. Please use the Buffer.alloc(), Buffer.allocUnsafe(), or Buffer.from() methods instead.
    ✓ Should publish message to SNS
    ✓ Should publish event to Kinesis

  When we invoke the POST /orders endpoint
invoking via handler function place-order
{"message":"placing order","orderId":"ba36c09c-8cad-5ab5-9c93-09d7ec0f760d","restaurantName":"Fangtasia","awsRegion":"us-east-1","environment":"dev","debug-log-enabled":"false","call-chain-length":1,"level":20,"sLevel":"DEBUG"}
{"message":"published 'order_placed' event into Kinesis","awsRegion":"us-east-1","environment":"dev","debug-log-enabled":"false","call-chain-length":1,"level":20,"sLevel":"DEBUG"}
    ✓ Should return 200
    ✓ Should publish a message to Kinesis stream

  When we invoke the POST /restaurants/search endpoint with theme 'cartoon'
invoking via handler function search-restaurants
    ✓ Should return an array of 4 restaurants (253ms)


  7 passing (3s)
```

10. Deploy the project.

11. Run the acceptance tests to make sure they're still passing.

`STAGE=dev REGION=us-east-1 npm run acceptance`

</p></details>