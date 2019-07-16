# Deployment Guide for LoyaltyOne
This deployment guide outlines the prerequisites and steps required to set up the monitoring dashboard for Datahub platform infrastructure in the LoyaltyOne AWS account. 
The document starts with the high level prerequites for the overall deployment and then breaks down the prerequistes and steps for the different AWS resource types.

For the purpose of this guide we have hard-coded some of the parameter values in the cloudformation template. These parameters are subject to change per the project.

## Prerequisite
1. an AWS account
2. an IAM role that has permissions to run cloudformation template

### Lambda
The lambda cloudformation template creates the lambda that is responsible for the creation of monitoring dashboard on CloudWatch. The lambda execution will be an event based scheduled activity.

The dashboard creator lambda reads a monitoring configuration file from an S3 bucket treated as a storage location here.
This file includes name of the AWS service along with its metric specifications in a pre-defined format which is to be displayed on the dashboard.
Metric specifications such as region, metricName and refreshIntervalInSec are configurable for each individual metric.

This lambda also reads a DB configuration file from an S3 bucket. It will read the activity/file pending count of each service type on state machine from the database table and upload it as a metric on cloudwatch.
This will be the same configuration file that will be used for ECS autoscaling solution.

Another custom metric for the state of transfer server is also being published via this lambda onto AWS. Mapping for server state onto the cloudwatch metrics is done as follows:

   | Property Name     | Property Value     |
   | ----------------  |:------------------ |
   | ONLINE            |  1                 |
   | OFFLINE           |  0                 |

If the published metric value is 1 on the graph, it shows that the server is online. If the published metric value is 0 onto the graph (any downfall), it shows that the server is offline.
Based on the metric specifications, metric data will be fetched from AWS and a dashboard will be configured to display this data.

#### Prerequisite
1. Create/Use an existing S3 bucket to store the lambda code artifacts & configuration files for metrics. All bucket properties should be disabled.
2. Create a folder named `monitoring` under the bucket for configuration files & choose an existing folder `lambdas` that stores lambda zips. Choose the encryption setting for the object as `None (Use bucket settings)`.
   Upload the lambda zip file & configuration file under the bucket's intended folder.

#### Deployment Steps
1. Create the lambda cloudformation stack using the template - dashboard_creator_lambda.yaml
2. Fill in the stack parameters with the values in the table below:
    * To learn more about the parameters and what they are used for, check the template file in step 1.
    
   | Parameter Name                                     | Parameter Value                                                                                |
   | -------------------------------------------------  |:---------------------------------------------------------------------------------------------- |
   | LambdaCodeS3Bucket                                 | name of the S3 artifact bucket (see Prerequisite)                                              |
   | DashboardCreatorLambdaCodeS3Key                    | `lambdas/datalake_dashboard_creator_lambda.zip`                                                |
   | DashboardCreatorConfigurationS3Bucket              | name of the S3 bucket containing configuration file (see Prerequisite)                         |
   | DashboardCreatorMonitoringConfigurationFileS3Key   | `monitoring/MonitoringConfiguration.json`                                                      |
   | DashboardCreatorDBConfigurationFileS3Key           | `monitoring/AutoscalingConfiguration.json`                                                     |
   | DashboardName                                      | name of the CloudWatch Dashboard                                                               |
   | DashboardCreatorLambdaRate                         | frequency with which the cloudwatch event will trigger the lambda function eg: rate(10 minutes)|

3. Select "I acknowledge Capabilities" checkbox as AWS CloudFormation will create IAM resources.
4. Post stack creation, you should see the lambda under 'Functions' on the AWS Lambda page.
5. IAM role, policies, schedule rule and permission will be created for AWS lambda to access resources such as the S3 bucket and CloudWatch Dashboard.

## Post-deployment Notes
1. At this point you should have the Lambda function created in your AWS account.
2. Based on the schedule rule, lambda function will automatically get executed.
3. Post lambda execution, navigate to AWS Cloudwatch Dashboard page.
4. On AWS CloudWatch page, you should see a dashboard created with the name specified in the stack creation template.
5. Select the dashboard and it will display the metrics for each AWS service as mentioned in the configuration file along with the activity/file pending count of each service type on state machine.
6. Adjust the time interval from the top section on the righthand side for the data that you may want to consider for dashboard.

Note: Many other AWS services also publish metrics to CloudWatch. For information about the services, metrics and dimensions, see the documentation available here - https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/aws-services-cloudwatch-metrics.html
