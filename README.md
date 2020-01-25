
## Task Definition
Provide endpoint to create devices, on a hourly base update them if there was a request.

## Services to use
* Api Gateway (2x POST - create-device and update-device)
* Route 53 (Optional - Stable endpoint in case of redeplyoment)
* Lambda
* StepFunctions, Optional - in case of the task gets more compilcated (Keep Lambdas simple. In this case check StepFunction concurrent execution limit.)
* CloudWatch Logs - logging
* CloudWatch Alarms - easy way to monitor: let's setup an alarm when given Lambdas / Stepfunctions fail. Do not handle exceptions, on given conditons we can raise exceptions.
* CloudWatch Events - Start execution periodically
* DynamoDB - storage.
* SQS - to handle concurrency, scales well.
* IAM - Roles, priviledges.
* Cloudformation - IaaC; StackSets for multi account deployment
* S3 - (optional) SPA which consumes the interface
* SNS - Notification
* VPC, Optional. Private deployment? VPN / Direct Connect? Corporate Network? 

## Solution
1. Let's create the API GW.
2. API GW Security: API Key (think about key rotation), Create Model Validation for post requests, Authentication (IAM Authentication / Basic Auth (uname, pw)). Systems Manager Parameter Store can be a good place to store credentials.
Enable API GW logging.
* Q: Who will be the user (users) of the API? Internal / External? 
* Q: Do we want to put everyting into a VPC?
* Q: Which type of authentication?
* Q: From where should be the endpoints accessible?
3. Route 53 endpoint ? - Optional, based on the requirements. If we redeploy our template the endpint will change. This can cause problems.
4. Connect Lambda with given post endpoint.
5. /create-device [Method::Post]: Save it to dynamoDB. Quick win solution - make device-name unique.
https://stackoverflow.com/questions/34921224/case-insensitive-query-in-dynamo-db  
Validate the request. Let's make the validation case insensitive.
* TASK: think about the Primary key (Sort Key ?! )
6. update-device: put the data into SQS ?!Set some kind of visibility timeout or some kind of custom polling ?!. We want to execute only once in a hour.
* Task - think about the logging of the updates.
7. When time comes poll the data from sqs, execute. 
If the SAP request takes longer increase the Lambda visibility timout, make it async, raise Exception if smthg fails.
8. To deploy to multiple account we can use StackSets or CodePipeline. Possible it is easier to use StackSets if we have everything in CF already. Firstly we need to deploy a role to a given account, which trusts our "master".
9. GitLab, Personal Preference: I'd separate lambdas from the CF. Let's setup a GitLab Pipeline - on a push event generates deployment packages of our lambdas (only zip them :D ) and pushes it to S3. In CF let's just reference on these S3 locations.
10. Create and SNS topic so we will be informed on custom events (like fail). Let's create and SNS for our customers (if there are). This way we can inform them if they send us invalid data / they will know once their task is done / device is created.
11. Think about testing, mock endpoint for customers if needed. How to roll out changes? Will be future changes? How often? Is it worth to build a PipeLine around it?