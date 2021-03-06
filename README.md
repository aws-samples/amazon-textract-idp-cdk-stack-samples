# Amazon Textract IDP Stack Samples
<!--BEGIN STABILITY BANNER-->

---

![Stability: Experimental](https://img.shields.io/badge/stability-Experimental-important.svg?style=for-the-badge)

> All classes are under active development and subject to non-backward compatible changes or removal in any
> future version. These are not subject to the [Semantic Versioning](https://semver.org/) model.
> This means that while you may use them, you may need to update your source code when upgrading to a newer version of this package.

---
<!--END STABILITY BANNER-->

# Deployment

This is a collection of sample workflows designed to showcase the usage of the [Amazon Textract IDP CDK Constructs](https://github.com/aws-samples/amazon-textract-idp-cdk-constructs/)

This project is set up like a standard Python project.  The initialization
process also creates a virtualenv within this project, stored under the `.venv`
directory.  To create the virtualenv it assumes that there is a `python3`
(or `python` for Windows) executable in your path with access to the `venv`
package. If for any reason the automatic creation of the virtualenv fails,
you can create the virtualenv manually.

To manually create a virtualenv on MacOS and Linux:

```
python3 -m venv .venv
```

After the init process completes and the virtualenv is created, you can use the following
step to activate your virtualenv.

```
source .venv/bin/activate
```

If you are a Windows platform, you would activate the virtualenv like this:

```
% .venv\Scripts\activate.bat
```

Once the virtualenv is activated, you can install the required dependencies.

```
pip install -r requirements.txt
```

Now start Docker and login to your Docker account.

The CDK Constructs will build Docker containers during the deploy cycle. 

At this point you can now synthesize the CloudFormation template for this code.

At the moment there are 9 stacks available:

* SimpleSyncWorkflow - very easy setup, calls Textract Sync
* SimpleAsyncWorkflow - easy, but with Async
* SimpleSyncAndAsyncWorkflow  - both async and sync
* PaystubAndW2Spacy - with classification using Spacy
* InsuranceStack - including A2I Construct call
* AnalyzeID - only calling AnalyzeID
* AnalyzeExpense - only calling AnalyzeExpense
* DemoQueries - workflow with calling Textract + Queries for alldocs
* PaystubAndW2Comprehend - using Comprehend classification

```
cdk synth (sample-stack-name)
```

To add additional dependencies, for example other CDK libraries, just add
them to your `setup.py` file and rerun the `pip install -r requirements.txt`
command.

Deploy the stack with following command
```
cdk deploy (sample-stack-name)
```

if you have not already setup your environment, run below command before cdk deploy
```
cdk bootstrap aws://<account>/<region>
```

You are all set with your deployment!

## Useful commands

 * `cdk ls`          list all stacks in the app
 * `cdk synth`       emits the synthesized CloudFormation template
 * `cdk deploy`      deploy this stack to your default AWS account/region
 * `cdk diff`        compare deployed stack with current state
 * `cdk docs`        open CDK documentation

Enjoy!
# Sample Workflows

## Paystub And W2 Comprehend

This sample showcases a number of components, including classification using Comprehend and routing based on the document type, followed by configuration based on the document types.
It is called Paystub and W2, because those are the ones configured in the RouteDocType and the DemoIDP-Configurator.

At the moment it does single page, once we add the splitter we can have this flow as part of a map.

Here is the flow:

<img alt="PaystubAndW2Comprehend" width="800px" src="images/PaystubAndW2Comprehend_graph.svg" />

Check the API definition for the Constructs at: https://github.com/aws-samples/amazon-textract-idp-cdk-constructs/blob/main/API.md

From top to bottom:

1. DemoIDP-Decicer - Takes in the manifest, usually a link to a file on S3. Look at the bottom and the lambda_start_step_function code to see how the workflow is triggered.
1. NumberOfPagesChoise - Choice state failing the flow if the number of pages > 1
1. Randomize and Randomize2 - Just a function to generate a random number 0-100, so we can route between sync and async for demo purposes. (Will increase throughput as well, but that is not the main purpose, it is just to demo both async and sync in one flow)
1. TextractAsync - calls Textract async through Start*. When passed features are passed, will call AnalyzeDocument, otherwise DetectText. The flow is configured to first only call with text. The process abstract calling Textract and waiting for the SNS Notification and output to OutputConfig
1. TextractSync - similar to TextractAsync, but calling the Textract sync API (DetectText, AnalyzeDocument)
1. TextractAsyncToJSON2 - TextractAsync will store paginated JSON files to OutputConfig. This step combines them to one JSON.
1. GenerateText - Takes the Textract JSON and outputs all LINES into a text file.
1. Classification - Uses the generated text file from GenerateText and sends to the Comprehend endpoint defined by the ARN.
1. RouteDocType - routes based on the classification result, aka the document type. Unless it is ID, Expense or AWS_OTHER or not known, we send to further Textract processing
1. DemoIDP-Configurator - based the document type, pulls configuration from DynamoDB what to call Textract with (e. g. specific queries and/or with forms and/or with tables) 
1. Then we repeat essentially the calls to Textract like at the top, but this time we do have the configuration set with queries, forms and/or tables
1. GenerateCsvTask - output is one CSV from queries and forms information to a structure that includes also confidence scores and bounding box information
1. CsvToAurora - sends the generated CSV to a Serverless Aurora RDS Cluster

The Aurora RDS Cluster runs in a private VPC. To get there, check the commented section for EC2 in the sample stack. Put in your setting for Security Groups, AMI and keypair. (We'll make it easier in the future)

## Simple Async Workflow

Very basic workflow to demonstrate AsyncProcessing. This out-of-the-box will only call with DetectText, generating OCR output.
When you are interested in running specific queries or features like forms or tables on a set of documents, look at [DemoQueries](#demo-queries)

<img alt="SimpleAsyncWorkflow" width="400px" src="images/SimpleAsyncWorkflow_graph.svg" />

## Demo Queries

Basic workflow to demonstrate how Sync and Async can be routed based on numberOfPages and numberOfQueries and how the workflow can be triggered with queries. 
Calls AnalyzeDocument with the 2 sample queries. Obviously, modify to your own needs. The location in the code where queries are configed is [here](https://github.com/aws-samples/amazon-textract-idp-cdk-stack-samples/blob/471905e06786e0def269695d5585b39a0d77b825/lambda/start_queries/app/start_execution.py#L52) when kicking off the Step Functions workflow.
The GenerateCsvTask will output one CSV file to S3 with key/value, confidence scores and bounding box information based on the forms and queries output.

<img alt="DemoQueries" width="400px" src="images/DemoQueries_graph.svg" />

## Insurance

Simple flow including A2I 

<img alt="Insurance" width="300px" src="images/Insurance_graph.svg" />

## Paystub And W2 Spacy

Similar to the Comprehend classification task, but implemented using Spacy textcat and deployed as a Lambda container instead of using a Comprehend endpoint.

<img alt="PaystubAndW2Spacy_graph" width="800px" src="images/PaystubAndW2Spacy_graph.svg" />

Simple example of a flow only calling synchronous Textract for DetectText.

## Simple Sync Workflow 

<img alt="PaystubAndW2Spacy_graph" width="400px" src="images/SimpleSyncWorkflow_graph.svg" />

Simple flow calling the Textract AnalzyeID API.

## Analyze ID

<img alt="AnalyzeID_graph" width="300px" src="images/AnalyzeID_graph.svg" />

Simple flow calling the Textract AnalyzeExpense API.

## Analyze Expense

<img alt="AnalyzeExpense_graph" width="300px" src="images/AnalyzeExpense_graph.svg" />

# Create your own workflows

Take a look at the sample workflows. Copy one as a starting point and go from there.

