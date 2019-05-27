# Module 7: Writing acceptance tests

## Add acceptance tests

**Goal:** Write acceptance tests

<details>
<summary><b>Reuse get-index test case for acceptance testing</b></summary><p>

1. In the `tests/steps/when.js`, we need to add new functions to invoke functions remotely via API Gateway. **Replace** `tests/steps/when.js` with the following

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
  const contentType = _.get(response, 'headers.Content-Type', 'application/json');
  if (response.body && contentType === 'application/json') {
    response.body = JSON.parse(response.body);
  }
  return response
}

const we_invoke_get_index = () => viaHandler({}, 'get-index')

const we_invoke_get_restaurants = () => viaHandler({}, 'get-restaurants')

const we_invoke_search_restaurants = theme => {
  let event = {
    body: JSON.stringify({ theme })
  }
  return viaHandler(event, 'search-restaurants')
}

module.exports = {
  we_invoke_get_index,
  we_invoke_get_restaurants,
  we_invoke_search_restaurants
}
```

2. Staying in the `tests/steps/when.js` file, **replace** the `we_invoke_get_index` function with the following (which toggles between invoking function locally and remotely based on the `mode` variable).

```javascript
const we_invoke_get_index = async () => {
  const res = 
    mode === 'handler' 
      ? await viaHandler({}, 'get-index')
      : await viaHttp('', 'GET')

  return res
}
```

3. Open `tests/steps/init.js` and add a new `TEST_ROOT` environment variable (to the `init` function), using the API Gateway endpoint you have deployed.

Your `init` function should look like this (**remember to replace the URLs with your deployed URL and the restaurants table name**)

```javascript
const init = async () => {
  if (initialized) {
    return
  }

  process.env.TEST_ROOT = "https://xxx.execute-api.us-east-1.amazonaws.com/dev"
  process.env.restaurants_api = "https://xxx.execute-api.us-east-1.amazonaws.com/dev/restaurants"
  process.env.restaurants_table = "<replace with your table>"
  process.env.AWS_REGION = "us-east-1"
  
  const { credentials } = await promisify(awscred.load)()
  
  process.env.AWS_ACCESS_KEY_ID     = credentials.accessKeyId
  process.env.AWS_SECRET_ACCESS_KEY = credentials.secretAccessKey

  console.log('AWS credential loaded')

  initialized = true
}
```

4. Open the `package.json` file at the project root, **add** `TEST_MODE` to integration test script, and add an acceptance test script

```json
"scripts": {
  "test": "TEST_MODE=handler mocha tests/test_cases --reporter spec",
  "acceptance": "TEST_MODE=http mocha tests/test_cases --reporter spec"
},
```

5. Run the acceptance test

`npm run acceptance`

and see that the `get-index` function is failing

```
  When we invoke the GET / endpoint
AWS credential loaded
invoking via HTTP GET https://exun14zd2h.execute-api.us-east-1.amazonaws.com/dev/
    1) Should return the index page with 8 restaurants

  1) When we invoke the GET / endpoint
       Should return the index page with 8 restaurants:
     AssertionError: expected undefined to equal 'text/html; charset=UTF-8'
      at Context.it (tests/test_cases/get-index.js:13:44)
      at <anonymous>
      at process._tickCallback (internal/process/next_tick.js:188:7)
```

This is because the HTTP client `superagent` lower-cases the `Content-Type` automatically.

6. Modify `tests/test_cases/get-index.js` to look for `content-type` instead of `Content-Type`

```javascript
expect(res.headers['content-type']).to.equal('text/html; charset=UTF-8')
```

7. Modify `functions/get-index.js` to return `content-type` header instead of `Content-Type`

```javascript
const response = {
  statusCode: 200,
  headers: {
    'content-type': 'text/html; charset=UTF-8'
  },
  body: html
}
```

8. Modify `tests/steps/when.js` to look for `content-type` instead of `Content-Type`

```javascript
const viaHandler = async (event, functionName) => {
  const handler = require(`${APP_ROOT}/functions/${functionName}`).handler
  console.log(`invoking via handler function ${functionName}`)

  const context = {}
  const response = await handler(event, context)
  const contentType = _.get(response, 'headers.content-type', 'application/json');
  if (response.body && contentType === 'application/json') {
    response.body = JSON.parse(response.body);
  }
  return response
}
```

9. Re-run the acceptance test

`npm run acceptance`

and see that the `get-index` function is now passing

```
  When we invoke the GET / endpoint
AWS credential loaded
invoking via HTTP GET https://sr73zpk0el.execute-api.us-east-1.amazonaws.com/dev/
    ✓ Should return the index page with 8 restaurants (658ms)
```

</p></details>

<details>
<summary><b>Reuse get-restaurants test case for acceptance testing</b></summary><p>

1. Open `tests/steps/when.js` and **replace** the `we_invoke_get_restaurants` function so we can toggle between invoking function locally and remotely

```javascript
const we_invoke_get_restaurants = async () => {
  const res =
    mode === 'handler' 
      ? await viaHandler({}, 'get-restaurants')
      : await viaHttp('restaurants', 'GET', { iam_auth: true })

  return res
}
```

2. Re-run the acceptance test

`npm run acceptance`

and see that both `get-index` and `get-restaurants` tests are passing

```
  When we invoke the GET / endpoint
AWS credential loaded
invoking via HTTP GET https://exun14zd2h.execute-api.us-east-1.amazonaws.com/dev/
    ✓ Should return the index page with 8 restaurants (632ms)

  When we invoke the GET /restaurants endpoint
invoking via HTTP GET https://exun14zd2h.execute-api.us-east-1.amazonaws.com/dev/restaurants
    ✓ Should return an array of 8 restaurants (380ms)
```

</p></details>

<details>
<summary><b>Modify search-restaurants test case for acceptance testing</b></summary><p>

1. Open `tests/steps/when.js` and **replace** the `we_invoke_search_restaurants` function with the following so we can toggle between invoking function locally and remotely

```javascript
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
```

2. Re-run the acceptance tests

`npm run acceptance`

and see that all 3 tests are passing

```
  When we invoke the GET / endpoint
AWS credential loaded
invoking via HTTP GET https://sr73zpk0el.execute-api.us-east-1.amazonaws.com/dev/
    ✓ Should return the index page with 8 restaurants (1196ms)

  When we invoke the GET /restaurants endpoint
invoking via HTTP GET https://sr73zpk0el.execute-api.us-east-1.amazonaws.com/dev/restaurants
    ✓ Should return an array of 8 restaurants (394ms)

  When we invoke the POST /restaurants/search endpoint with theme 'cartoon'
invoking via HTTP POST https://sr73zpk0el.execute-api.us-east-1.amazonaws.com/dev/restaurants/search
    ✓ Should return an array of 4 restaurants (2138ms)


  3 passing (4s)
```

3. Re-run the **integration tests**

`npm run test`

and check that all 3 tests are still passing as well

```
  When we invoke the GET / endpoint
AWS credential loaded
loading index.html...
loaded
    ✓ Should return the index page with 8 restaurants (740ms)

  When we invoke the GET /restaurants endpoint
    ✓ Should return an array of 8 restaurants (1269ms)

  When we invoke the POST /restaurants/search endpoint with theme 'cartoon'
    ✓ Should return an array of 4 restaurants (253ms)


  3 passing (2s)
```

</p></details>