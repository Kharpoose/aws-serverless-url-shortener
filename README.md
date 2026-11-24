# AWS Serverless URL Shortener

A simple serverless URL shortener built with **AWS Lambda**, **API Gateway**, and **Amazon DynamoDB**.

This project demonstrates how to build a fully serverless REST API using AWS services. Users can submit a long URL, receive a generated short ID, and access the original URL through an HTTP redirect.

## Architecture

![Architecture diagram](docs/architecture-diagram.png)

A step-by-step walkthrough of both request flows, with AWS console screenshots, is in [`Serverless_Architecture.xlsx`](Serverless_Architecture.xlsx).

<details>
<summary>Text version</summary>

```text
                    Client
                       │
        ┌──────────────┴──────────────┐
        │                             │
        ▼                             ▼
 POST /shorten                 GET /{shortId}
        │                             │
        └──────────────┬──────────────┘
                       ▼
                Amazon API Gateway
                       │
        ┌──────────────┴──────────────┐
        │                             │
        ▼                             ▼
create-short-url              redirect-url
     Lambda                       Lambda
        │                             │
        └──────────────┬──────────────┘
                       ▼
                Amazon DynamoDB
```

</details>

## AWS Services

- AWS Lambda
- Amazon API Gateway (HTTP API)
- Amazon DynamoDB
- AWS IAM
- Amazon CloudWatch

## Features

- Create shortened URLs
- Redirect to the original URL using HTTP 302
- Fully serverless architecture
- Store URL mappings in DynamoDB
- Environment Variables for configuration
- Error handling
- CloudWatch logging

## Project Structure

```text
aws-serverless-url-shortener/
├── lambda/
│   ├── create/
│   │   └── index.py               # create-short-url Lambda (POST /shorten)
│   └── redirect/
│       └── index.py               # redirect-url Lambda (GET /{shortId})
├── docs/
│   ├── apigateway.md              # API type, routes, request/response examples
│   ├── dynamodb.md                # table configuration and item example
│   └── architecture-diagram.png
├── Serverless_Architecture.xlsx   # step-by-step walkthrough with console screenshots
└── README.md
```

> In the Lambda console each function's code lives in `lambda_function.py` (handler `lambda_function.lambda_handler`). In this repo the same code is saved as `index.py`, so if you upload the file as-is, set the handler to `index.lambda_handler`.

## DynamoDB

### Table Configuration

| Setting | Value |
|---|---|
| Table name | `url-shortener` |
| Partition key | `shortId` (String) |
| Billing mode | On-demand |

### Item Example

```json
{
  "shortId": "Ab12Cd",
  "originalUrl": "https://www.google.com",
  "createdAt": "2026-07-07T10:20:30.123456+00:00"
}
```

`createdAt` is written with `datetime.now(timezone.utc).isoformat()`, so it is an ISO 8601 UTC timestamp with microseconds and a `+00:00` offset.

## API Gateway

### API Type

HTTP API

### Routes

| Method | Path | Lambda |
|---------|------|--------|
| POST | /shorten | create-short-url |
| GET | /{shortId} | redirect-url |

## API Examples

### Create Short URL

```bash
curl -X POST https://<api-id>.execute-api.ap-southeast-2.amazonaws.com/shorten \
  -H "Content-Type: application/json" \
  -d '{"url": "https://www.google.com"}'
```

Response (`200 OK`):

```json
{
  "shortId": "Ab12Cd",
  "originalUrl": "https://www.google.com"
}
```

### Redirect

```bash
curl -i https://<api-id>.execute-api.ap-southeast-2.amazonaws.com/Ab12Cd
```

If the short ID exists, the API returns an **HTTP 302 Redirect** with a `Location` header pointing to the original URL, and the browser opens it automatically.

### Error Responses

Errors are returned as JSON with a `message` field:

| Endpoint | Status | `message` | When |
|---|---|---|---|
| POST /shorten | 400 | `Invalid JSON` | The request body is not valid JSON |
| POST /shorten | 400 | `url is required` | `url` is missing or empty |
| POST /shorten | 500 | `Failed to save URL` | The DynamoDB write failed |
| GET /{shortId} | 400 | `shortId is required` | The path parameter is missing |
| GET /{shortId} | 404 | `Short URL not found` | No item exists for this ID |
| GET /{shortId} | 500 | `Internal Server Error` | The DynamoDB read failed |

## Lambda Functions

### create-short-url

Responsibilities:

- Parse the incoming request body
- Generate a random short ID
- Store the URL in DynamoDB
- Return the generated short ID

### redirect-url

Responsibilities:

- Receive the short ID from the URL path
- Retrieve the original URL from DynamoDB
- Return an HTTP 302 redirect response
- Return HTTP 404 if the short ID does not exist

## Environment Variables

Both Lambda functions require the following environment variable:

| Name | Value |
|------|-------|
| TABLE_NAME | url-shortener |

## Setup (manual, AWS console)

The project was built by hand in the AWS console in `ap-southeast-2` (Sydney); there is no infrastructure as code yet. To recreate it:

1. **DynamoDB** — Create the `url-shortener` table with partition key `shortId` (String) and on-demand billing.
2. **IAM** — Give each function an execution role with the `AWSLambdaBasicExecutionRole` managed policy (CloudWatch logging) plus only the table access it needs:

   `create-short-url` (write only):
   ```json
   {
     "Version": "2012-10-17",
     "Statement": [
       {
         "Effect": "Allow",
         "Action": "dynamodb:PutItem",
         "Resource": "arn:aws:dynamodb:<region>:<account-id>:table/url-shortener"
       }
     ]
   }
   ```

   `redirect-url` (read only): the same policy with `dynamodb:GetItem` instead of `dynamodb:PutItem`.
3. **Lambda** — Create the `create-short-url` and `redirect-url` Python functions with those roles, paste in the code from `lambda/create/index.py` and `lambda/redirect/index.py`, and set `TABLE_NAME=url-shortener` on both.
4. **API Gateway** — Create an HTTP API with a `POST /shorten` route integrated with `create-short-url` and a `GET /{shortId}` route integrated with `redirect-url`.
5. **Test** — Use the `curl` commands in [API Examples](#api-examples).

## Cost and Cleanup

At demo traffic the cost is effectively zero: Lambda and DynamoDB stay within their free allowances, and HTTP API requests are billed per million.

To remove everything, delete the HTTP API, both Lambda functions and their log groups (`/aws/lambda/create-short-url`, `/aws/lambda/redirect-url`), the `url-shortener` table, and the IAM roles.

## Known Limitations

- **Short ID collisions** — IDs are 6 random characters and are written without a condition, so a (rare) collision would silently overwrite an existing link. A conditional write (`attribute_not_exists(shortId)`) with a retry would prevent this.
- **No URL validation** — any non-empty string is stored and later returned in the `Location` header. Accepting only `http`/`https` URLs would be safer.

## Future Improvements

- Return the complete shortened URL instead of only the short ID.
- Prevent duplicate shortened URLs for the same original URL.
- Add click statistics.
- Add URL expiration using DynamoDB TTL.
- Deploy the infrastructure using AWS SAM or Terraform.
- Add automated tests.

## Learning Objectives

This project demonstrates:

- Serverless application development
- REST API design
- AWS Lambda
- Amazon API Gateway
- Amazon DynamoDB
- IAM permissions
- CloudWatch logging
- HTTP redirects
- Environment Variables
- Python with boto3
