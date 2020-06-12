# Serverless AWS Alerts Plugin
  [![NPM version][npm-image]][npm-url]
  [![Build Status][travis-image]][travis-url]
  [![Dependency Status][daviddm-image]][daviddm-url]
  [![Coverage percentage][coveralls-image]][coveralls-url]

A Serverless plugin to easily add CloudWatch alarms to functions

## Installation
`npm i serverless-plugin-aws-alerts`

NOTE: to install this fork (with support for `skipOKMetric` and `metricValue`), use:

`npm install --save-dev git+https://git@github.com/chinatti/serverless-plugin-aws-alerts.git`

## Usage

```yaml
service: your-service
provider:
  name: aws
  runtime: nodejs4.3

custom:
  alerts:
    stages: # Optionally - select which stages to deploy alarms to
      - production
      - staging

    dashboards: true

    nameTemplate: $[functionName]-$[metricName]-Alarm # Optionally - naming template for alarms, can be overwritten in definitions
    prefixTemplate: $[stackName] # Optionally - override the alarm name prefix

    topics:
      ok: ${self:service}-${opt:stage}-alerts-ok
      alarm: ${self:service}-${opt:stage}-alerts-alarm
      insufficientData: ${self:service}-${opt:stage}-alerts-insufficientData
    definitions:  # these defaults are merged with your definitions
      functionErrors:
        period: 300 # override period
      customAlarm:
        description: 'My custom alarm'
        namespace: 'AWS/Lambda'
        nameTemplate: $[functionName]-Duration-IMPORTANT-Alarm # Optionally - naming template for the alarms, overwrites globally defined one
        prefixTemplate: $[stackName] # Optionally - override the alarm name prefix, overwrites globally defined one
        metric: duration
        threshold: 200
        statistic: Average
        period: 300
        evaluationPeriods: 1
        datapointsToAlarm: 1
        comparisonOperator: GreaterThanOrEqualToThreshold
    alarms:
      - functionThrottles
      - functionErrors
      - functionInvocations
      - functionDuration

plugins:
  - serverless-plugin-aws-alerts

functions:
  foo:
    handler: foo.handler
    alarms: # merged with function alarms
      - customAlarm
      - name: fooAlarm # creates new alarm or overwrites some properties of the alarm (with the same name) from definitions
        namespace: 'AWS/Lambda'
        metric: errors # define custom metrics here
        threshold: 1
        statistic: Minimum
        period: 60
        evaluationPeriods: 1
        datapointsToAlarm: 1
        comparisonOperator: GreaterThanOrEqualToThreshold
      - name: functionInvocations
        enabled: false # Disable this globally defined alarm for this lambda
```

## Multiple topic definitions

You can define several topics for alarms. For example you want to have topics for critical alarms
reaching your pagerduty, and different topics for noncritical alarms, which just send you emails.

In each alarm definition you have to specify which topics you want to use. In following example
you get an email for each function error, pagerduty gets alarm only if there are more than 20
errors in 60s

```yaml
custom:
  alerts:

    topics:
      critical:
        ok:
          topic: ${self:service}-${opt:stage}-critical-alerts-ok
          notifications:
          - protocol: https
            endpoint: https://events.pagerduty.com/integration/.../enqueue
        alarm:
          topic: ${self:service}-${opt:stage}-critical-alerts-alarm
          notifications:
          - protocol: https
            endpoint: https://events.pagerduty.com/integration/.../enqueue

      nonCritical:
        alarm:
          topic: ${self:service}-${opt:stage}-nonCritical-alerts-alarm
          notifications:
          - protocol: email
            endpoint: alarms@email.com

    definitions:  # these defaults are merged with your definitions
      criticalFunctionErrors:
        namespace: 'AWS/Lambda'
        metric: Errors
        threshold: 20
        statistic: Sum
        period: 60
        evaluationPeriods: 10
        comparisonOperator: GreaterThanOrEqualToThreshold
        okActions:
          - critical
        alarmActions:
          - critical
      nonCriticalFunctionErrors:
        namespace: 'AWS/Lambda'
        metric: Errors
        threshold: 1
        statistic: Sum
        period: 60
        evaluationPeriods: 10
        comparisonOperator: GreaterThanOrEqualToThreshold
        alarmActions:
          - nonCritical
    alarms:
      - criticalFunctionErrors
      - nonCriticalFunctionErrors

```
## SNS Topics

If topic name is specified, plugin assumes that topic does not exist and will create it. To use existing topics, specify ARNs or use Fn::ImportValue to use a topic exported with CloudFormation.

#### ARN support

```yaml
custom:
  alerts:
    topics:
      alarm:
        topic: arn:aws:sns:${self:region}:${self::accountId}:monitoring-${opt:stage}
```

#### Import support

```yaml
custom:
  alerts:
    topics:
      alarm:
        topic:
          Fn::ImportValue: ServiceMonitoring:monitoring-${opt:stage, 'dev'}
```

## SNS Notifications

You can configure subscriptions to your SNS topics within your `serverless.yml`. For each subscription, you'll need to specify a `protocol` and an `endpoint`.

The following example will send email notifications to `me@example.com` for all messages to the Alarm topic:

```yaml
custom:
  alerts:
    topics:
      alarm:
        topic: ${self:service}-${opt:stage}-alerts-alarm
        notifications:
          - protocol: email
            endpoint: me@example.com
```

You can configure notifications to send to webhook URLs, to SMS devices, to other Lambda functions, and more. Check out the AWS docs [here](http://docs.aws.amazon.com/sns/latest/api/API_Subscribe.html) for configuration options.

## Metric Log Filters
You can monitor a log group for a function for a specific pattern. Do this by adding the pattern key.
You can learn about custom patterns at: http://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/FilterAndPatternSyntax.html

The following would create a custom metric log filter based alarm named `barExceptions`. Any function that included this alarm would have its logs scanned for the pattern `exception Bar` and if found would trigger an alarm.

```yaml
custom:
  alerts:
    definitions:
      barExceptions:
        metric: barExceptions
        threshold: 0
        statistic: Minimum
        period: 60
        evaluationPeriods: 1
        comparisonOperator: GreaterThanThreshold
        pattern: 'exception Bar'
      bunyanErrors:
        metric: bunyanErrors
        threshold: 0
        statistic: Sum
        period: 60
        evaluationPeriods: 1
        datapointsToAlarm: 1
        comparisonOperator: GreaterThanThreshold
        pattern: '{$.level > 40}'
```

> Note: For custom log metrics, namespace property will automatically be set to stack name (e.g. `fooservice-dev`).

## Custom Naming
You can define custom naming template for the alarms. `nameTemplate` property under `alerts` configures naming template for all the alarms, while placing `nameTemplate` under alarm definition configures (overwrites) it for that specific alarm only. Naming template provides interpolation capabilities, where supported placeholders are:
  - `$[functionName]` - function name (e.g. `helloWorld`)
  - `$[functionId]` - function logical id (e.g. `HelloWorldLambdaFunction`)
  - `$[metricName]` - metric name (e.g. `Duration`)
  - `$[metricId]` - metric id (e.g. `BunyanErrorsHelloWorldLambdaFunction` for the log based alarms, `$[metricName]` otherwise)

> Note: All the alarm names are prefixed with stack name (e.g. `fooservice-dev`).

## Default Definitions
The plugin provides some default definitions that you can simply drop into your application. For example:

```yaml
alerts:
  alarms:
    - functionErrors
    - functionThrottles
    - functionInvocations
    - functionDuration
```

If these definitions do not quite suit i.e. the threshold is too high, you can override a setting without
creating a completely new definition.

```yaml
alerts:
  definitions:  # these defaults are merged with your definitions
    functionErrors:
      period: 300 # override period
      treatMissingData: notBreaching # override treatMissingData
```

The default definitions are below.

```yaml
definitions:
  functionInvocations:
    namespace: 'AWS/Lambda'
    metric: Invocations
    threshold: 100
    statistic: Sum
    period: 60
    evaluationPeriods: 1
    datapointsToAlarm: 1
    comparisonOperator: GreaterThanOrEqualToThreshold
    treatMissingData: missing
  functionErrors:
    namespace: 'AWS/Lambda'
    metric: Errors
    threshold: 1
    statistic: Sum
    period: 60
    evaluationPeriods: 1
    datapointsToAlarm: 1
    comparisonOperator: GreaterThanOrEqualToThreshold
    treatMissingData: missing
  functionDuration:
    namespace: 'AWS/Lambda'
    metric: Duration
    threshold: 500
    statistic: Average
    period: 60
    evaluationPeriods: 1
    comparisonOperator: GreaterThanOrEqualToThreshold
    treatMissingData: missing
  functionThrottles:
    namespace: 'AWS/Lambda'
    metric: Throttles
    threshold: 1
    statistic: Sum
    period: 60
    evaluationPeriods: 1
    datapointsToAlarm: 1
    comparisonOperator: GreaterThanOrEqualToThreshold
    treatMissingData: missing
```
## Additional dimensions

The plugin allows users to provide custom dimensions for the alarm. Dimensions are provided in a list of key/value pairs {Name: foo, Value:bar} 
The plugin will always apply dimension of {Name: FunctionName, Value: ((FunctionName))}
 For example:
 
```yaml
    alarms: # merged with function alarms
      - name: fooAlarm
        namespace: 'AWS/Lambda'
        metric: errors # define custom metrics here
        threshold: 1
        statistic: Minimum
        period: 60
        evaluationPeriods: 1
        comparisonOperator: GreaterThanThreshold
        dimensions:
          -  Name: foo
             Value: bar
```

```json
'Dimensions': [
                {
                    'Name': 'foo',
                    'Value': 'bar'
                },
            ]
```

## Using Percentile Statistic for a Metric

Statistic not only supports SampleCount, Average, Sum, Minimum or Maximum as defined in CloudFormation [here](http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-cw-alarm.html#cfn-cloudwatch-alarms-statistic), but also percentiles. This is possible by leveraging  [ExtendedStatistic](http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-cw-alarm.html#cfn-cloudwatch-alarms-extendedstatistic) under the hood. This plugin will automatically choose the correct key for you. See an example below:

```yaml
definitions:
  functionDuration:
    namespace: 'AWS/Lambda'
    metric: Duration
    threshold: 100
    statistic: 'p95'
    period: 60
    evaluationPeriods: 1
    datapointsToAlarm: 1
    comparisonOperator: GreaterThanThreshold
    treatMissingData: missing
```

## Collecting Custom Metrics

By default for pattern based alarms this plugin will create a single SNS topic, with 2 separate metrics feeding into it:
- `<yourfilter>ALERT`: this is set to trigger on your pattern, with a MetricValue of 1
- `<yourfilter>OK   `: this is set to trigger on *any* pattern, with a MetricValue of 0
This default behavior makes it difficult to use the custom metrics for anything other than a simple occurance filter.

The following additional arguments can be used with custom filters:
```yaml
        skipOKMetric: true
        metricValue: <custom_metric_value>
```

If `skipOKMetric` is true, then only the <yourfilter>ALERT filter will be created.
If `metricValue` is specified, then this will be used instead of "1" for the metricValue for the `<yourfilter>ALERT` filter.

So, for example, if you want to create a filter to track the value of a custom parameter from your logfile, you can create

```yaml
functions:
  my-lambda-function:
    alarms:
      - name: myCustomMetricAlarm
        namespace: 'AWS/Lambda'
        description: 'Tracks the current value of widget_cnt from cloudwatch log, alarm if a value >= 5 seen in the previous minute'
        metric: widgetCount
        threshold: 5
        statistic: Maximum
        period: 60
        evaluationPeriods: 1
        comparisonOperator: GreaterThanOrEqualToThreshold
        pattern: '{$.widget_cnt = *}'
        metricValue: '$.widget_cnt'
        skipOKMetric: true
```

So if your lambda log looks like:

```
START RequestId: xxx Version: $LATEST
Log File Message from first lambda run
IS_STALL_CNT: {"widget_cnt": 4}
END RequestId: xxx
REPORT RequestId: xxx	Duration: 103.09 ms	Billed Duration: 200 ms 	Memory Size: 512 MB	Max Memory Used: 49 MB

START RequestId: yyy Version: $LATEST
Log File Message from next lambda run
IS_STALL_CNT: {"widget_cnt": 13}
END RequestId: yyy
REPORT RequestId: yyy	Duration: 103.09 ms	Billed Duration: 200 ms 	Memory Size: 512 MB	Max Memory Used: 49 MB
```

Then you will see the metric get a value of 4, and then a value of 13 (as extracted from the JSON in the logfile).  More details on the syntax for metricValue and more complex examples can be found [here](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/FilterAndPatternSyntax.html).

## License

MIT © [A Cloud Guru](https://acloud.guru/)


[npm-image]: https://badge.fury.io/js/serverless-plugin-aws-alerts.svg
[npm-url]: https://npmjs.org/package/serverless-plugin-aws-alerts
[travis-image]: https://travis-ci.org/ACloudGuru/serverless-plugin-aws-alerts.svg?branch=master
[travis-url]: https://travis-ci.org/ACloudGuru/serverless-plugin-aws-alerts
[daviddm-image]: https://david-dm.org/ACloudGuru/serverless-plugin-aws-alerts.svg?theme=shields.io
[daviddm-url]: https://david-dm.org/ACloudGuru/serverless-plugin-aws-alerts
[coveralls-image]: https://coveralls.io/repos/ACloudGuru/serverless-plugin-aws-alerts/badge.svg
[coveralls-url]: https://coveralls.io/r/ACloudGuru/serverless-plugin-aws-alerts
