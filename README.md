# aws-serverless-task-manager
AWS serverless Task Manager API using Python Lambda, API Gateway and DynamoDB with CRUD operations and CloudWatch logging.


# AWS Serverless Task Manager

## Project Overview
A serverless REST API for managing tasks using AWS Lambda,
Amazon API Gateway and Amazon DynamoDB.

## Architecture
Client / PowerShell
        |
Amazon API Gateway (HTTP API)
        |
AWS Lambda (Python)
        |
Amazon DynamoDB

CloudWatch Logs captures Lambda execution logs.

## AWS Services
- Amazon API Gateway
- AWS Lambda
- Amazon DynamoDB
- AWS IAM
- Amazon CloudWatch

## Features
- Create a task
- List tasks
- Retrieve a task by ID
- Update a task
- Delete a task
- Validate request data
- Store tasks in DynamoDB
- Log execution details in CloudWatch

## API Endpoints
| Method | Route | Description |
|--------|-------|-------------|
| POST | /tasks | Create a task |
| GET | /tasks | List tasks |
| GET | /tasks/{id} | Get one task |
| PUT | /tasks/{id} | Update a task |
| DELETE | /tasks/{id} | Delete a task |

## Deployment
1. Create the DynamoDB Tasks table with taskId as a String partition key.
2. Create a Python Lambda function.
3. Configure the TABLE_NAME environment variable.
4. Grant the Lambda execution role the required table permissions.
5. Deploy the Lambda code.
6. Create an API Gateway HTTP API.
7. Configure the five routes and attach the Lambda integration.
8. Deploy the API and test each endpoint.

## Testing
Test create, list, get, update and delete operations.
Also test invalid JSON, empty titles and missing task IDs.

## Security
- Follow least-privilege IAM.
- Do not commit credentials.
- Monitor Lambda logs.
- Add authentication and appropriate throttling
  before exposing a production application.

## Screenshots
See the screenshots/ directory for deployment and test evidence.

## Cleanup
Delete the test API, Lambda function, DynamoDB table
and unused IAM resources when the lab is complete.

## Disclaimer
This is an educational project and is not a production-ready
authenticated task management system.

lamda function.py
import json
import os
import uuid
from datetime import datetime, timezone

import boto3
from botocore.exceptions import ClientError

dynamodb = boto3.resource("dynamodb")
table = dynamodb.Table(os.environ["TABLE_NAME"])


def response(status_code, body):
    return {
        "statusCode": status_code,
        "headers": {
            "Content-Type": "application/json"
        },
        "body": json.dumps(body)
    }


def lambda_handler(event, context):
    method = event.get("requestContext", {}).get(
        "http", {}
    ).get("method", "")

    path_params = event.get("pathParameters") or {}
    task_id = path_params.get("id")
    raw_body = event.get("body")

    try:
        if method == "POST":
            data = json.loads(raw_body or "{}")
            title = data.get("title")

            if not isinstance(title, str) or not title.strip():
                return response(400, {
                    "error": "title must be a non-empty string"
                })

            status = data.get("status", "pending")
            if status not in ("pending", "completed"):
                return response(400, {
                    "error": "status must be pending or completed"
                })

            now = datetime.now(timezone.utc).isoformat()
            item = {
                "taskId": str(uuid.uuid4()),
                "title": title.strip(),
                "status": status,
                "createdAt": now,
                "updatedAt": now
            }

            table.put_item(Item=item)
            return response(201, item)

        if method == "GET" and not task_id:
            result = table.scan(Limit=100)
            return response(200, {
                "tasks": result.get("Items", []),
                "count": len(result.get("Items", [])),
                "note": "This lab lists up to 100 items per request."
            })

        if method == "GET" and task_id:
            result = table.get_item(
                Key={"taskId": task_id}
            )
            item = result.get("Item")

            if not item:
                return response(404, {"error": "Task not found"})

            return response(200, item)

        if method == "PUT" and task_id:
            data = json.loads(raw_body or "{}")
            title = data.get("title")
            status = data.get("status")

            if not isinstance(title, str) or not title.strip():
                return response(400, {
                    "error": "title must be a non-empty string"
                })

            if status not in ("pending", "completed"):
                return response(400, {
                    "error": "status must be pending or completed"
                })

            now = datetime.now(timezone.utc).isoformat()

            result = table.update_item(
                Key={"taskId": task_id},
                UpdateExpression=(
                    "SET #title = :title, "
                    "#status = :status, updatedAt = :updatedAt"
                ),
                ConditionExpression="attribute_exists(taskId)",
                ExpressionAttributeNames={
                    "#title": "title",
                    "#status": "status"
                },
                ExpressionAttributeValues={
                    ":title": title.strip(),
                    ":status": status,
                    ":updatedAt": now
                },
                ReturnValues="ALL_NEW"
            )

            return response(200, result["Attributes"])

        if method == "DELETE" and task_id:
            table.delete_item(
                Key={"taskId": task_id},
                ConditionExpression="attribute_exists(taskId)"
            )
            return response(200, {
                "message": "Task deleted",
                "taskId": task_id
            })

        return response(400, {
            "error": "Unsupported method or route"
        })

    except json.JSONDecodeError:
        return response(400, {"error": "Request body must be valid JSON"})

    except ClientError as error:
        code = error.response.get("Error", {}).get("Code", "")

        if code == "ConditionalCheckFailedException":
            return response(404, {"error": "Task not found"})

        print("AWS service error:", code)
        return response(500, {"error": "Internal server error"})

    except Exception as error:
        print("Unexpected error:", str(error))
        return response(500, {"error": "Internal server error"})
