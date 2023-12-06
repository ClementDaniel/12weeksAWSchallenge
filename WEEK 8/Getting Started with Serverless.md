# Overview
We'll build a straightforward "To-do" web application in this session that uses an API to save and retrieve tasks from a NoSQL cloud database. Additionally, machine learning is integrated to recognize and annotate things in photos that are linked to tasks automatically.

Amazon Lambda, Amazon API Gateway, Amazon DynamoDB, Amazon Simple Storage Service (S3), and AWS Amplify Console are the AWS services that were utilized in the development of the application.   
The static web resources, which are loaded in the user's browser and include HTML, CSS, JavaScript, and image files, are continuously deployed and hosted by Amplify Console. 
Data is sent and received from a public backend API created using Lambda and API Gateway by JavaScript running in the browser. The API's Lambda function can store data in a persistence layer made available by DynamoDB. Images that are uploaded are stored on S3. Lastly, items in those photos are identified and labeled using Amazon Rekognition.  


 ![FZs5ExWXEAAnadm](https://github.com/ClementDaniel/12weeksAWSchallenge/assets/96403532/35d0b2af-7dcf-428d-add5-1f62a0c8caae)

# Step 0: Prerequisites
First, create a new Cloud9 environment and open it up. Then, we need to install the `Node.js v16` version, which is compatible with this workshop and not use the preinstalled one on Cloud9.  

Run the following command to ensure you have the latest version of the Node.js Version Manager (nvm):
```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.2/install.sh | bash
```
Install Node.js v16 "Gallium":
```bash
nvm install 'lts/gallium'
```
Then, set this version as the default Node.js version:
```bash
nvm alias default 'lts/gallium'
```
![im1] ![287259221-7ce6b8a9-ccfb-4b05-8e8a-b1924c0a5035](https://github.com/ClementDaniel/12weeksAWSchallenge/assets/96403532/ff6fbc66-2cbe-4f6d-aab8-a049b8f3fbd1)

Clone the git repo of this workshop with the following commands:
```bash
cd ~/environment
git clone https://github.com/aws-samples/serverless-tasks-webapp
```
Make sure we're using the latest version of the AWS CLI by running this command:
```bash
pip install --user --upgrade awscli aws-sam-cli
```
To avoid running into free space issues, we will resize the the Amazon EBS volume to 20 GiB by running the following script:
```bash
cd serverless-tasks-webapp
bash resize.sh 20
df -h
```
As a last step, we'll install and configure the Amplify CLI using the following commands:
```sh
npm i -g @aws-amplify/cli
amplify configure
```
Amplify CLI will ask you to create an IAM user with the permission policy `AdministratorAccess-Amplify`. Finish the user creation then copy the Access Key ID and Secret Access Keys 
into the terminal.

# Step 1: Build a serverless backend
We start by defining a DynamoDB table using Serverless Application Model (SAM).  
Navigate to the file named `sam/template.yaml` and add the following code to the `Resources` section.
```yaml
  # Create DynamoDB table
  TasksTable:
    Type: AWS::DynamoDB::Table
    Properties:
      AttributeDefinitions:
        - AttributeName: "user"
          AttributeType: "S"
        - AttributeName: "id"
          AttributeType: "S"
      KeySchema:
        - AttributeName: "user"
          KeyType: "HASH"
        - AttributeName: "id"
          KeyType: "RANGE"
      BillingMode: PAY_PER_REQUEST
```
Similarly, define the Lambda function in the SAM `template.yaml` template file by inserting the following code under the `Resources` section:

```yaml
  # CreateTask Lambda Function
  CreateTaskFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: src/handlers/createTask
      Handler: app.handler
      Policies:
        - DynamoDBCrudPolicy:
            TableName: !Ref TasksTable
      Environment:
        Variables:
          TASKS_TABLE: !Ref TasksTable
      Events:
        PostTaskFunctionApi:
          Type: Api
          Properties:
            RestApiId: !Ref TasksApi
            Path: /tasks
            Method: POST
            Auth:
              Authorizer: MyLambdaTokenAuthorizer
```
Create a Lambda function by editing the file named `sam/src/handlers/createTask/app.js` and pasting the following code into it:
```js
const { DynamoDBClient } = require('@aws-sdk/client-dynamodb')
const { DynamoDBDocumentClient, PutCommand } = require('@aws-sdk/lib-dynamodb')
const uuid = require('uuid')

const ddbClient = new DynamoDBClient()
const ddbDocClient = DynamoDBDocumentClient.from(ddbClient)
const tableName = process.env.TASKS_TABLE

exports.handler = async (event) => {
  console.info('received:', event)

  const body = JSON.parse(event.body)
  const user = event.requestContext.authorizer.principalId
  const id = uuid.v4()
  const title = body.title
  const bodyText = body.body
  const createdAt = new Date().toISOString()

  let dueDate = createdAt

  if ('dueDate' in body) {
    dueDate = body.dueDate
  }

  const params = {
    TableName: tableName,
    Item: { user: `user#${user}`, id: `task#${id}`, title: title, body: bodyText, dueDate: dueDate, createdAt: createdAt }
  }

  console.info(`Writing data to table ${tableName}`)
  const data = await ddbDocClient.send(new PutCommand(params))
  console.log('Success - item added or updated', data)

  const response = {
    statusCode: 200,
    headers: {
      'Access-Control-Allow-Origin': '*'
    },
    body: JSON.stringify(data)
  }
  return response
}
```

# Step 2: Configure API authorization
In this step, we will configure a Lambda authorizer to control access to the API. However, in this workshop we'll simply mock the identity provider.  
Edit the file named `sam/src/auth/app.js` and paste the following code into it:
```js
const jwt = require('njwt')

exports.handler = function (event, context, callback) {
  console.info('received:', event)
  const token = event.authorizationToken.split(' ')[1]
  jwt.verify(token, 'secretphrase', (err, verifiedJwt) => {
    if (err) {
      console.log(err.message)
      callback('Error: Invalid token')
    } else {
      console.log(`Verified token: ${verifiedJwt}`)
      const resource = `${event.methodArn.split('/', 2).join('/')}/*`
      const policy = generatePolicy(verifiedJwt.body.sub, 'Allow', resource)
      console.log(`Generated policy: ${JSON.stringify(policy)}`)
      callback(null, policy)
    }
  })
}

const generatePolicy = function (principalId, effect, resource) {
  const authResponse = {}

  authResponse.principalId = principalId
  if (effect && resource) {
    const policyDocument = {}
    policyDocument.Version = '2012-10-17'
    policyDocument.Statement = []
    const statementOne = {}
    statementOne.Action = 'execute-api:Invoke'
    statementOne.Effect = effect
    statementOne.Resource = resource
    policyDocument.Statement[0] = statementOne
    authResponse.policyDocument = policyDocument
  }

  authResponse.context = {
    userId: 1,
    createdAt: new Date().toISOString()
  }
  return authResponse
}
```
# Step 3: Build and deploy a web application
Using the SAM CLI, we'll build and deploy the serverless application stack: 
```bash
cd ~/environment/serverless-tasks-webapp/sam
sam build
sam deploy --guided
```
Open `/webapp/src/main.js` in an editor, and paste in the API gateway endpoint URL generated as output of the CloudFormation stack for the value of `axios.defaults.baseURL` variable.

![im2](https://github.com/xhelma/12weekawsworkshopchallenge/assets/97184575/7b92ea16-5a27-4584-af33-5befff6cf3e9)

Next, initialize Amplify from the root of the web project `/webapp`:
```sh
cd ~/environment/serverless-tasks-webapp/webapp
amplify init
```
Then, build the web application:
```sh
cd ~/environment/serverless-tasks-webapp/webapp
npm install
npm run build
```
Now, we'll deploy the web app manually by first adding hosting to Amplify. Select `Hosting with Amplify Console` as plugin to use for hosting and `Manual` for the type of deployment.
```sh
amplify add hosting
```
Finally, run the following command to publish the app. 
```sh
amplify publish
```
Browse to the resulting URL app:
![im3] ![287290529-80b85257-dc45-469d-be34-00081c4a5103](https://github.com/ClementDaniel/12weeksAWSchallenge/assets/96403532/3e0a7a44-d1b8-42c2-adbc-5af8b327c3b4)

Use any username and password to access the app since no authentication is configured in the backend.
(![287291084-1ecd310a-55d8-4138-9af0-7c422f8aa3fd](https://github.com/ClementDaniel/12weeksAWSchallenge/assets/96403532/2b72165d-6248-4122-96a5-7ae49f60787e)
)

# Step 4: Test the application
Create a new task by providing a title and a body. 
(![287299713-4450a33f-cb84-4a24-b3cb-2939fa83dc93](https://github.com/ClementDaniel/12weeksAWSchallenge/assets/96403532/99f1e05b-1012-4474-acfd-0caa42d0d18f)
)
You can view the same task from the DynamoBD table:
(![287300611-7b800523-4a6c-4658-8727-ae2feea565f4](https://github.com/ClementDaniel/12weeksAWSchallenge/assets/96403532/2609d17a-b4d1-4aa7-a6c4-a05d3f67ff3a)
)

# Step 5: Configure image metadata extraction
In this step, we will add a feature to detect objects in an uploaded image and apply labels to the detected objects.  
Let's configure Amazon Rekognition Integration by pasting the following code into the file named `sam/src/handlers/detectLabels/app.js`:
```sh
const { DynamoDBClient } = require('@aws-sdk/client-dynamodb')
const { DynamoDBDocumentClient, UpdateCommand } = require('@aws-sdk/lib-dynamodb')
const { RekognitionClient, DetectLabelsCommand } = require('@aws-sdk/client-rekognition')
const ddbClient = new DynamoDBClient()
const ddbDocClient = DynamoDBDocumentClient.from(ddbClient)
const rekognitionClient = new RekognitionClient()
const tableName = process.env.TASKS_TABLE

exports.handler = async (event) => {
  console.info(JSON.stringify(event, null, 2))

  const bucket = event.Records[0].s3.bucket.name
  const key = decodeURIComponent(event.Records[0].s3.object.key.replace(/\+/g, ' '))

  const user = key.split('/')[0]
  const taskId = key.split('/')[1]

  const command = new UpdateCommand({
    TableName: tableName,
    Key: { user: `user#${user}`, id: `task#${taskId}` },
    UpdateExpression: 'SET upload = :u',
    ExpressionAttributeValues: {
      ':u': `s3://${bucket}/${key}`
    },
    ReturnValues: 'UPDATED_NEW'
  })

  console.log(`UpdateCommand: ${JSON.stringify(command, null, 2)}`)

  try {
    console.log(`Saving upload for task ${taskId}: s3://${bucket}/${key}`)
    const data = await ddbDocClient.send(command)
    console.log('UpdateItem succeeded:', JSON.stringify(data, null, 2))
  } catch (err) {
    console.log('Unable to update item. Error JSON:', JSON.stringify(err, null, 2))
    throw err
  }

  console.log(`Detecting labels for bucket ${bucket} and key ${key}`)

  const imageParams = {
    Image: {
      S3Object: {
        Bucket: bucket,
        Name: key
      }
    }
  }

  const labelData = await rekognitionClient.send(
    new DetectLabelsCommand(imageParams)
  )
  console.log('Success, labels detected.', labelData)
  const labels = []
  for (let j = 0; j < labelData.Labels.length; j++) {
    const name = labelData.Labels[j].Name
    labels.push(name)
  }

  console.log(`Label data: ${JSON.stringify(labels)}`)

  const updateLabelsCommand = new UpdateCommand({
    TableName: tableName,
    Key: { user: `user#${user}`, id: `task#${taskId}` },
    UpdateExpression: 'SET labels = :s',
    ExpressionAttributeValues: {
      ':s': labels
    },
    ReturnValues: 'UPDATED_NEW'
  })

  try {
    console.log(`Saving labels for task ${taskId}: ${labels}`)
    const data = await ddbDocClient.send(updateLabelsCommand)
    console.log('UpdateItem succeeded:', JSON.stringify(data, null, 2))
  } catch (err) {
    console.log('Unable to update item. Error JSON:', JSON.stringify(err, null, 2))
    throw err
  }

  const response = {
    statusCode: 200,
    headers: {
      'Access-Control-Allow-Origin': '*'
    },
    body: JSON.stringify(labels)
  }
  return response
}
```
We need to add new `AWS::Serverless::Function` resource to `sam/template.yaml` under the `Resources` section. Here's the code for it:
```sh
# DetectLables Lambda Function
  DetectLabelsFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: src/handlers/detectLabels
      Environment:
        Variables:
          TASKS_TABLE: !Ref TasksTable
      Events:
        ObjectCreatedEvent:
          Type: S3
          Properties:
            Bucket: !Ref UploadsBucket
            Events: s3:ObjectCreated:*
      Handler: app.handler
      Policies:
        - DynamoDBCrudPolicy:
            TableName: !Ref TasksTable
        - RekognitionDetectOnlyPolicy: {}
        - Version: 2012-10-17
          Statement:
            - Effect: Allow
              Action: s3:GetObject*
              Resource: !Sub "arn:aws:s3:::uploads-${AWS::StackName}-${AWS::Region}-${AWS::AccountId}*"
      Runtime: nodejs14.x
```
Finally, build and deploy the changes to AWS:
```sh
cd ~/environment/serverless-tasks-webapp/sam
sam build
sam deploy
```
Let's test our app by uplaoding this image file as an attachement to a task.
 ![download](https://github.com/ClementDaniel/12weeksAWSchallenge/assets/96403532/c1612671-1c0d-487a-9e18-60ec91aee5d5)



















