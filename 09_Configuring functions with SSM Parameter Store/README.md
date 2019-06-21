# Module 09: Configuring functions with SSM Parameter Store

## Use SSM parameter store to configure resource names, URLs, etc.

**Goal:** Configurations are managed in SSM parameter store

<details>
<summary><b>Add parameters to Terraform</b></summary><p>

First, let's share the resources we create as parameters in SSM Parameter Store to share them with other services, and with our tests and seed script as well.

1. In the `terraform` folder, add a new file called `parameters.tf`.

2. Copy the following into the newly created `parameters.tf` file:

```terraform
resource "aws_ssm_parameter" "table_name" {
  name = "/big-mouth-${var.my_name}/${var.stage}/table_name"
  type = "String"
  value = "${aws_dynamodb_table.restaurants_table.name}"
}

resource "aws_ssm_parameter" "url" {
  name = "/big-mouth-${var.my_name}/${var.stage}/url"
  type = "String"
  value = "${aws_api_gateway_stage.stage.invoke_url}"
}
```

3. Deploy the changes either through Drone, or by running `./build.sh deploy dev` from the project root. Doing so would create the new parameters in SSM Parameter Store.

</p></details>

<details>
<summary><b>Use SSM to parameterise seed-restaurants script</b></summary><p>

1. **Replace** the `seed-restaurants.js` script with the following, and replace the `<PUT YOUR NAME HERE>` placeholder:

```javascript
const { REGION, STAGE } = process.env

const AWS = require('aws-sdk')
AWS.config.region = REGION
const dynamodb = new AWS.DynamoDB.DocumentClient()
const ssm = new AWS.SSM()

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

const getTableName = async () => {
  console.log('getting table name...')
  const req = {
    Name: `/big-mouth-<PUT YOUR NAME HERE>/${STAGE}/table_name`
  }
  const ssmResp = await ssm.getParameter(req).promise()
  return ssmResp.Parameter.Value
}

const run = async () => {
  const tableName = await getTableName()

  console.log(`table name: `, tableName)

  let putReqs = restaurants.map(x => ({
    PutRequest: {
      Item: x
    }
  }))
  
  const req = { 
    RequestItems: {}
  }
  req.RequestItems[tableName] = putReqs
  await dynamodb.batchWrite(req).promise()
}

run().then(() => console.log("all done")).catch(err => console.error(err.message))
```

2. Rerun the script

`STAGE=dev REGION=eu-central-1 node seed-restaurants.js`

and go to DynamoDB console to see that the newly created stage-specific table is now populated

</p></details>

<details>
<summary><b>Use SSM to parameterise tests</b></summary><p>

1. **Replace** the `tests/steps/init.js` file with the following, and replace the `<PUT YOUR NAME HERE>` placeholder:

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
    'url'
  ])

  console.log('SSM params loaded')

  process.env.TEST_ROOT                = params.url
  process.env.restaurants_api          = `${params.url}/restaurants`
  process.env.restaurants_table        = params.table_name
  process.env.AWS_REGION               = REGION
  
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

2. Rerun the integration tests

`STAGE=dev REGION=eu-central-1 npm run test`

and see that all the tests are passing

```
  When we invoke the GET / endpoint
SSM params loaded
AWS credential loaded
invoking via handler function get-index
loading index.html...
loaded
    ✓ Should return the index page with 8 restaurants (421ms)

  When we invoke the GET /restaurants endpoint
invoking via handler function get-restaurants
    ✓ Should return an array of 8 restaurants (405ms)

  Given an authenticated user
[test-Gilbert-Caselli-cRsp(Egv] - user is created
[test-Gilbert-Caselli-cRsp(Egv] - initialised auth flow
[test-Gilbert-Caselli-cRsp(Egv] - responded to auth challenge
    When we invoke the POST /restaurants/search endpoint with theme 'cartoon'
invoking via handler function search-restaurants
      ✓ Should return an array of 4 restaurants (248ms)
[test-Gilbert-Caselli-cRsp(Egv] - user deleted


  3 passing (3s)
```

3. Rerun the acceptance tests

`STAGE=dev REGION=eu-central-1 npm run acceptance`

```
  When we invoke the GET / endpoint
SSM params loaded
AWS credential loaded
invoking via HTTP GET https://exun14zd2h.execute-api.eu-central-1.amazonaws.com/dev/
    ✓ Should return the index page with 8 restaurants (916ms)

  When we invoke the GET /restaurants endpoint
invoking via HTTP GET https://exun14zd2h.execute-api.eu-central-1.amazonaws.com/dev/restaurants
    ✓ Should return an array of 8 restaurants (341ms)

  Given an authenticated user
[test-Viola-Brewer-keQeBQHj] - user is created
[test-Viola-Brewer-keQeBQHj] - initialised auth flow
[test-Viola-Brewer-keQeBQHj] - responded to auth challenge
    When we invoke the POST /restaurants/search endpoint with theme 'cartoon'
invoking via HTTP POST https://exun14zd2h.execute-api.eu-central-1.amazonaws.com/dev/restaurants/search
      ✓ Should return an array of 4 restaurants (1514ms)
[test-Viola-Brewer-keQeBQHj] - user deleted


  3 passing (5s)
```

4. Commit and push your changes to see that they're still passing on Drone too

</p></details>