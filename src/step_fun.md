# AWS Step Functions

<figure>
  <img src="./images/Arch_AWS-Step-Functions_64@5x.png" alt="AWS Step Functions Icon" width=200>
  <figcaption><center>AWS Step Functions<br><i>Image source: AWS Documentation</i></center></figcaption>
</figure>

**Overview**: AWS Step Functions is a serverless orchestration service that lets you coordinate multiple AWS services into **workflows (state machines)**. On the Security Specialty exam, Step Functions is the primary tool for **incident response playbooks** — automating multi-step remediation processes such as isolating a compromised instance, rotating credentials, collecting forensic data, and notifying the security team.

**Domain weight**: Appears most heavily in Incident Response (~19%). Step Functions is the **orchestrator for security automation** — Lambda handles individual actions, EventBridge routes events, Step Functions coordinates the overall response workflow.

---

## 1. Core Concepts

### 1.1. State Machines

- A **state machine** is a workflow defined in **Amazon States Language (ASL)** — a JSON-based declarative language
- Each state machine has a **StartAt** state and can have one or more **End** states
- State machines are identified by **ARN**: `arn:aws:states:<region>:<account>:stateMachine:<name>`
- State machine executions have a unique **execution ID** and can be referenced for audit trails

### 1.2. Workflow Types

| Feature               | Standard Workflow                  | Express Workflow                      |
| --------------------- | ---------------------------------- | ------------------------------------- |
| **Duration**          | Up to 1 year                       | Up to 5 minutes                       |
| **Execution rate**    | Limited (thousands/second)         | Virtually unlimited (millions/second) |
| **Execution history** | Full history (up to 25,000 events) | CloudWatch Logs only (no history API) |
| **Pricing**           | Per state transition               | Per execution + duration              |
| **Use case**          | Long-running security playbooks    | High-volume event processing          |
| **Start workflow**    | EventBridge, API Gateway, SDK      | EventBridge, API Gateway, SDK         |

**Exam scenario**: A security incident response playbook may take hours to complete (e.g., forensic data collection, manual approval steps) → use **Standard Workflow** (up to 1 year duration, full execution history for audit).

**Exam scenario**: A high-volume event processing pipeline (e.g., processing millions of CloudTrail events per day for anomaly detection) → use **Express Workflow** (high throughput, lower cost).

### 1.3. State Types

| State        | Purpose                                                                    |
| ------------ | -------------------------------------------------------------------------- |
| **Task**     | Perform a unit of work (invoke Lambda, run ECS task, publish to SNS, etc.) |
| **Choice**   | Branch based on input data (conditionals)                                  |
| **Parallel** | Execute multiple branches concurrently                                     |
| **Map**      | Iterate over a list of items, executing steps for each                     |
| **Wait**     | Delay for a specified time or until a timestamp                            |
| **Pass**     | Pass input to output, optionally inject data                               |
| **Fail**     | Stop execution and mark as failed                                          |
| **Succeed**  | Stop execution and mark as succeeded                                       |

**Exam tip**: **Parallel** and **Map** states are heavily used in security playbooks — e.g., collecting forensic data from multiple instances in parallel (Parallel), or processing a list of compromised IAM keys (Map).

## 2. Service Integrations

### 2.1. Integration Patterns

Step Functions integrates with over 200 AWS services using three patterns:

| Pattern                                   | Description                                                     | Use Case                              |
| ----------------------------------------- | --------------------------------------------------------------- | ------------------------------------- |
| **Request Response**                      | Call a service and continue immediately (don't wait for result) | SNS publish, SQS send message         |
| **Run a Job (.sync)**                     | Call a service and wait for the job to complete                 | ECS RunTask, Glue job, Batch job      |
| **Wait for Callback (.waitForTaskToken)** | Call a service and wait for a task token to be returned         | Human approval step, external process |

### 2.2. Key Integrations for Security Automation

| Service                      | Pattern           | Security Use Case                                     |
| ---------------------------- | ----------------- | ----------------------------------------------------- |
| **Lambda**                   | Request Response  | Execute individual security actions (isolate, revoke) |
| **SNS**                      | Request Response  | Send notification to security team                    |
| **SQS**                      | Request Response  | Queue security events for downstream processing       |
| **ECS / Fargate**            | Run a Job (.sync) | Run forensic analysis container                       |
| **Glue**                     | Run a Job (.sync) | Run data analysis on security logs                    |
| **SageMaker**                | Request Response  | Invoke ML model for anomaly detection                 |
| **DynamoDB**                 | Request Response  | Store incident data, update case management           |
| **EventBridge**              | Request Response  | Emit events to trigger further automation             |
| **CodeBuild / CodePipeline** | Run a Job (.sync) | Deploy security tooling or build forensic images      |
| **ECS RunTask**              | Run a Job (.sync) | Run a containerized security scanning tool            |
| **S3 PutObject**             | Request Response  | Store forensic evidence, logs, artifacts              |

### 2.3. Wait for Callback (Human Approval)

- The `.waitForTaskToken` pattern is critical for **manual approval steps** in incident response
- Step Functions pauses execution and sends a **task token** to a target (e.g., SNS topic)
- A human reviews the request and calls `SendTaskSuccess` or `SendTaskFailure` with the task token
- The workflow continues based on the human's decision

**Exam scenario**: An incident response playbook requires a security analyst to approve isolating a production EC2 instance → use **Wait for Callback (.waitForTaskToken)** integration — send a notification via SNS with the task token, wait for the analyst to approve or deny, then proceed accordingly.

## 3. Incident Response Playbooks (Exam Critical)

### 3.1. Common Security Automation Workflows

| Playbook                         | Steps                                                                                                                                                               |
| -------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **EC2 Compromise Response**      | 1. Get instance details → 2. Isolate instance (change SG) → 3. Take EBS snapshot → 4. Notify security team → 5. Wait for analyst approval → 6. Terminate or restore |
| **IAM Key Compromise Response**  | 1. Identify the compromised key → 2. Disable the key → 3. Detach all policies → 4. Notify the user → 5. Wait for confirmation → 6. Rotate or delete                 |
| **S3 Public Bucket Remediation** | 1. Identify public buckets → 2. Apply Block Public Access → 3. Enable encryption → 4. Log the change → 5. Notify the bucket owner                                   |
| **Malware Detection Response**   | 1. Isolate EC2 instance → 2. Take forensic snapshot → 3. Deploy remediation → 4. Scan snapshot with GuardDuty/Macie → 5. Restore if clean                           |
| **Credential Leak Response**     | 1. Identify leaked credentials → 2. Rotate the secret (Secrets Manager) → 3. Revoke existing sessions → 4. Update affected resources → 5. Notify                    |
| **Multi-Account Investigation**  | 1. Assume role into affected account → 2. Gather evidence → 3. Apply containment → 4. Log findings to central account                                               |

### 3.2. Example: EC2 Compromise Response Workflow

```json
{
  "Comment": "EC2 Compromise Incident Response Playbook",
  "StartAt": "GetInstanceDetails",
  "States": {
    "GetInstanceDetails": {
      "Type": "Task",
      "Resource": "arn:aws:states:::lambda:invoke",
      "Parameters": {
        "FunctionName": "${GetInstanceDetailsFunction}",
        "Payload": {
          "instanceId.$": "$.detail.resources[0]"
        }
      },
      "Next": "IsolateInstance"
    },
    "IsolateInstance": {
      "Type": "Task",
      "Resource": "arn:aws:states:::lambda:invoke",
      "Parameters": {
        "FunctionName": "${IsolateInstanceFunction}",
        "Payload": {
          "instanceId.$": "$.instanceId",
          "securityGroupId.$": "$.isolationSgId"
        }
      },
      "Next": "TakeForensicSnapshot"
    },
    "TakeForensicSnapshot": {
      "Type": "Task",
      "Resource": "arn:aws:states:::lambda:invoke",
      "Parameters": {
        "FunctionName": "${TakeSnapshotFunction}",
        "Payload": {
          "instanceId.$": "$.instanceId"
        }
      },
      "Next": "NotifySecurityTeam"
    },
    "NotifySecurityTeam": {
      "Type": "Task",
      "Resource": "arn:aws:states:::sns:publish",
      "Parameters": {
        "TopicArn": "${SecurityTopicArn}",
        "Message": {
          "instanceId.$": "$.instanceId",
          "status": "Isolated, forensic snapshot taken. Awaiting approval.",
          "taskToken.$": "$$.Task.Token"
        }
      },
      "Next": "WaitForApproval"
    },
    "WaitForApproval": {
      "Type": "Task",
      "Resource": "arn:aws:states:::lambda:invoke.waitForTaskToken",
      "Parameters": {
        "FunctionName": "${ApprovalFunction}",
        "Payload": {
          "taskToken.$": "$$.Task.Token",
          "instanceId.$": "$.instanceId"
        }
      },
      "Next": "HandleApproval"
    },
    "HandleApproval": {
      "Type": "Choice",
      "Choices": [
        {
          "Variable": "$.approval",
          "StringEquals": "APPROVED",
          "Next": "TerminateInstance"
        },
        {
          "Variable": "$.approval",
          "StringEquals": "DENIED",
          "Next": "RestoreInstance"
        }
      ]
    },
    "TerminateInstance": {
      "Type": "Task",
      "Resource": "arn:aws:states:::lambda:invoke",
      "Parameters": {
        "FunctionName": "${TerminateInstanceFunction}",
        "Payload": {
          "instanceId.$": "$.instanceId"
        }
      },
      "End": true
    },
    "RestoreInstance": {
      "Type": "Task",
      "Resource": "arn:aws:states:::lambda:invoke",
      "Parameters": {
        "FunctionName": "${RestoreInstanceFunction}",
        "Payload": {
          "instanceId.$": "$.instanceId"
        }
      },
      "End": true
    }
  }
}
```

**Exam scenario**: An organization needs an automated incident response workflow that: isolates a compromised EC2 instance, takes an EBS snapshot for forensics, notifies the security team, and waits for a human to approve termination or restoration → use **AWS Step Functions** with `waitForTaskToken` for the approval step.

## 4. IAM Roles for Step Functions

### 4.1. Execution Role

- Step Functions needs an **execution role** (IAM role) that grants it permission to invoke the services in the workflow
- The role must allow `states:StartExecution` for Step Functions to run the state machine
- The role must allow the specific actions for each integrated service in the workflow

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "lambda:InvokeFunction",
        "sns:Publish",
        "ec2:DescribeInstances",
        "ec2:CreateSnapshot",
        "ec2:ModifyInstanceAttribute"
      ],
      "Resource": "*"
    }
  ]
}
```

**Exam scenario**: A Step Functions state machine invokes Lambda functions and publishes to SNS → the **execution role** must have `lambda:InvokeFunction` and `sns:Publish` permissions.

### 4.2. Task-Specific IAM Permissions

| Task Type        | Required Permission           | Resource                             |
| ---------------- | ----------------------------- | ------------------------------------ |
| Lambda invoke    | `lambda:InvokeFunction`       | Lambda function ARN                  |
| SNS publish      | `sns:Publish`                 | SNS topic ARN                        |
| SQS send message | `sqs:SendMessage`             | SQS queue ARN                        |
| ECS RunTask      | `ecs:RunTask`, `iam:PassRole` | ECS task definition + execution role |
| DynamoDB PutItem | `dynamodb:PutItem`            | DynamoDB table ARN                   |
| Glue StartJobRun | `glue:StartJobRun`            | Glue job ARN                         |
| Batch SubmitJob  | `batch:SubmitJob`             | Batch job queue/definition           |
| SageMaker Invoke | `sagemaker:InvokeEndpoint`    | SageMaker endpoint ARN               |

## 5. Error Handling

### 5.1. Retry

- **Retry** defines how to retry a failed state
- Configurable: `MaxAttempts`, `IntervalSeconds`, `BackoffRate`, `JitterEnabled`
- Retries are defined as a JSON array in the state definition

```json
"SomeTask": {
  "Type": "Task",
  "Resource": "arn:aws:states:::lambda:invoke",
  "Retry": [
    {
      "ErrorEquals": ["Lambda.ServiceException", "Lambda.AWSLambdaException", "Lambda.SdkClientException", "States.TaskFailed"],
      "IntervalSeconds": 2,
      "MaxAttempts": 3,
      "BackoffRate": 2.0
    }
  ],
  "Next": "NextState"
}
```

### 5.2. Catch

- **Catch** defines what to do if the state fails after retries are exhausted
- Routes to a fallback state (e.g., notify, log, or perform recovery)

```json
"SomeTask": {
  "Type": "Task",
  "Resource": "arn:aws:states:::lambda:invoke",
  "Catch": [
    {
      "ErrorEquals": ["States.ALL"],
      "Next": "NotifyFailure"
    }
  ],
  "Next": "SuccessState"
}
```

### 5.3. Timeouts

| Timeout Type       | Description                                             | Default                              |
| ------------------ | ------------------------------------------------------- | ------------------------------------ |
| `TimeoutSeconds`   | Maximum time the state can run                          | 60 seconds (Task), 99999999 (others) |
| `HeartbeatSeconds` | Maximum time between heartbeats for `.waitForTaskToken` | None                                 |

**Exam scenario**: A human approval step in a security playbook should time out after 1 hour if the analyst does not respond → set `TimeoutSeconds` to 3600 on the `.waitForTaskToken` task state.

## 6. Monitoring and Auditing

### 6.1. CloudWatch Metrics

| Metric                | Description                                      |
| --------------------- | ------------------------------------------------ |
| `ExecutionsStarted`   | Number of executions started                     |
| `ExecutionsSucceeded` | Number of executions that completed successfully |
| `ExecutionsFailed`    | Number of executions that failed                 |
| `ExecutionsTimedOut`  | Number of executions that timed out              |
| `ExecutionTime`       | The duration of execution (minutes)              |
| `ActivityScheduled`   | Number of activities scheduled                   |
| `ActivityStarted`     | Number of activities started                     |

### 6.2. CloudWatch Logs

- Step Functions can send **execution event logs** to **CloudWatch Logs**
- Logs capture: state transitions, input/output data, errors, and timing
- **Only Express Workflows** support CloudWatch Logs integration natively (no execution history API)
- Standard Workflows store execution history internally (accessible via `DescribeExecution` and `GetExecutionHistory` APIs)

### 6.3. CloudTrail

- All Step Functions API calls are logged in CloudTrail:
  - `CreateStateMachine`
  - `UpdateStateMachine`
  - `StartExecution`
  - `StopExecution`
  - `SendTaskSuccess` / `SendTaskFailure`
- Monitor for unauthorized creation or modification of state machines

### 6.4. X-Ray

- Step Functions integrates with **AWS X-Ray** for tracing
- Tracks end-to-end execution flow across all integrated services
- Useful for debugging security automation failures

## 7. Input and Output Processing

### 7.1. InputPath, OutputPath, ResultPath

| Path         | Purpose                                                   |
| ------------ | --------------------------------------------------------- |
| `InputPath`  | Filter the input to the state (select specific fields)    |
| `OutputPath` | Filter the output from the state (select specific fields) |
| `ResultPath` | Specify where to place the result in the state's input    |
| `Parameters` | Create a new JSON payload for the task                    |

### 7.2. Security Implications

- Use `InputPath` and `OutputPath` to **avoid passing sensitive data** between states
- A state might receive sensitive information (e.g., credentials, API keys) — use `ResultPath` to control what gets passed forward
- The execution history can be viewed by anyone with `states:DescribeExecution` permission — avoid logging sensitive data

**Exam scenario**: A security workflow receives a GuardDuty finding with sensitive instance metadata. Pass only the necessary fields (instance ID, finding type) to subsequent states → use **InputPath** and **ResultPath** to filter the payload between states.

## 8. Security Best Practices

| Practice                                                   | Description                                                         |
| ---------------------------------------------------------- | ------------------------------------------------------------------- |
| **Use least-privilege execution role**                     | Grant only the specific actions needed by the workflow              |
| **Filter sensitive data**                                  | Use InputPath, OutputPath, and ResultPath to limit data exposure    |
| **Enable CloudTrail logging**                              | Monitor state machine creation, execution, and modification         |
| **Use Standard Workflows for long-running playbooks**      | Incidents may take hours to resolve; Standard supports up to 1 year |
| **Use Express Workflows for high-volume event processing** | For processing large volumes of security events                     |
| **Set appropriate timeouts**                               | Avoid indefinite waits on human approval steps                      |
| **Use Retry and Catch for resilience**                     | Ensure security automation completes despite transient failures     |
| **Use tags on state machines**                             | Organize and control access by environment, team, or purpose        |
| **Use parameters for service integration**                 | Avoid hardcoding ARNs — use dynamic references                      |
| **Use .waitForTaskToken for human approval**               | Ensure critical actions require analyst confirmation                |
| **Validate input/output in workflows**                     | Use Choice states to validate before executing destructive actions  |
| **Use CloudFormation/SAM/CDK to deploy state machines**    | Infrastructure as code for security automation                      |

## 9. Step Functions vs Lambda (Orchestration)

### 9.1. When to Use Which

| Feature                | Step Functions                            | Lambda (chained)                               |
| ---------------------- | ----------------------------------------- | ---------------------------------------------- |
| **Orchestration**      | ✅ Designed for multi-step workflows      | ❌ Lambda-to-Lambda invocation (complex, deep) |
| **Human approval**     | ✅ `waitForTaskToken` pattern             | ❌ Must implement yourself                     |
| **Error handling**     | ✅ Built-in Retry, Catch, Timeout         | ❌ Basic (DLQ, retry)                          |
| **State tracking**     | ✅ Full execution history and audit trail | ❌ Limited to CloudWatch Logs                  |
| **Parallel execution** | ✅ Native Parallel and Map states         | ❌ Must implement with SNS/SQS fan-out         |
| **Wait/delay**         | ✅ Built-in Wait state                    | ❌ Must use CloudWatch Events or custom code   |
| **Conditional logic**  | ✅ Choice state                           | ❌ Must use if/else in code                    |
| **Visibility**         | ✅ Visual workflow in console             | ❌ Code only                                   |
| **Max duration**       | Up to 1 year (Standard)                   | 15 minutes                                     |

**Exam scenario**: An incident response process requires 8 separate steps, including human approval and conditional branching (isolate vs restore) → use **Step Functions** for orchestration, not chained Lambda calls.

## 10. Common Exam Scenarios

1. **Incident response playbook**: Orchestrate a multi-step security response — isolate instance, take snapshot, notify team, wait for approval, terminate or restore → **Step Functions** with `waitForTaskToken`.

2. **Human approval in automation**: Before terminating a production instance, a security analyst must approve → **`.waitForTaskToken`** pattern with SNS notification.

3. **Parallel forensic collection**: Collect evidence from multiple compromised instances simultaneously → **Parallel state**.

4. **Iterate over findings**: Process a list of GuardDuty findings in a batch → **Map state** to iterate over each finding.

5. **Long-running security scan**: A containerized security scanner takes 30 minutes → **Standard Workflow** with **ECS RunTask (.sync)** to wait for the scan to complete.

6. **Security automation with error recovery**: A step fails (e.g., Lambda timeout), but the workflow should continue with a fallback action → use **Catch** to route to a fallback state.

7. **Cross-account security orchestration**: A state machine in the central security account needs to perform actions in member accounts → the execution role uses `sts:AssumeRole` to assume a role in each member account for each task.

8. **Input filtering for sensitive data**: A workflow receives a GuardDuty finding that includes sensitive metadata — pass only the instance ID to subsequent steps → use **InputPath**.

9. **CloudTrail monitoring of Step Functions**: An organization needs to detect unauthorized changes to security playbooks → monitor CloudTrail for `CreateStateMachine`, `UpdateStateMachine`, `DeleteStateMachine`.

10. **Step Functions + EventBridge + Lambda**: The complete security automation stack — EventBridge routes security events to Step Functions, Step Functions orchestrates the response using Lambda actions.

## 11. Exam Tips

1. **Step Functions = orchestration**: It coordinates multi-step workflows. Lambda does individual actions. EventBridge routes events.

2. **Standard vs Express**: Standard = long-running, full history (for audits). Express = high-throughput, CloudWatch Logs only.

3. **`.waitForTaskToken` is the key pattern**: This is how human approval is implemented in security playbooks. Expect it on the exam.

4. **Parallel and Map states**: Use Parallel to run actions concurrently. Use Map to iterate over a list (e.g., a list of compromised instances).

5. **Error handling with Retry and Catch**: Retry = retry on transient failures. Catch = fallback when retries are exhausted.

6. **Execution role permissions**: The execution role must have permissions for every service the state machine calls.

7. **Input and output filtering**: Use InputPath, OutputPath, ResultPath to control data flow and protect sensitive information.

8. **TimeoutSeconds**: Critical for human approval steps — always set a timeout to avoid indefinite waits.

9. **Execution history**: Standard Workflows store full execution history for audit — useful for incident post-mortems.

10. **Activity tasks**: The exam may test Activity tasks (workers running on EC2 or on-premises that poll for tasks) — these are less common but relevant for hybrid environments.

11. **Step Functions is not for real-time**: There is some latency (hundreds of ms) for state transitions — not suitable for sub-millisecond responses.

12. **State machine ARN format**: `arn:aws:states:<region>:<account>:stateMachine:<name>`. Execution ARN: `arn:aws:states:<region>:<account>:execution:<name>:<execution-id>`.

13. **Maximum execution history**: 25,000 events (state transitions). If exceeded, the execution is truncated (newer events overwrite older ones).

14. **Maximum execution size**: 256 KB for input, output, and execution history combined.

15. **Task token**: Used in `.waitForTaskToken` pattern. The token is a long-lived identifier that allows the external process to resume the workflow.

16. **Express Workflow pricing**: Pay per execution + duration. Good for high-volume security event processing (millions of events).

17. **Standard Workflow pricing**: Pay per state transition. Good for long-running, infrequent security playbooks.

18. **AWS Managed Policies**: `AWSStepFunctionsFullAccess`, `AWSStepFunctionsReadOnlyAccess`, `AWSStepFunctionsConsoleAccess`.

19. **Cross-account workflows**: Use `sts:AssumeRole` within task states to perform actions in other accounts.

20. **Step Functions is regional**: State machines are regional resources. For cross-region workflows, use separate state machines or trigger executions in other regions.
