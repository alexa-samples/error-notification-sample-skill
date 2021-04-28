
# Error Notifications for Alexa Skills

## Synopsis

This repo contains an Alexa skill that demonstrates sending messages for different types of errors, using AWS Short Notification Service (SNS) and Slack Webhooks.

**Because of this skill's complex infrastructure requirements, it is not well-suited as an Alexa-hosted skill.**

## Introduction

Awareness of defects is a basic building block of operational excellence for Alexa skills. Skill crashes, i.e. failed responses / sessions, are a very disruptive experience for Alexa skill customers, and can have a range of negative consequences:
- Reduced engagement + retention generally
- Reduced trust, so that customers are less likely to purchase from the skill
- Negative reviews, especially by customers who turned from engaged to frustrated

Thus, monitoring skill crashes should be part of every skill developer's toolkit.

This repository contains an Alexa skill that simulates some common causes of skill crashes, and mechanisms to generate alerts about these via email and/or Slack. The error notifications are designed to be maximum helpful for identifying the errors in your logs.

### Challenges in monitoring errors

If you're the only one using a skill, particularly while in development, making sense of your logs is quite easy. Typically you can are the customer who experiences the error, and if you look it up right away, all your session's logs will be neatly ordered in the most current CloudWatch log group.

However, if your skill experiences errors in a production environment, tracing errors will me more challenging:
- You need to identify the log group in which the error was logged
- If you have concurrent Lambda instances, the logs of the failed session might be distributed over multiple log groups
- You want to have the right amount of logging to be able to reproduce the error, e.g. the device's interfaces, the customer's requests, the skill's response, session variables etc.

A particularly severe kind of error is when your Lambda function crashes, e.g. because you try to load a non-existent module. Inthis case, the skill's error handler crashes along with the Lambda function, so that you don't get a full error mail. To mitigate this scenario, this skill provisions a CloudWatch alert that is triggered by failed Lambda invocations. This error message is only available via email (not Slack), and is less verbose, but gives you some degree of visibility into the error.

## Pre-requisites

This repository assumes the following:
- You have ASK CLI v2 installed and set up
- You use AWS, have a programmatic IAM user associated to your ASK CLI, and have permissions for Lambda, CloudWatch, IAM, and SNS
- Optional: You have access to a Slack workspace where you can create channels and apps

## Setup

This project makes heavy use of a CloudFormation template (`infrastructure/cfn-deployer/skill-stack.yaml`) for the ASK CLI deployment. You should customize your CloudFormation template with the following parameters before deploying the skill:
- `NotificationEMail`: The email address to which you want error notification mails delivered by AWS SNS
- `DebugMode`: Should be either `true` or `false`. In debug mode (default setting), the error handler will emit the error message and location within Alexa's speech output. If debug mode is disabled, it will only emit a sound.
- `LogRetention`: The number of days that AWS CloudWatch will retain your logs before deleting them. Default (for dev environment) is 1, but for a production skill you'd want at least 30 ndays.
- `SlackWebhook`: This contains a dummy webhook that is not functional. You can either leave it empty if you don't want to receive error messages in Slack, or follow the instructions in the next section to create your own Slack webhook URL

### Create a Slack webhook

Follow these steps to create an app that can publish error messages into a selected channel in your workspace:
1. Log into a Slack workspace in which you are authorized to create channels and apps.
2. In Slack, create a new channel for error notifications to be posted in, e.g. 'Error Notifications'.
3. Open your [Slack API Apps overview page](https://api.slack.com/apps) and click on 'Create New App'.
4. In the wizard, select a name for your app (e.g. 'Error Notification Bot') and the Workspace you want to work with.
5. Now you should be in the 'Basic Information' screen for your new app. With 'Display Information' below you can give your app an icon. When you're done, click on 'Incoming Webhooks' in the 'Add features and functionality' section.
6. Toggle 'Active Incoming Webhooks' to 'On' in the 'Incoming Webhooks' screen.
7. Now scroll down and click on the button 'Add New Webhook to Workspace'.
8. In the dropdown menu 'Where should `<App Name>` post?', choose the channel you created in step 2 and confirm.
9. Back in the 'Incoming Webhooks' screen, scroll to the bottom to see the webhook URL that your app can use to post in the selected channel. Copy the URL and paste as the value of `SlackWebhook` in your CloudFormation template.

### Deploy

Once you have customized your CloudFormation template, you're ready to deploy:

```
ask deploy
```

This will create a new custom skill with `de-DE`, `en-GB` and `en-US` locales, and a CloudFormation stack with the following resources:
- A Lambda function with environment variables for most of the parameters you specified above
- The IAM role for the Lambda function, with permissions for CloudWatch and SNS
- The event permission of the Alexa skill to invoke the Lambda function
- A CloudWatch log group with your selected log retention
- A CloudWatch alarm that is triggered once the number of errors per 60 seconds is above one
- An SNS topic with the `NotificationEMail` email as a subscriber

Please note that you will get an email from AWS to the `NotificationEMail` address. You need to open this email and confirm that you opt-in to this subscription, otherwise you won't receive emails with error messages.

## Using the skill

The happy path of this skill is a conversation like the following:

- **Customer:** :speech_balloon: Alexa, open Crash Monitor!
- **Alexa:** :loud_sound: Welcome! Which kind of error do you want to trigger: Time-out, invalid response, or runtime exception?
- **Customer:** :speech_balloon: Invalid response.
- **Alexa:** :loud_sound: As you wish! [short break] There was a problem with the requested skill's response.

If you set up your skill completely, you will receive a message on both email and Slack, with the following data:
- Error type: `INVALID_RESPONSE`
- Error message: `Invalid SSML Output Speech [...]. Error: Invalid attribute target for SSML element break`
- Log stream: `2020/09/01/[$LATEST]abcdef0123456789...`
- Log stream URL: `https://console.aws.amazon.com/cloudwatch/home?region=...`
- Session ID: `amazn1.echo-api.session.abcdef0123456789...`
- Session query URL: `https://console.aws.amazon.com/cloudwatch/home?region=...`
- AWS Request ID: `abcdef01-2345-6789-abcdef012345`
- Error timestamp: `2020-09-01T12:00:00.000Z`
- Handler input: `[object Object]` (Basically, the request envelope)

With this data, you can efficiently trace the error in CloudWatch logs. But before you do, take a step back and investigate which log data the skill emits:
- If the request is the first in a session, the full handler input object will be logged
- For subsequent requests in a session, only the request propoerty will be logged
- To enable better tacing of a user's interaction with the skill, the current `sessionId` gets appended to the logged request property
- The `outputSpeech` text or SSML is also logged, so that SSML errors can be identified easier
- Every log item also contains the prepended AWS request ID. This will be useful for showing all log items for one interaction

To quickly access the logs for the Lambda invocation in which the error was raised:
1. Copy the value of **AWS Request ID** from the error message to your clipboard
2. Follow the **Log stream URL** link. It will take you to the corresponding CloudWatch log stream in your AWS console (asking you to log in if you aren't already).
3. Paste the AWS Request ID into the search bar. This will show you the logs for this exact turn.

To trace the entire session until the point where it failed:
1. Follow the **Session query URL** link in the email, which will lead you to CloudWatch insights
2. Give CloudWatch up to a minute to ingest and index all the log data. If you execute the query too soon, you will get 0 results.
3. After about a minute, execute the pre-populated query with 'Run query'
4. Each line in the result should represent one turn in your failed session. You can use the fields `logStream` and `requestId` to 'zoom in' on individual turns in more detail.

## AWS Services and Costs

For full transparency, here is a list of the AWS services that this project uses, and their respective costs (in the `us-east-1` region, at 2020-07-20). If you're using this as for learning and demonstration purposes, you can expect no AWS costs, but if you build error notifications into your skill projects and have a lot of errors, you might incur minor costs for AWS SNS.

| Service              | Free Tier                                       | Costs                                               | Comment                                                                                 |
|----------------------|-------------------------------------------------|-----------------------------------------------------|-----------------------------------------------------------------------------------------|
| Lambda               | 400,000 GB-seconds per month                    | $0.0000166667 per GB-second                         |                                                                                         |
| IAM roles            | n/a                                             | n/a                                                 | IAM roles are free of charge                                                            |
| CloudWatch Log Group | 5GB Data (ingestion, archival + scanning)       | $0.50 per GB (ingestion) + $0.03  per GB (archival) | To reduce archival costs, the CloudFormation template contains a log retention policy   |
| CloudWatch Insights  | 5GB Data (ingestion, archival + scanning)       | $0.005  per GB of data scanned                      | The error notification mail contains a prepared CloudWatch Insight query                |
| CloudWatch Alarms    | 10 Alarm metrics                                | $0.10 per alarm metric                              | This project creates a CloudWatch alarm to monitor your Lambda for errors               |
| SNS Publishes        | First 1 million SNS requests per month          | $0.50  per 1 million Amazon SNS requests thereafter | If you have so many errors, these 50 cents will be the least of your concerns :D        |
| SNS email delivery   | 1000 requests (of max 64 KB per item) per month | $2.00 per 100,000 requests                          | This, along with CloudWatch logs, is where you would most likely see any kind of costs. |
| CloudFormation       | n/a                                             | n/a                                                 | CloudFormation is free of charge, but you get billed for the services it provisions     |                                                                                    |
