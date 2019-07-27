# aws-scheduler

This project provides a solution to schedule large amounts of point in time events with a great time precision. See the Performance section for more details.

The two interfaces are two SNS topics. One for input and one for output. You send a scheduling payload to the input topic and at the specified datetime you will receive your data at the output topic. See the Usage section on how to attach your functions.

![Service Overview](https://github.com/bahrmichael/aws-scheduler/raw/master/pictures/overview.png)

You can deploy this yourself or use our SAAS offer.

## SAAS Offer

While we're ironing things out you can use our service free of charge. Just publish an event to `arn:aws:sns:us-east-1:256608350746:scheduler-input-prod`.  This topic is public so anyone can publish to it. 

If you become a heavy user with more than 100.000 events per month we might want to get in touch with you, so make sure to fill out the `user` field with some way to contact you (email, twitter handle, ...).

## Usage

### Setup
First of all you need an output topic that we can publish events to once the scheduled datetime arrives. To do this run `python setup/init_output_topic.py <stage>`. This will create a topic called `scheduler-output-<stage>` and grant our account the right to publish messages. You can see the added policy below.

```json
{
  "Sid": "scheduler-output-<stage>-publish-access",
  "Effect": "Allow",
  "Principal": {
    "AWS": "arn:aws:iam::256608350746:root"
  },
  "Action": "SNS:Publish",
  "Resource": "arn:aws:sns:us-east-1:256608350746:scheduler-output-<stage>"
}
``` 

If you don't want to grant our account the right to publish, you can pass your own accountId as a third argument.

```
Creating topic scheduler-output-<stage>
Created topic scheduler-output-<stage> with arn arn:aws:sns:us-east-1:<your-account-id>:scheduler-output-<stage>
Granting publish rights to scheduler-output-<stage> for accountId 256608350746
Done
```

Write down the ARN of your output topic as you will need it for the input events.

### Input
To schedule a trigger you have to publish an event which follows the structure below to the ARN of the input topic. You can find the ARN of our service in the SAAS Offer section.

```json
{
  "date": "utc timestamp following ISO 8601",
  "target": "arn of your output topic",
  "user": "some way we can get in touch with you",
  "payload": "any string payload"
}
```

SNS messages must be strings. First string encode the json structure and then publish it to the input topic.

```python
# Python example
import json
import boto3

client = boto3.client('sns')

event = {
          "date": "2019-07-27T12:20:24.919071",
          "target": "arn:aws:sns:us-east-1:256608350746:scheduler-output-prod",
          "user": "Twitter @michabahr",
          "payload": "46607451-3e67-49bc-972b-425c150c5456"
        }

input_topic = 'arn:aws:sns:us-east-1:256608350746:scheduler-input-prod'
client.publish(TopicArn=input_topic, Message=json.dumps(event))
```

All fields are mandatory. Please make sure that the `payload` can be utf-8 encoded. If you submit an event that does not follow the spec, it will be dropped. Future versions will improve on this.

So far there is no batch publishing available for SNS. Make sure the event stays within the 256KB limit of SNS. We recommend that you only submit IDs and don't transfer any real data to the service.

### Output
Once the datetime specified in the payload is reached, the service will publish the content of the event field to your output topic. You can attach an AWS Lambda function to your topic to process the events. 

## Deploy it yourself
This section explains how you can deploy the service yourself. Once set up use it like shown above.

The following picture shows you the structure of the service.

![Detailed Overview](https://github.com/bahrmichael/aws-scheduler/raw/master/pictures/detailed.png)

### Prerequisites
You must have the following tools installed:
- serverless framework 1.48.3 or later
- node
- npm
- python3
- pip

Run `setup/init_table.py <stage>` to setup the database. Replace `<stage>` with the stage of your application.

Run `setup/init_input_topic.py <stage> [public]`. Replace `<stage>` with the stage of your application. You may append the parameter `public` to grant public publish rights.

Run `setup/init_queue.py <stage>`. Replace `<stage>` with the stage of your application.

### Deploy
1. Navigate into the project folder
2. With a tooling of your choice create and activate a venv
3. `pip install -r requirements.txt`
4. `npm i serverless-python-requirements`
5. `sls deploy`

Wait for the deployment to finish. Test the service by first attaching a function to the output topic and then send a few events to the input topic.

You can disable the `user` check on the input event for your own deployment with the `ENFORCE_USER` environment variable.
 
## Performance
We ran tests by sending events that included the scheduled timestamp in the payload. Once received we compared those timestamps with the current time.

Our results showed that most of the events arrive within one second of the specified datetime and the rest within the next few seconds.

The charts show the amount of events received on the y axis and the distribution by delay on the x axis.

![Regular Scaled 100000 events within 10 minutes](https://github.com/bahrmichael/aws-scheduler/raw/master/pictures/regular-scaled-100k-10m.png)
![Log Scaled 100000 events within 10 minutes](https://github.com/bahrmichael/aws-scheduler/raw/master/pictures/log-scaled-100k-10m.png)

## Scalability

Based on our tests from the Performance section, we are confident that this stack can handle 100.000.000 events per month and might scale up to 500.000.000 events per month.

These numbers are not confirmed though, as that volume incurs significant AWS costs.

## Limitations
Events may arrive more than once at the output topic.

## Contributions
Contributions are welcome, both issues and code. Get in touch at twitter [@michabahr](https://twitter.com/michabahr) or create an issue.

## TODOs
- attach to own project
- example functions for quick start: put into dedicated folder
- limitation of message size (10kb), also explain why
- use a proper logger
- secure the PoC with test
- include a failure queue and adjust the docs
- add a safe guard that pulls messages from dead letter queues back into the circuit
- handling for messages that can't be utf-8 encoded
