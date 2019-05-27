# Module 3: Securing API with IAM authorization

## Securing get-restaurants endpoint with IAM-authorization

**Goal:** Protect the `get-restaurants` endpoint is with IAM

<details>
<summary><b>Use AWS_IAM authorizer for get-restaurants endpoint</b></summary><p>

1. In the `terraform/apigateway.tf` file, look for the resource `aws_api_gateway_method.get_restaurants_get` and **replace** it with the following

```terraform
resource "aws_api_gateway_method" "get_restaurants_get" {
  rest_api_id   = "${aws_api_gateway_rest_api.api.id}"
  resource_id   = "${aws_api_gateway_resource.get_restaurants.id}"
  http_method   = "GET"
  authorization = "AWS_IAM"
}
```

The change here is to set `authorization` to `AWS_IAM`.

</p></details>

<details>
<summary><b>Add execute-api:Invoke to the IAM execution role</b></summary><p>

1. We now need to give the `get-index` function permission to execute the `/restaurants` endpoint. To do that, open `terraform/get-index.tf` and **add** to the following to **the end of the file**

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
    }
  ]
}
EOF
}

resource "aws_iam_role_policy_attachment" "get_index_lambda_apigateway_policy" {
  role       = "${aws_iam_role.get_index_lambda_role.name}"
  policy_arn = "${aws_iam_policy.get_index_lambda_apigateway_policy.arn}"
}
```

This gives the function permission to execute the `GET /restaurants` endpoint, which is not protected by `AWS_IAM`.

</p></details>

<details>
<summary><b>Signing request with IAM role</b></summary><p>

Finally, we need to update our code for the `get-index` function and sign the HTTP request with its IAM credentials. To do that, we can use the `aws4` library.

1. Install `aws4` as dependency

`npm install --save aws4`

2. Modify `get-index.js` to the following

```javascript
const fs = require("fs")
const Mustache = require('mustache')
const http = require('superagent-promise')(require('superagent'), Promise)
const aws4 = require('aws4')
const URL = require('url')

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
  const url = URL.parse(restaurantsApiRoot)
  const opts = {
    host: url.hostname,
    path: url.pathname
  }

  aws4.sign(opts)

  return (await http
    .get(restaurantsApiRoot)
    .set('Host', opts.headers['Host'])
    .set('X-Amz-Date', opts.headers['X-Amz-Date'])
    .set('Authorization', opts.headers['Authorization'])
    .set('X-Amz-Security-Token', opts.headers['X-Amz-Security-Token'])
  ).body
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

3. Deploy the project

`./build.sh deploy dev`

Reload the `index.html` and it should still work. But if you curl the `/restaurants` endpoint you should see

```json
{
  "message": "Missing Authentication Token"
}
```

</p></details>