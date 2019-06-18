# Module 8: Building CI/CD pipeline with Drone

## Create a CI/CD pipeline with Drone

<details>
<summary><b>Sign up to Drone cloud</b></summary><p>

1. Head over to [https://cloud.drone.io/](https://cloud.drone.io/).

2. Click `Log In`.

3. Log in with your Github credentials.

4. `Activate` your Github repo for the demo app.

5. In the project root, add a new file called `.drone.yml`, this is where you define your CI/CD pipeline.

6. Copy the following into the newly created `.drone.yml` file:

```yml
kind: pipeline
name: default

steps:
  - name: build
    image: node:12.4.0
    commands:
      - npm ci
    when:
      event:
        - push
```

This is just the skeleton for our pipeline, we will fill it in later.

Couple of things to note here:

* Every step of the pipeline is implemented using `Docker`.
* We're using `npm ci` to restore packages instead of `npm install`. This restores the locked version of our dependencies (and their dependencies). This is important because `npm install` installs the most recent and compatible version of our dependencies instead, which allows small dependency updates to creep in.
* You can optional specify which step is executed using the `when` attribute. For more information about how Drone works, please refer to the official [documentation](https://docs.drone.io/).

</p></details>

<details>
<summary><b>Add S3 backend to Terraform</b></summary><p>

So far, our Terraform state is only stored locally. We need to share the Terraform state with the CI/CD pipeline. To do that, we can add a S3 backend for Terraform.

1. In the `terraform` folder, add a new file called `backend.tf`.

2. Copy the following into the new `backend.tf` file, and replace the placeholder `<PUT YOUR NAME HERE>`. The bucket name should match what you created in module 01.

```terraform
terraform {
  backend "s3" {
    bucket="ynap-production-ready-serverless-<PUT YOUR NAME HERE>"
    key="terraform.tfstate"
    region="us-east-1"
  }
}
```

3. Staying in the `terraform`, run the command `terraform init`.

Now, the Terraform state should be synch'd to S3. You can check the S3 bucket in the AWS console to confirm. However, Drone would need access to AWS to initialize the state. We can configure AWS credentials by adding the necessary secrets to the repo settings.

4. In Drone, navigate to the `SETTINGS` page for the repo. In the `Secrets` section, add the following secrets:

* `AWS_ACCESS_KEY_ID`
* `AWS_SECRET_ACCESS_KEY`

![](/images/mod08-001.png)

We'll reference them later.

</p></details>

<details>
<summary><b>Configure Drone steps</b></summary><p>

As we discussed at the start of the module, I prefer to have a simple `build.sh` script to encapsulate the build steps. To do that, we require a Docker image that has all the dependencies. Usually, I'll create the custom image and registered in a private Docker repository. However, for this workshop, that's overkill!

So instead, we'll use the official Node and Terraform images. As a result, we won't be able to use the `build.sh` script on CI, so we'll need something else to package and upload our artefacts. But first...

1. Add an integration test step. **Replace** the `.drone.yml` with the following:

```yml
kind: pipeline
name: default

steps:
  - name: integration test
    image: node:12.4.0
    commands:
      - npm ci
      - npm run test
    environment:
      AWS_ACCESS_KEY_ID:
        from_secret: AWS_ACCESS_KEY_ID
      AWS_SECRET_ACCESS_KEY:
        from_secret: AWS_SECRET_ACCESS_KEY
    when:
      event:
        - push

  - name: build
    image: node:12.4.0
    commands:
      - npm ci
    when:
      event:
        - push
```

Notice that we referenced the two secrets we added earlier, and assigned them to the environment variables with the same time. Commit and push the change to Github. It should trigger a Drone build, and the integration tests should be passing.

```
+ npm ci
added 221 packages in 2.331s
+ npm run test

> production-ready-serverless-workshop-ynap-demo@1.0.0 test /drone/src
> TEST_MODE=handler mocha tests/test_cases --reporter spec --timeout 5000



When we invoke the GET / endpoint
AWS credential loaded
loading index.html...
loaded
✓ Should return the index page with 8 restaurants (2290ms)

When we invoke the GET /restaurants endpoint
✓ Should return an array of 8 restaurants (335ms)

When we invoke the POST /restaurants/search endpoint with theme 'cartoon'
✓ Should return an array of 4 restaurants (182ms)


3 passing (3s)
```

2. The existing `build.sh` script will not work because the `node:12.4.0` image does not have the required commands. But, since we already have Node, we can replace it with a simple Node script.

In the project root, add a new file called `build.js`.

3. Copy the following into the newly created `build.js` file:

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

archive.finalize()
```

This script zips the `functions`, `static` and `node_modules` file to produce the artefact. Then uploads the artefact to S3, as we did before. But we also need to let the next step (which will use the official Terraform image and doesn't have access to Node) know the S3 key for the artefact.

The only way to share data between different Drone steps is through the disk. Hence why this script also writes the MD5 of `workshop.zip` to a local file so it can be read by the `deploy` step later.

4. Now we have a Node script for building and uploading the project artefact, let's add a `build` step to the pipeline. **Replace** the `.drone.yml` file with the following and replace the `<PUT YOUR NAME HERE>` placeholder:

```yml
kind: pipeline
name: default

steps:
  - name: integration test
    image: node:12.4.0
    commands:
      - npm ci
      - npm run test
    environment:
      AWS_ACCESS_KEY_ID:
        from_secret: AWS_ACCESS_KEY_ID
      AWS_SECRET_ACCESS_KEY:
        from_secret: AWS_SECRET_ACCESS_KEY
    when:
      event:
        - push

  - name: build
    image: node:12.4.0
    commands:
      - node build.js "<PUT YOUR NAME HERE>"
    environment:
      AWS_ACCESS_KEY_ID:
        from_secret: AWS_ACCESS_KEY_ID
      AWS_SECRET_ACCESS_KEY:
        from_secret: AWS_SECRET_ACCESS_KEY
    when:
      event:
        - push
```

Commit, and push the change to Github. It should trigger another Drone build. The `build` step should be passing, and you should see something like the following:

```
+ node build.js "yancui"
deployment bucket is ynap-production-ready-serverless-yancui
deployment artefact created
uploading to S3 as workshop/7bb59b96d2268698a46e9348c1075221.zip
artefact has been uploaded to S3
```

5. Now let's add the `deploy` step. **Replace** the `.drone.yml` file with the following, and replace the `<PUT YOUR NAME HERE>` placeholder (in 2 places):

```yml
kind: pipeline
name: default

steps:
  - name: integration test
    image: node:12.4.0
    commands:
      - npm ci
      - npm run test
    environment:
      AWS_ACCESS_KEY_ID:
        from_secret: AWS_ACCESS_KEY_ID
      AWS_SECRET_ACCESS_KEY:
        from_secret: AWS_SECRET_ACCESS_KEY
    when:
      event:
        - push

  - name: build
    image: node:12.4.0
    commands:
      - node build.js "<PUT YOUR NAME HERE>"
    environment:
      AWS_ACCESS_KEY_ID:
        from_secret: AWS_ACCESS_KEY_ID
      AWS_SECRET_ACCESS_KEY:
        from_secret: AWS_SECRET_ACCESS_KEY
    when:
      event:
        - push

  - name: deploy
    image: hashicorp/terraform:0.11.12
    commands:
      - MD5=$(cat workshop_md5.txt)
      - cd terraform
      - terraform init
      - terraform apply --var "my_name=<PUT YOUR NAME HERE>" --var "file_name=$MD5" -auto-approve
    environment:
      AWS_ACCESS_KEY_ID:
        from_secret: AWS_ACCESS_KEY_ID
      AWS_SECRET_ACCESS_KEY:
        from_secret: AWS_SECRET_ACCESS_KEY
    when:
      event:
        - push
```

Commit, and push the change to Github to trigger another Drone build.

6. Once the deployment is done, we'd want to run the acceptance test to make sure everything is still working end-to-end. **Replace** the `.drone.yml` file with the following, and replace the `<PUT YOUR NAME HERE>` placeholder (in 2 places):

```yml
kind: pipeline
name: default

steps:
  - name: integration test
    image: node:12.4.0
    commands:
      - npm ci
      - npm run test
    environment:
      AWS_ACCESS_KEY_ID:
        from_secret: AWS_ACCESS_KEY_ID
      AWS_SECRET_ACCESS_KEY:
        from_secret: AWS_SECRET_ACCESS_KEY
    when:
      event:
        - push

  - name: build
    image: node:12.4.0
    commands:
      - node build.js "<PUT YOUR NAME HERE>"
    environment:
      AWS_ACCESS_KEY_ID:
        from_secret: AWS_ACCESS_KEY_ID
      AWS_SECRET_ACCESS_KEY:
        from_secret: AWS_SECRET_ACCESS_KEY
    when:
      event:
        - push

  - name: deploy
    image: hashicorp/terraform:0.11.12
    commands:
      - MD5=$(cat workshop_md5.txt)
      - cd terraform
      - terraform init
      - terraform apply --var "my_name=<PUT YOUR NAME HERE>" --var "file_name=$MD5" -auto-approve
    environment:
      AWS_ACCESS_KEY_ID:
        from_secret: AWS_ACCESS_KEY_ID
      AWS_SECRET_ACCESS_KEY:
        from_secret: AWS_SECRET_ACCESS_KEY
    when:
      event:
        - push

  - name: acceptance test
    image: node:12.4.0
    commands:
      - npm run acceptance
    environment:
      AWS_ACCESS_KEY_ID:
        from_secret: AWS_ACCESS_KEY_ID
      AWS_SECRET_ACCESS_KEY:
        from_secret: AWS_SECRET_ACCESS_KEY
    when:
      event:
        - push
```

Commit and push the change to Github and it should trigger another Drone build.

If everything is still passing then congrats! You have a working CI/CD pipeline :-)

</p></details>
