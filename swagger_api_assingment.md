# multi-API Gateways Setup

[TOC]



## 1. Background Info

- we are developing a microservice-based Business Platform running solely on AWS
- all our deployments are Cloudformation based, we dont deploy Resources into Dev/ Prod Environment via Console
- the stacks are being launched using AWSCLI Commands
- to connect to the backend resources we implement a “multi-API-Gateway under on domain” - approach involving basepathmapping in our Cloudformation stacks 
- the API-Gateway configuration is being delivered by a Swagger (OpenAPI) file which is being uploaded onto an s3 bucket and used during stack deployment as source for the API definition
- the API access is being secured by a Cognito based Authorizer which uses a Cognito pool (Cognito stack)
- we have created an Open API definition for our APIs containing all our resources for the first module (Contact Manager)





## 2. Scope of delivery

| __Results__              | __Description__                                              |
| ------------------------ | ------------------------------------------------------------ |
| Documentation            | A documentation describing the resources and the approach as well as the elaborations on the deployed  AWS extensions |
| Cloudformation templates | Cloudformation templates with the resources working as expected in an automated way (cognito, Api Gateway, IAM, swagger) |



## 3. Details of the assignment

### 3.1 General Focus

- the goal is to split up the provided swagger file into small swagger files
- the swagger files shall be integrated as following:

```yaml
  Api:
    Type: AWS::ApiGateway::RestApi
    Properties:
      Description: .....
      Name: !Ref RestApiName
      ApiKeySourceType: !Ref ApiKeySourceType
      Body:
        Fn::Transform:
          Name: AWS::Include
          Parameters:
            Location: !Sub "s3://bucketname/swagger.yaml"
      EndpointConfiguration:
        Types:
          - !Ref EndpointConfiguration
      FailOnWarnings: true
      MinimumCompressionSize: !Ref minimumCompressionSize
```

- the provided swagger file should look like this:

```yaml
swagger: "2.0"
info:
  version: 1.0.0
  title: Contact Manager - API
  description: >
    This API is intented to enable external users to get information for the path.....

# Specifies a request validator, by referencing a request_validator_name
x-amazon-apigateway-request-validators:
.....

# Specifies the source to receive an API key
x-amazon-apigateway-api-key-source: HEADER

# list of binary media types to be supported by API Gateway
x-amazon-apigateway-binary-media-types:
  - application/json
  - application/xml
  - text/plain; charset=utf-8
  - text/html
  - application/pdf
  - image/png
  - image/gif
  - image/jpeg

securityDefinitions:
  Lambdaauthorizer:
    type: apiKey # Required and the value must be "apiKey" for an API Gateway API.
    name: Authorization # The name of the header containing the authorization token
    in: header # Required and the value must be "header" for an API Gateway API.
    x-amazon-apigateway-authtype: cognito_user_pools # Specifies the authorization mechanism for the client.
    x-amazon-apigateway-authorizer: # An API Gateway Lambda authorizer definition
      type: cognito_user_pools # Required property and the value must "token"
      providerARNs:
      - Fn::ImportValue: { "Fn::Sub" : "${EnvironmentName}-${Project}-UserPoolARN" }

schemes: [http, https]
consumes:
  - application/json
  - application/xml
  - text/plain; charset=utf-8
  - text/html
  - application/pdf
produces:
  - application/json
  - application/xml
  - image/png
  - image/gif
  - image/jpeg
paths:
  /users/{userId}:
    get:
      description: TBD
      security:
      - Lambdaauthorizer: []
      x-amazon-apigateway-integration:
        uri:
          Fn::Join:
          - ''
          - - 'arn:aws:apigateway:'
            - Ref: AWS::Region
            - ":lambda:path/2015-03-31/functions/"
            - Fn::GetAtt:
              - Lambda
              - Arn
            - "/invocations"
        credentials:
          Fn::ImportValue: {"Fn::Sub" : "${EnvironmentName}-ApiGatewayRoleARN"}
        passthroughBehavior: when_no_match
        httpMethod: POST
        type: aws_proxy
        .....
```



### 3.2 Split up the Swagger file

- split the file we are providing into multiple Swagger files and enrich them with the appropriate AWS extension to connect to the backend (serverless) 

**implement the following Pathes as single API Gateway:**

- activities
- attachments
- bookmarks
- categories
- comments
- contracts
- energymeters
- favorites
- groups
- roles
- status
- tags
- tasks
- userprofiles



**implement the following Pathes as multi-path API Gateways:**

- multi-path API **"financials"** contains:
  - bankaccounts
  - bankdetails
  - creditcards
  - financialprofiles
  - sepastandingorders

- multi-path API **"buildings"** contains:
  - apartments
  - apartmentprofiles
  - buildings
  - buildingnetworks
  - entrances
  - historicenergyconsumptions
  - rooms

- multi-path API **"contacts"** multi-path contains:
  - addresses
  - contacts
  - contactnetworks

- multi-path API **"organizations"** contains:
  - organizations
  - organizationidentifiers

- multi-path API **"POD"** contains:
  - pointofdelivery
  - pointofdeliverynetworks



### 3.3 Further details

- the stacks must be created as `yaml` files

- ensure the backend database for our APIs speaks DynamoDB

- integrate 2 API Gateways (see split up above) in a way, that the response is an image (png, jpeg); these API Gateways are:

  - Attachements
  - contracts

- integrate the remaining API Gateways with a Dynamo DB  table in 2 ways (50/50)

  - with Lambda function and  
  - bypass Lambda, as the AWS API Gateway can work directly with AWS DynamoDB API

  to enable us to test the API Gateways

- enable Request Validation in API Gateway (specify request Validator)

- enable Payload Compression for the API

- create a Usage Plan Resource for the API Gateway and an API Key

- enable Cloudwatch logs

- Stage in Swagger is `/v1`

- use API Gateway **Cognito** Authorizers

- enable cross-origin resource sharing (CORS) for selected methods on the resource

- use Client-Side SSL Certificates for Authentication by the Backend/ Configure Backend HTTPS Server to Verify the Client Certificate

- create and Use Usage Plans with API Keys

- we use a custom API domain name

- use the following OpenAPI Extensions and its sub properties to integrate AWS API Gateway and Swagger API

  - `x-amazon-apigateway-authorizer`
  - `x-amazon-apigateway-authtype`
  - `x-amazon-apigateway-request-validator`
  - `x-amazon-apigateway-integration`
  - `x-amazon-apigateway-any-method Object`
  - `x-amazon-apigateway-binary-media-types Property`
  - `x-amazon-apigateway-gateway-responses`
  - `x-amazon-apigateway-api-key-source`



## 4. Technology Stack

The following technologies are mandatory and cannot be replaced by any other technology similar or not without our explicit approval:

- **Devops-Technologies**
  - AWS API Gateway as the API layer
  - AWS Cognito
  - AWS Lambda
  - AWS DynamoDB
  - Node.js running in Lambda for the code layer of this API
  - OpenAPI definition/ Swagger
  - postman to check the API



## 5. Code repository

- we provide a repository named "serverless_api_rds"
- Fork the repository and work inside your own account
- Give access to `jprivillaso@gmail.com` and `aerioeus@gmail.com` so we can track your work
- arrange your code like this

```shell
--api-masterstack
  |__api_rds.yaml
--nodejs
  |__index.js
--swaggerapis
  |__swagger.yaml
```

- Create a Pull Request and we will review the code



## 6. Provided by us

- Documentation template as reference
- Swagger (OpenAPI) file containing all Pathes for the entire module
- Github Repository to pull from



## 7. Timeline

- Project Start: 24.12.2018
- the timeframe for the project is set to 7 days upon hiring the developer
- meaning the concept must be ready ==30.12.18==













