# Module 6: Writing integration tests

## Add integration tests

**Goal:** Write integration tests

<details>
<summary><b>Prepare tests</b></summary><p>

1. Add a `tests` folder to the project root

2. Add a `test_cases` folder under `tests`

3. Add a `steps` folder under `tests`

At this point, your project folder should look like this

```
functions
  |-- get-index.js
  |-- get-restaurants.js
terraform
  |-- apigateway.tf
  |-- dynamodb.tf
  |-- get-index.tf
  |-- get-restaurants.tf
  |-- locals.tf
  |-- outputs.tf
  |-- provider.tf
  |-- variables.tf
static
  |-- index.html
tests
  |-- /test_cases
  |-- /steps
build.sh
package.json
seed-restaurants.js
```

4. Install `chai` as a dev dependency

`npm install --save-dev chai`

5. Install `mocha` as a dev dependency

`npm install --save-dev mocha`

6. Install `cheerio` as a dev dependency

`npm install --save-dev cheerio`

7. Install `awscred` as a dependency

`npm install --save awscred`

8. Install `lodash` as a dependency

`npm install --save lodash`

</p></details>

<details>
<summary><b>Add test case for get-index</b></summary><p>

1. Add `get-index.js` file under `test_cases`

2. Copy the following into `tests/test_cases/get-index.js`

```javascript
const { expect } = require('chai')
const cheerio = require('cheerio')

describe(`When we invoke the GET / endpoint`, () => {
  it(`Should return the index page with 8 restaurants`, async () => {
    const res = await when.we_invoke_get_index()

    expect(res.statusCode).to.equal(200)
    expect(res.headers['Content-Type']).to.equal('text/html; charset=UTF-8')
    expect(res.body).to.not.be.null

    const $ = cheerio.load(res.body)
    const restaurants = $('.restaurant', '#restaurantsUl')
    expect(restaurants.length).to.equal(8)
  })
})
```

3. Add `when.js` file under `steps`

4. Copy the following into `tests/steps/when.js`

```javascript
const APP_ROOT = '../../'
const _ = require('lodash')

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

module.exports = {
  we_invoke_get_index
}
```

5. **Add** `const when = require('../steps/when')` to the top of `tests/test_cases/get-index.js`

6. Modify the `package.json` and add a `test` script

```json
"scripts": {
  "test": "mocha tests/test_cases --reporter spec"
},
```

7. Run the integration test

`npm run test`

and see that the test fails with the error 

```
When we invoke the GET / endpoint
loading index.html...
loaded
    1) Should return the index page with 8 restaurants


  0 passing (92ms)
  1 failing

  1) When we invoke the GET / endpoint
       Should return the index page with 8 restaurants:
     TypeError [ERR_INVALID_ARG_TYPE]: The "url" argument must be of type string. Received type undefined
      at Url.parse (url.js:152:11)
      at Object.urlParse [as parse] (url.js:146:13)
      at getRestaurants (functions/get-index.js:23:19)
      at module.exports.handler (functions/get-index.js:42:29)
      at viaHandler (tests/steps/when.js:8:26)
      at Object.we_invoke_get_index (tests/steps/when.js:16:35)
      at Context.it (tests/test_cases/get-index.js:7:28)
```

This is because the `get-index` function needs a number of environment variables.

8. Add `init.js` under `steps` folder

9. Copy the following into `tests/steps/init.js`. Don't forget to **replace** the `restaurants_api` URL to the invoke URL you have been using, and **replace** the `restaurants_table` to the name of the table you created.

```javascript
const { promisify } = require('util')
const awscred = require('awscred')

let initialized = false

const init = async () => {
  if (initialized) {
    return
  }

  process.env.restaurants_api      = "<invoke URL>/restaurants"
  process.env.restaurants_table    = "<your table name>"
  process.env.AWS_REGION           = "eu-central-1"
  
  const { credentials } = await promisify(awscred.load)()
  
  process.env.AWS_ACCESS_KEY_ID     = credentials.accessKeyId
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

10. In the `tests/test_cases/get-index.js`, we need to require the `init` module and execute it before running the test. **Replace** the module with the following

```javascript
const when = require('../steps/when')
const { expect } = require('chai')
const cheerio = require('cheerio')
const { init } = require('../steps/init')

describe(`When we invoke the GET / endpoint`, () => {
  before(async () => await init())
  
  it(`Should return the index page with 8 restaurants`, async () => {
    const res = await when.we_invoke_get_index()

    expect(res.statusCode).to.equal(200)
    expect(res.headers['Content-Type']).to.equal('text/html; charset=UTF-8')
    expect(res.body).to.not.be.null

    const $ = cheerio.load(res.body)
    const restaurants = $('.restaurant')
    expect(restaurants.length).to.equal(8)
  })
})
```

11. Run the integration test again

`npm run test`

and see that the test fails with a different error 

```
When we invoke the GET / endpoint
AWS credential loaded
loading index.html...
loaded
    1) Should return the index page with 8 restaurants


  0 passing (116ms)
  1 failing

  1) When we invoke the GET / endpoint
       Should return the index page with 8 restaurants:
     TypeError [ERR_HTTP_INVALID_HEADER_VALUE]: Invalid value "undefined" for header "X-Amz-Security-Token"
      at ClientRequest.setHeader (_http_outgoing.js:473:3)
      at PromiseRequest.Request.request (node_modules/superagent/lib/node/index.js:814:69)
      at PromiseRequest.Request.end (node_modules/superagent/lib/node/index.js:930:8)
      at /Users/yan.cui/SourceCode/personal/production-ready-serverless-workshop-ynap-demo/node_modules/superagent-promise/index.js:44:12
      at new Promise (<anonymous>)
      at PromiseRequest.then (node_modules/superagent-promise/index.js:43:12)
      at process._tickCallback (internal/process/next_tick.js:68:7)
```

This is because when we execute the test we are using an IAM role associated with an IAM user, which does not have a security token. The code we added to the `get-index` function earlier would sign the HTTP request to the `/restaurants` endpoint while making the assumption that it's using a temporary IAM credential for an IAM role.

To fix this error, we need to add an `if` condition and only set the `X-Amz-Security-Token` header if the 

12. In the `functions/get-index.js`, **replace** the `getRestaurants` function with the below (to only set the `X-Amz-Security-Token` header if it's applicable).

```javascript
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
```

13. Run the integration test again

`npm run test`

The test should pass now

```
  When we invoke the GET / endpoint
AWS credential loaded
loading index.html...
loaded
    ✓ Should return the index page with 8 restaurants (449ms)


  1 passing (467ms)
```

</p></details>

If you find that some tests fails sporadically, they might just need longer to run. Try raising the test timeout with `--timeout 5000`, the default timeout is 2s.

## Exercises

<details>
<summary><b>Add test case for get-restaurants</b></summary><p>

1. Add `get-restaurants.js` under `tests/test_cases`

2. Copy the following into `get-restaurants.js`

```javascript
const { expect } = require('chai')
const { init } = require('../steps/init')
const when = require('../steps/when')

describe(`When we invoke the GET /restaurants endpoint`, () => {
  before(async () => await init())

  it(`Should return an array of 8 restaurants`, async () => {
    let res = await when.we_invoke_get_restaurants()

    expect(res.statusCode).to.equal(200)
    expect(res.body).to.have.lengthOf(8)

    for (let restaurant of res.body) {
      expect(restaurant).to.have.property('name')
      expect(restaurant).to.have.property('image')
    }
  })
})
```

3. Open `tests/steps/when.js` and **add** a `we_invoke_get_restaurants` function (just above `module.exports = {`)

```javascript
const we_invoke_get_restaurants = () => viaHandler({}, 'get-restaurants')
```

4. Now that we have added a new function in `tests/steps/when.js` we need to export it still. Staying in `tests/steps/when.js` and **replace** the following lines 

```javascript
module.exports = {
  we_invoke_get_index
}
```

with this:

```javascript
module.exports = {
  we_invoke_get_index,
  we_invoke_get_restaurants
}
```

5. Run the integration test

`npm run test`

and see that both tests pass

```
  When we invoke the GET / endpoint
AWS credential loaded
loading index.html...
loaded
    ✓ Should return the index page with 8 restaurants (371ms)

  When we invoke the GET /restaurants endpoint
    ✓ Should return an array of 8 restaurants (451ms)


  2 passing (839ms)
```

</p></details>

<details>
<summary><b>Add test case for search-restaurants</b></summary><p>

1. Add `search-restaurants.js` under `tests/test_cases`

2. Copy the following into `tests/test_cases/search-restaurants.js`

```javascript
const { expect } = require('chai')
const { init } = require('../steps/init')
const when = require('../steps/when')

describe(`When we invoke the POST /restaurants/search endpoint with theme 'cartoon'`, () => {
  before(async () => await init())

  it(`Should return an array of 4 restaurants`, async () => {
    let res = await when.we_invoke_search_restaurants('cartoon')

    expect(res.statusCode).to.equal(200)
    expect(res.body).to.have.lengthOf(4)

    for (let restaurant of res.body) {
      expect(restaurant).to.have.property('name')
      expect(restaurant).to.have.property('image')
    }
  })
})
```

3. Open `tests/steps/when.js` and **add** a `we_invoke_search_restaurants` function (just above `module.exports = {`)

```javascript
const we_invoke_search_restaurants = theme => {
  let event = {
    body: JSON.stringify({ theme })
  }
  return viaHandler(event, 'search-restaurants')
}
```

4. Now that we have added a new function in `tests/steps/when.js` we need to export it still. Staying in `tests/steps/when.js` and **replace** the following lines 

```javascript
module.exports = {
  we_invoke_get_index,
  we_invoke_get_restaurants
}
```

with this:

```javascript
module.exports = {
  we_invoke_get_index,
  we_invoke_get_restaurants,
  we_invoke_search_restaurants
}
```

5. Run the integration test

`npm run test`

and see that the tests pass

```
  When we invoke the GET / endpoint
AWS credential loaded
loading index.html...
loaded
    ✓ Should return the index page with 8 restaurants (435ms)

  When we invoke the GET /restaurants endpoint
    ✓ Should return an array of 8 restaurants (440ms)

  When we invoke the POST /restaurants/search endpoint with theme 'cartoon'
    ✓ Should return an array of 4 restaurants (249ms)


  3 passing (1s)
```

</p></details>

<details>
<summary><b>What other test cases would you add?</b></summary><p>

</p></details>