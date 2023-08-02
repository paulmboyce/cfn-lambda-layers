
## Package template and resources:


### References:

- [lambda layers](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-lambda-layerversion.html)
- [Using AWS CloudFormation with layers](https://docs.aws.amazon.com/lambda/latest/dg/layers-cfn.html)
- [Reference local files in CloudFormation template](https://catalog.workshops.aws/cfn101/en-US/intermediate/templates/package-and-deploy#reference-local-files-in-cloudformation-template) - this same idea is then extended to Lambda Layer
- [Creating a .zip deployment package with dependencies](https://docs.aws.amazon.com/lambda/latest/dg/nodejs-package.html)

IMPORTANT: Note how "Code" points to a folder, instead of S3 bucket. 
The cloudformation package command will then replace with S3 paths.

```
Type: AWS::Lambda::Function
  Properties:
    ...
    Code: lambda/ # <<< This is a local directory
    ...
```


### create layer (PROD deps only):

NOTE: omit=dev is npm version specific, [see here](https://stackoverflow.com/questions/9268259/how-do-you-prevent-install-of-devdependencies-npm-modules-for-node-js-package)

```
cd HelloWorldFunction/
rm -rf node_modules
npm ci  --omit=dev
rm -rf ../layers/layer1/nodejs/node_modules
mv node_modules ../layers/layer1/nodejs
rm -rf node_modules
cd ../layers/layer1/nodejs/node_modules
ls -la
cd ../../../../

```

#### Our structure is:

```
tree -L 2
.
.
├── functions
│   └── index.js
├── HelloWorldFunction
│   ├── app.mjs
│   ├── package.json
│   └── package-lock.json
├── layers
│   └── layer1
│       └── nodejs
│           └── node_modules
├── pack
│   └── packaged-template.yaml
├── README.md
└── template.yaml
```


### make a deployment folder on S3 (optional):

```
aws s3 mb s3://package-lambda-2023 --region eu-west-1
```

### create package folder for packaged version of template:

```
mkdir pack
```

### package resources - zips and pushes to S3, and adds S3 paths to template:

```
aws cloudformation package \
    --template-file template.yaml \
    --s3-bucket package-lambda-2023 \
    --s3-prefix "v0.0.1"  \
    --output-template-file pack/packaged-template.yaml
```

### deploy resources - zips and pushes changes to S3, and adds S3 paths to template:
```
aws cloudformation deploy \
    --template-file pack/packaged-template.yaml \
    --stack-name stack-lambda-2023 \
    --capabilities CAPABILITY_IAM
```




### Lambda layer and function - all configured as external assets

#### Sample Cloudformation:

```
  MyLambdaLayer:
    Type: 'AWS::Lambda::LayerVersion'
    Properties:
      LayerName: my-lambda-layer
      Description: My Lambda Layer
      Content: ./layers/HelloWorldFunction # <<< This is a local directory
      CompatibleRuntimes:
        - nodejs18.x
      
  MyLambdaFn:
    Type: 'AWS::Lambda::Function'
    Properties:
      FunctionName: cfn-workshop-node-function
      Description: Node Function to do stuff
      Runtime: nodejs18.x
      Role: !GetAtt LambdaExecutionRole.Arn
      Handler: app.lambdaHandler
      Code: ./HelloWorldFunction # <<< This is a local directory
      Timeout: 15
      MemorySize: 128
      Layers:
        - !Ref MyLambdaLayer

```
