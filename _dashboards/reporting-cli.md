---
layout: default
title: Creating reports with the Reporting CLI
nav_order: 75
---

# Creating reports with the Reporting CLI

You can programmatically create dashboard reports in PDF or PNG format with the Reporting CLI without using OpenSearch Dashboards or the Reporting plugin. This allows you to create reports automatically within your email workflows.

If you want to download a CSV file, you need to have the Reporting plugin installed.
{: .note }

For any dashboard view, you can request a report in PNG or PDF format to be sent to an email address. This can be useful for sending reports to multiple email recipients with an email alias. The only dashboard view that supports creating a CSV report is the **Discover** view.

With the Reporting CLI, you can specify options for your report in the command line. The report is sent to an email address as a PDF attachment by default. You can also request a PNG image or a CSV file with the `--formats` argument or embed the image in the email body instead of attaching it.

You can download the report to the directory in which you are running the Reporting CLI, or you can email the report by specifying Amazon Simple Email Service (Amazon SES) or SMTP for the email transport option.

You can connect to OpenSearch with any of the following authentication types:

- **Basic** – Basic HTTP authentication. Use `-a basic`.
- **Cognito** – Authentication through Amazon Cognito. Use `-a cognito`.
- **SAML** – Authentication between an identity provider and a service provider. Use `-a saml`. Okta provides the SAML third-party authentication.
- **No auth** – No authentication. Use `-a none`. Authentication defaults to No auth if the `-a` flag is not specified.

To learn more about Amazon Cognito, see [What is Amazon Cognito?](https://docs.aws.amazon.com/cognito/latest/developerguide/what-is-amazon-cognito.html).

<!--
### Bypass authentication option

The reporting CLI tool allows you to integrate it into your own workflow or environment so that you can bypass authentication or potential security issues. For example, if you use the Reporting CLI tool within an AWS Lambda instance, no security issues would occur as long as you run the Reporting plugin in OpenSearch Dashboards. In this case, you would use "No auth" to bypass the authentication process. To specify "No Auth" use `--auth none` in your request. Lambda users should test to make sure they can bypass access to Dashboards without credentials using No Auth.  

To get a list of all options, see [Reporting CLI options](#reporting-cli-options).
-->

## Downloading and installing the Reporting CLI tool

You can download and install the Reporting CLI tool from either the npm software registry or the OpenSearch.org [Download & Get Started](https://opensearch.org/downloads.html) page. Refer to the following sections for instructions.

To learn more about the npm software registry, see the [npm](https://docs.npmjs.com/about-npm) documentation.

### Downloading and installing the Reporting CLI from npm

To download and install the Reporting CLI from npm, run the following command to initiate installation:

```
npm i @opensearch-project/opensearch-reporting-cli
```

### Downloading and installing the Reporting CLI from OpenSearch.org

You can download the `opensearch-reporting-cli` tool from the OpenSearch.org [Command Line Tools](https://opensearch.org/downloads.html#command-line-tools) section.

Next, run the following command to install the .tar archive:

```
npm install -g opensearch-reporting-cli-1.0.0.tgz
```

## Creating and requesting a visualization report

First, you need to get the URL for the visualization that you want to download as an image file or PDF.

To generate a visualization report, you need to specify the Dashboards URL.

Open the visualization for which you want to generate a report, and select **Share >  Permalinks > Generate link as Shapshot > Short URL > Copy link**, as shown in the following image.

![Copy link]({{site.url}}{{site.baseurl}}/images/dashboards/dash-url.png)

You will need to add the URL with the `-u` argument when you request the report in the CLI.

### Example: Requesting a PNG file

The following command requests a report in PNG format with basic authentication and sends the report to an email address using Amazon SES:

```
opensearch-reporting-cli -u https://localhost:5601/app/dashboards#/view/7adfa750-4c81-11e8-b3d7-01146121b73d -a basic -c admin:1234 -e ses -s <email address>  -r <email address> -f png
```

### Example: Requesting a PDF file

The following command requests a PDF file and specifies the recipient's email address:

```
opensearch-reporting-cli -u https://localhost:5601/app/dashboards#/view/7adfa750-4c81-11e8-b3d7-01146121b73d -a basic -c admin:Test@1234 -e ses -s <email address> -r <email address> -f pdf
```

Upon success, the file will be sent to the specified email address. The following image shows an example PDF report.

![PDF example]({{site.url}}{{site.baseurl}}/images/dashboards/cli-pdf-report.png)

### Example: Requesting a CSV file

The following command generates a report that contains all table content in CSV format and sends the report to an email address using Amazon SES transport:

```
opensearch-reporting-cli -u https://search-basic-auth2-3-b477rrluhlckrmwmtvp6k3xxsi.us-west-2.es.amazonaws.com/_dashboards/goto/f166434919d2d8d3708a4705b6f2ed2d?security_tenant=global -f csv -a basic -c admin:Test1234 -e ses -s <email address> -r <email address>
```

Upon success, the email will be sent to the specified email address with the CSV file attached.

## Automatically run reports with a cron job

You can run reports automatically by setting up a cron job. OpenSearch Reporting CLI supports any cron job use case, such as setting a specified time of day.

### Prerequisites

You need a machine with Ubuntu and root access privileges.

If cron is not installed by default, run the following command:

```
sudo apt install cron
```

### Step 1: Install the reporting CLI

Install the Reporting CLI by running the following command:

```
npm i @opensearch-project/opensearch-reporting-cli
```

### Step 2: Specify the details for the report

Open the crontab editor by running the following command:

```
crontab -e
```
In the crontab editor, enter the report request. For example, the following example shows a cron report that runs every day at 8am.

```
0 8 * * * opensearch-reporting-cli -u https://playground.opensearch.org/app/dashboards#/view/084aed50-6f48-11ed-a3d5-1ddbf0afc873 -e ses -s <sender_email> -r <recipient_email>
```

## Scheduling reports with AWS Lambda

You can use AWS Lambda with the Reporting CLI tool to specify an AWS Lambda function to trigger the report generation.

This requires that you use an AM64 system and Docker.

### Prerequisites

To use the Reporting CLI with AWS Lambda, you need to do the following preliminary steps.

- Get an AWS account. For instructions, see [Creating an AWS account](https://docs.aws.amazon.com/accounts/latest/reference/manage-acct-creating.html) in the AWS Account Management reference guide.
- Get a Dockerfile to assemble an image. For Docker instructions, see [Dockerfile reference](https://docs.docker.com/engine/reference/builder/).
- Set up an Amazon Elastic Container Registry (ECR). For instructions, see [Getting started with Amazon ECR using the AWS Management Console](https://docs.aws.amazon.com/AmazonECR/latest/userguide/getting-started-console.html).


### Step 1: Create a container image with a Dockerfile

You need to assemble the image by running a Dockerfile. When you run the Dockerfile, it downloads the OpenSearch artifact required to use the Reporting CLI.

Copy the following sample configurations into a Dockerfile.

```dockerfile

# Define function directory
ARG FUNCTION_DIR="/function"

# Base image of the docker container
FROM node:lts-slim as build-image

# Include global arg in this stage of the build
ARG FUNCTION_DIR

# AWS Lambda runtime dependencies
RUN apt-get update && \
    apt-get install -y \
        g++ \
        make \
        unzip \
        libcurl4-openssl-dev \
        autoconf \
        automake \
        libtool \
        cmake \
        python3 \
        libkrb5-dev \
        curl

# Copy function code
WORKDIR ${FUNCTION_DIR}
RUN curl -LJO https://artifacts.opensearch.org/reporting-cli/opensearch-reporting-cli-1.0.0.tgz
RUN tar -xzf opensearch-reporting-cli-1.0.0.tgz
RUN mv package/* .
RUN npm install && npm install aws-lambda-ric

# Build Stage 2: Copy Build Stage 1 files in to Stage 2. Install chromium dependencies and chromium.
FROM node:lts-slim
# Include global arg in this stage of the build
ARG FUNCTION_DIR
# Set working directory to function root directory
WORKDIR ${FUNCTION_DIR}
# Copy in the build image dependencies
COPY --from=build-image ${FUNCTION_DIR} ${FUNCTION_DIR}

# Install latest chrome dev package and fonts to support major char sets (Chinese, Japanese, Arabic, Hebrew, Thai and a few others)
# Note: this installs the necessary libs to make the bundled version of Chromium that Puppeteer installs, work.
RUN apt-get update \
    && apt-get install -y wget gnupg \
    && wget -q -O - https://dl-ssl.google.com/linux/linux_signing_key.pub | apt-key add - \
    && sh -c 'echo "deb [arch=amd64] http://dl.google.com/linux/chrome/deb/ stable main" >> /etc/apt/sources.list.d/google.list' \
    && apt-get update \
    && apt-get install -y google-chrome-stable fonts-ipafont-gothic fonts-wqy-zenhei fonts-thai-tlwg fonts-kacst fonts-freefont-ttf libxss1 \
      --no-install-recommends \
    && apt-get remove -y google-chrome-stable \
    && rm -rf /var/lib/apt/lists/*

ENTRYPOINT ["/usr/local/bin/npx", "aws-lambda-ric"]

ENV HOME="/tmp"
CMD [ "/function/src/index.handler" ]

```

Next, run the following build command within the same directory that contains the Dockerfile:

```
docker build -t opensearch-reporting-cli .
```

### Step 2: Create a private repository with Amazon ECR

Follow the instructions in the Amazon ECR user guide to create a repository with the name `opensearch-reporting-cli`. For instructions, see [Getting started with Amazon ECR using the AWS Management Console](https://docs.aws.amazon.com/AmazonECR/latest/userguide/getting-started-console.html).

In addition to the Amazon EC2 instructions, you need to make the following adjustments for the Reporting CLI to function properly:

Navigate to the test function **Configuration** tab to modify two required settings.
- Make sure to set **Timeout** to at least 5 minutes to allow the reporting CLI to generate the report.
- Change the setting for **Ephemeral storage** to 1024MB. The default setting is not a sufficient storage amount.
{: .note }

### Step 3: Run the push commands

You need to get several commands from the AWS EC2 Console to run within the Dockerfile directory, as shown in the following image:

![Amazon ECR console view push commands]({{site.url}}{{site.baseurl}}/images/dashboards/push-commands.png)

1. In the AWS ECR console, go to **Repositories** and choose **opensearch-reporting-cli**.
1. Choose **view push commands**.
1. In **Push commands for opensearch-reporting-cli**, copy each push command and run it in the Dockerfile directory.

### Step 4: Create a lambda function with the container image

1. In **Select container image**, select the container `opensearch-reporting-cli` from the list, and then choose **Select image**.
1. Specify a name for the function.
1. In **Architecture**, choose **x86_64**.
1. Choose **Create function**.
1. Go to **Lambda function > Configuration > General configuration> Edit timeout** and set the timeout in lambda to 5 minutes.
1. Set the memory size to at least 1024MB.
1. Next, test the function either by providing fixed values or variable values in a JSON file.

- If the function contains fixed values, such as email address you do not need a JSON file. You can specify an environment variable in AWS Lambda.
- If the function takes a variable key-pair value, then you need to specify the values in a JSON file.
{: .note }

 The following example shows fixed values provided for the sender and recipient email addresses:

```json
{
  "url": "https://playground.opensearch.org/app/dashboards#/view/084aed50-6f48-11ed-a3d5-1ddbf0afc873",
  "transport": "ses",
  "from": "sender@amazon.com", 
  "to": "recipient@amazon.com", 
  "subject": "Test lambda docker image"
}
```



### Step 5: Add the trigger to start the AWS Lambda function

Set the trigger to start running the report. AWS Lambda can use any AWS service as a trigger, such as SNS, S3, or an AWS CloudWatch EventBridge.

1. In the **Triggers** section, choose **Add trigger**.
1. Select a trigger from the list. For example, you can set an AWS CloudWatch Event. To learn more about Amazon ECR events you can schedule, see [Sample events from Amazon ECR](https://docs.aws.amazon.com/AmazonECR/latest/userguide/ecr-eventbridge.html#ecr-eventbridge-bus).
1. Choose **Test** to initiate the function.

### (Optional) Step 6: Add the role permission for Amazon SES

1. Select **Configuration** and choose **Excecution role**.
1. In **Summary**, choose **Permissions**.
1. Select **{}JSON** to open the JSON policy editor.
1. Add the permissions for the Amazon SES resource that you want to use.

The following example provides the resource ARN for the send email action:

```json
{
            "Effect": "Allow",
            "Action": [
                "ses:SendEmail",
                "ses:SendRawEmail"
            ],
            "Resource": "arn:aws:ses:us-west-2:840298398745:identity/username@amazon.com"
        }
```

## Getting help

To get a list of all available CLI arguments, run the following command:

``` 
$ opensearch-reporting-cli -h
```


## Reporting CLI options

You can use any of the following arguments with the `opensearch-reporting-cli` tool.

| Argument      | Description       | Acceptable values and usage | Environment variable
:--------------------- | :--- | :--- |
`-u`, `--url` | The URL for the visualization. | Obtain from OpenSearch Dashboards > Visualize > Share > Permalinks > Copy link. | OPENSEARCH_URL
`-a`, `--auth` | The authentication type for the report. | You can specify either Basic `basic`, Cognito `cognito`, SAML `saml`, or No Auth `none`. If no value is specified, the Reporting CLI tool defaults to no authentication, type `none`. Basic, Cognito, and SAML require credentials with the `-c` flag. | N/A
`-c`, `--credentials` | The OpenSearch login credentials. | Enter your username and password separated by a colon. For example, username:password. Required for Basic, Cognito, and SAML authentication types. | OPENSEARCH_USERNAME and OPENSEARCH_PASSWORD
`-t`, `--tenant` | The tenants in OpenSearch Dashboards. | The default tenant is private.| N/A
`-f`, `--format` | The file format for the report. | Can be either `pdf`, `png`, or `csv`. The default is `pdf`.| N/A
`-w`, `--width` | The window width in pixels for the report. | Default is `1680`.| N/A
`-l`, `--height` | The minimum window height in pixels for the report. | Default is `600`. | N/A
`-n`, `--filename` | The file name of the report. | Default is `reporting`. | OPENSEARCH_FILENAME
`-e`, `--transport` | The transport mechanism for sending the email. | For Amazon SES, specify `ses`. Amazon SES requires an AWS configuration on your system to store the credentials. For SMTP, use `smtp` and also specify the login credentials with `--smtpusername` and `--smtppassword`. | OPENSEARCH_TRANSPORT
`-s`, `--from` | The email address of the sender. | For example, `user@amazon.com`. | OPENSEARCH_FROM
`-r`, `--to` | The email address of the recipient. | For example, `user@amazon.com`. | OPENSEARCH_TO
`--smtphost` | The hostname of the SMTP server. | For example, `SMTP_HOST`. | OPENSEARCH_SMTP_HOST
`--smtpport` | The port for the SMTP connection. | For example, `SMTP_PORT`. | OPENSEARCH_SMTP_PORT
`--smtpsecure` | Specifies to use TLS when connecting to the server. | For example, `SMTP_SECURE`. | OPENSEARCH_SMTP_SECURE
`--smtpusername` | The SMTP username.| For example, `SMTP_USERNAME`. | OPENSEARCH_SMTP_USERNAME
`--smtppassword` | The SMTP password.| For example, `SMTP_PASSWORD`. | OPENSEARCH_SMTP_PASSWORD
`--subject` | The email subject text encased in quotes. | Can be any string. The default is "This is an email containing your dashboard report". | OPENSEARCH_EMAIL_SUBJECT
`-h`, `--help` | Specifies to display the list of optional arguments from the command line. | N/A

## Using environment variables with the Reporting CLI

Instead of explicitly providing values in the command line, you can save them as environment variables. The Reporting CLI reads environment variables from the current directory inside the project.

To set the environment variables in Linux, use the following command:

```
export NAME=VALUE
```

Each line should use the format `NAME=VALUE`.
Each line that starts with a hashtag (#) is considered to be a comment.
Quotation marks (") don't get any special handling.

Values from the command line argument have higher priority than the environment file. For example, if you add the file name as *test* in the *.env* file and also add the `--filename report` command option, the generated report's name will be *report*.
{: .note }

### Example: Requesting a PNG report with environment variables set

The following command requests a report with basic authentication in PNG format:

```
opensearch-reporting-cli --url https://localhost:5601/app/dashboards#/view/7adfa750-4c81-11e8-b3d7-01146121b73d --format png --auth basic --credentials admin:admin
```

Upon success, the report will download to the current directory.

### Using Amazon SES to request an email with a report attachment

To use Amazon SES as the email transport mechanism, the following prerequisites apply:

- If a URL contains an exclamation point (!), then the history expansion needs to be disabled temporarily. The sender's email address must be verified by [Amazon SES](https://aws.amazon.com/ses/). The [AWS Command Line Interface (AWS CLI)](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-welcome.html) is required to interact with Amazon SES. To configure basic settings used by the AWS CLI, see [Quick configuration with `aws configure`](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-quickstart.html#cli-configure-quickstart-config) in the AWS Command Line Interface user guide.
- Amazon SES transport requires the `ses:SendRawEmail` role:

```
{
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "ses:SendRawEmail",
      "Resource": "*"
    }
  ]
}
```

The following command requests an email with the report attached:

```
opensearch-reporting-cli --url https://localhost:5601/app/dashboards#/view/7adfa750-4c81-11e8-b3d7-01146121b73d --transport ses --from <sender_email_id> --to <recipient_email_id>
```

The following command uses default values for all other options. You can also set `OPENSEARCH_FROM`, `OPENSEARCH_TO`, and `OPENSEARCH_TRANSPORT` in your .env file and use the following command:

```
opensearch-reporting-cli --url https://localhost:5601/app/dashboards#/view/7adfa750-4c81-11e8-b3d7-01146121b73d
```

To modify the body of your email, you can edit the *index.hbs* file.

### Example: Sending a report to an email address with SMTP

To send a report to an email address with SMTP transport, you need to set the options `OPENSEARCH_SMTP_HOST`, `OPENSEARCH_SMTP_PORT`, `OPENSEARCH_SMTP_USER`, `OPENSEARCH_SMTP_PASSWORD`, and `OPENSEARCH_SMTP_SECURE` in your .env file.

Once the transport options are set in your .env file, you can send the email using the following command:

```
opensearch-reporting-cli --url https://localhost:5601/app/dashboards#/view/7adfa750-4c81-11e8-b3d7-01146121b73d --transport smtp --from <sender_email_id> --to <recipient_email_id>
```

You can choose to set options using either your *.env* file or the command line argument values in any combination. Make sure to specify all required values to avoid errors.

To modify the body of your email, you can edit the *index.hbs* file.

## Environment variable limitations

The following limitations apply to environment variable usage with the Reporting CLI:

- Supported platforms are Windows x86, Windows x64, Mac Intel, Mac ARM, Linux x86, and Linux x64.
  
  For any other platform, users can take advantage of the *CHROMIUM_PATH* environment variable to use custom Chromium.

- If a URL contains an exclamation point (!), then the history expansion needs to be disabled temporarily. Depending on which shell you are using, you can disable history expansion using one of the following commands:

  * For bash, use `set +H`. 
  * For zsh, use `setopt nobanghist`.

  Alternatively, you can add a URL value as an environment variable using this format: `URL="<url-with-!>"`.

- All command options only accept lowercase letters.

## Troubleshooting

To resolve **MessageRejected: Email address is not verified**, see [Why am I getting a 400 "message rejected" error with the message "Email address is not verified" from Amazon SES?](https://aws.amazon.com/premiumsupport/knowledge-center/ses-554-400-message-rejected-error/) in the AWS Knowledge Center.