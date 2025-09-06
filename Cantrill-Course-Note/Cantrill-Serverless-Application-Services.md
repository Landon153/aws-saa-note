# Architecture Deep Dive

## Monolithic Architecture
- Fails together
- Scales together
- Billed together

## Tiered Architecture
- Each tier can be scaled independently
- Each tier connects to each
- Enables horizontal scaling of individual tiers since a single endpoint (LB) is used to direct traffic between the tiers
- Each tier has to be running something for the app to function
  - Ex. Upload expects and REQUIRES Processing to respond

## Event Driven Architecture
- Producers, Consumers, or BOTH
- No constant running or waiting for things
- Producers generate events when something happens
  - Clicks, errors, criteria met, uploads, actions
- Events are delivered to consumers
  - Generally via Event Router
  - Actions are taken and the system returns to waiting
- Mature event-driven architecture only consumes resources while handling events
  - Serverless

# AWS Lambda In-depth – Part 1
- Function-as-a-Service (FaaS) – short running & focused
- Lambda function – a piece of code lambda runs
- Functions use a runtime – e.g. Python 3.8
- Functions are loaded and run in a runtime environment
- The environment has a direct memory (indirect vCPU) allocation
- You are billed for the duration that a function runs
- A key part of Serverless architectures
- When a function is invoked, deployment package is sent to a Lambda environment – 50 MB zipped, 250 MB unzipped
  - Once arrived, executes within the runtime environment
- Common Runtimes – Python, Ruby, Java, Go, C#
- Custom runtimes such as Rust are possible using Layers
- Stateless – every time a function is invoked, a new runtime environment is provided
  - Above is default, can be configured for multiple invocations in a single runtime environment
- You directly control the memory allocated for Lambda functions whereas vCPU is allocated indirectly
  - 128 MB - 10240 MB
- 1769 MB = 1vCPU
- Runtime environment has disk memory mounted – 512 MB storage available as /tmp – default.
  - Can scale up to 10240 MB
- Lambda function can run max 900s (15 minutes)
  - Function timeout
- Use step functions for longer function flows
- Execution Roles – IAM roles assumed by the Lambda function which provides permissions to interact with other AWS services
- Common uses
  - Serverless Applications (S3, API Gateway, Lambda)
  - File Processing (S3, S3 Events, Lambda)
  - Database Triggers (DynamoDB, Streams, Lambda)
  - Serverless CRON (EventBridge/CWEvents + Lambda)
  - Realtime Stream Data Processing (Kinesis + Lambda)

# Lambda In-depth Part 2
## Networking Modes
- Public or VPC

### Public
- By default, functions are given public networking. They can access public AWS services and the public internet
  - E.g. SQS, DynamoDB
- Public networking offers the best performance because no customer specific VPC networking is required
- Public Lambda functions have no access to VPC based services unless public IPs are provided and security controls allow external access
- Typically the desired network configuration

### Private (VPC)
- Lambda is configured to execute in a private subnet
- Private Lambda functions running in a VPC obey all VPC networking rules
  - No access to internet based endpoints unless proper networking and permissions are configured
- Use GWLB or VPC endpoints to access public endpoints (AWS + external)
- NatGW and Internet GW are required for VPC lambdas to access internet resources
- Private lambda needs ec2 network permissions (execution role)

## Security
- Lambda Execution roles are IAM roles attached to lambda functions which control the PERMISSIONS the Lambda function RECEIVES
- Lambda Resource Policy controls WHAT service and accounts can INVOKE lambda functions
  - E.g., sns, s3, or external services
  - Only configurable via the CLI

## Logging
- Lambda uses CloudWatch, CloudWatch Logs & X-Ray
- Logs from Lambda executions – CloudWatch Logs
- Metrics – invocation success/failure, retries, latency – stored in Cloud Watch
- Lambda can be integrated with X-Ray for distributed tracing
- CloudWatch Logs requires permissions via Execution Role – log information into CloudWatch Logs

# Lambda In-depth Part 3
## Invocation
- Synchronous invocation
- Asynchronous invocation
- Event Source mappings

### Synchronous Invocation
- Waits for complete/response pass/fail
- CLI / API invoke a lambda function, passing in data and wait for a response
  - Lambda function responds with data or fails
- Lambda + API Gateway
  - Client communicates with API GW – Proxied to Lambda function
- Result (Success or Failure) – returned during the request
- Errors or Retries have to be handled within the client
- Synchronous Invocation is typically used when a human is directly or indirectly invoking a Lambda function

### Asynchronous Invocation
- No waiting for complete
- Typically used when AWS services invoke Lambda functions
- Example - S3 isn’t waiting for any kind of response. The event is generated and S3 stops tracking
- If processing of the event fails, Lambda will retry between 0 and 2 times (configurable). Lambda handles the retry logic.
- The lambda function code needs to be idempotent reprocessing as a result should have the same end state
  - Not additive/subtractive – use declarative
- Events can be sent to dead letter queues after the automatic (configured) retries repeatedly fail processing.
- Destinations – events processed by lambda functions can be delivered to another destination
  - SQS, SNS, Lambda & EventBridge – where successful or failed events can be sent.

### Event Source Mapping Invocation
- Typically used on streams or queues which don’t support event generation to invoke lambda (polling required)
  - Kinesis
  - DynamoDB Streams
  - SQS
- Source Batches – data that is read (polled) from the source then processed by Lambda – size considerations – lambda times out after 900s (15 mins).
- Because the services aren’t generating events, the data (source batch) requires permissions from the Lambda Execution role
- Event Source Mapping Permissions from the lambda execution role are used by the event source mapping to interact with the event source
- SQS queues or SNS topics can be used for any discarded or failed event batches – DLQ Failed Events (Failed Batch)

## Lambda Versions
- Lambda functions have versions – v1, v2, v3
- A version is the code + the configuration of the lambda function
- It’s immutable – it never changes once published & has its own Amazon Resource Name
- $Latest points at the latest
- Aliases (ex. Dev, Stage, Prod) point at a version – can be changed

## Execution Context
- The environment a lambda function runs in
- A cold start is a full creation and configuration including function code download
  - 100s of ms – slow if interacting with humans, but OK for processing events from S3, SQS, etc.
- A warm start, the same execution context is reused. A new event is passed in but the execution context creation can be skipped.
  - 1-2 ms
- A Lambda invocation can reuse an execution context but has to assume it can’t. If used infrequently contexts will be removed. Concurrent executions will use multiple (potentially new) contexts
- Provisioned Concurrency
  - AWS will create and keep X contexts warm and ready to use – Improving start speeds
- Use the /tmp space to predownload something in the context
  - Or DB connections

# CloudWatch Events and EventBridge
- If X happens, or at Y time(s) - do Z
- EventBridge is CloudWatch Events v2 (*)
- A default Event bus for the account – used by both
- In CloudWatch Events this is the only bus (implicit)
- EventBridge can have additional event busses
- Rules match incoming events – or schedules
- Rule matches – Route the events to 1+ Targets – e.g Lambda
- Event Pattern Rule – event matches In Event Bus
- Schedule Rule – chron format
- Events are JSON format

# Serverless Architecture
- Serverless isn’t one single thing
- You manage few, if any servers – low overhead
- Applications are a collection of small & specialized functions
- Functions run in Stateless and Ephemeral environments – duration billing
- Event-driven – consumption only when being used
- FaaS is used where possible for compute functionality
  - No consistent use for compute
- Managed services are used where possible
  - S3, DynamoDB, OIDC

# Simple Notification Service (SNS)
- Public AWS Service – Network connectivity with Public Endpoint
- Coordinates the sending and delivery of messages
  - Messages are <= 256KB payloads
- SNS topics are the base entity of SNS
  - Permissions and configurations
- A publisher sends messages to a TOPIC
- TOPICS have Subscribers which receive messages
  - Message formats –
    - HTTP(s) Endpoints
    - Email – JSON
    - SQS Queues
    - Mobile Push Notifications
    - SMS Messages
    - Lambda Functions
- SNS uses across AWS notification – CloudWatch & CloudFormation
- ASG can also be configured to scale based on SNS notifications
- Can be accessed from public internet and other vpcs with proper networking configuration
- By default, all subscribers receive all messages published to the topic
  - Configure filter on subscribers to control
- Fanout – single SNS topic with multiple SQS queues as subscribers
  - Create multiple related workloads
- Features
  - Delivery Status – HTTP, Lambda, SQS
  - Delivery Retried – Reliable Delivery
  - HA and Scalable – Regionally Resilient
  - Server Side Encryption – SSE
  - Cross-Account access – via TOPIC Policy (resource policy)

# Step Functions
- Long – running serverless workflow-based applications.
- Some problems with Lambda
  - Lambda is FaaS
  - 15 minute max execution time
  - Can be chained together – gets messy at scale
  - Runtime Environments are stateless
- Step Function State Machines
  - Serverless workflow - START –> STATES -> END
  - States are THINGS that occur
  - Maximum Duration – 1 year
  - Standard Workflow (Default)