---
title: 'Resume Job Desc Comparison With OpenAI'
date: 2024-02-10T09:45:15-06:00
draft: false
tags:
- aws
- openai
categories:
- cloud
---
## Preamble

The objective of this project was to use provide a job description and resume to the [OpenAI API](https://platform.openai.com/docs/overview). The API would then return a short paragraph stating whether the resume was suitable for the job. To accomplish this, a single webpage was created that makes a request to an [AWS Lambda function](https://aws.amazon.com/pm/lambda/) which processes the request and returns OpenAI's response.

For brevity the code examples that follow will only contain code necessary to explain the concept being discussed. When available, relevant links to documentation, or helpful articles will be provided.

The author and creator of the site is not a web development professional or an expert in AWS. Code may not follow best practices, but it works! It is not suggested you follow these examples in a real working environment. The project was made as an assignment for classes being given at [IT Expert System](https://www.itexps.net/) out of Schaumburg, Illinois.
## Technology
To accomplish this the following technologies were used.
- [Tailwindcss](https://tailwindcss.com/) and [Tailwindui components](https://tailwindui.com/) for page and form styling (including typography and form plugins)
- [Alpinejs](https://alpinejs.dev/) for interactivity and making the request
- [AWS API Gateway](https://aws.amazon.com/api-gateway/) for handling the API and requests
- [AWS Lambda](https://aws.amazon.com/pm/lambda/) with a [Python 3.9](https://www.python.org/) runtime to process the request
	- OpenAI python library
	- Python's built in JSON library
	- Pythons built in os
- [AWS S3 Bucket](https://aws.amazon.com/pm/serv-s3/) for hosting the webpage and storing the layers `.zip` file

## Creating the Page Locally
The first step was to create a `src` directory, a blank `index.html` file, and blank `input.css` file. Then install Tailwindcss following the [install documentation](https://tailwindcss.com/docs/installation). Some basic content was added to the `index.html` and a link to the `output.css` file was added to the `<head>` to link up the output of the npx command (found in the Tailwind installation instructions) that is run to watch for any changes.

```bash
npx tailwindcss -i ./src/input.css -o ./src/output.css --watch
```

A few Tailwind classes were added to make sure that the set-up was running properly. Example below:

```html
<!DOCTYPE html>
<html>

<head>
	<meta charset="UTF-8">
	<meta name="viewport" content="width=device-width, initial-scale=1.0">
	<title>Job Fit Analysis</title>
	<link href="output.css" rel="stylesheet">
</head>

<body>
	<main class="container-sm mx-auto px-4 font-sans">
			<h1 class="text-6xl font-bold text-center">
				Job fit analysis
			</h1>
	</main>
</body>
</html>
```

Now that the page has been created we can view it in our browser. To do this you can right click on your file `index.html` in a finder window (mac) or file explorer (windows), then select `Open with>Firefox`
## Create Lambda function
The ultimate purpose of the website is for a user to see if their resume is suitable for a job description. The user will provide a resume and job description via a form, and that data will be sent via a request to an API endpoint. The API will be created and managed with an AWS API Gateway. The endpoint will then be associated with a Lambda function, written in Python, that will handle the processing of the data and returning a response.

First let's take a brief moment to look at what [Lambda functions](https://aws.amazon.com/lambda/) are. 

>AWS Lambda is a serverless, event-driven compute service that lets you run code for virtually any type of application or backend service without provisioning or managing servers.

This project was a perfect use case for Lambda. The site only does one thing! It asks OpenAI to compare a resume and job description. Provisioning an entire server for this would be wasteful and expensive, as it needs to be running all times. Making an API call to the endpoint triggers the event required (event-driven from above) to run the code.

As is the case with many AWS services, a Lambda function can be created using the AWS Console, [AWS cli](https://aws.amazon.com/cli/), or [AWS SDKs](https://aws.amazon.com/developer/tools/). The lambda function for this project was created with the [Lambda AWS console](https://us-east-2.console.aws.amazon.com/lambda/home?region=us-east-2#/begin), with the settings found below. 

- __Author from scratch selected__
- Function name: __compare-openai__
- Runtime: __Python 3.9__ (*why 3.9 and not another version? It has to do with layers which will be covered later*)
- Architecture: __x86_64__

A more detailed tutorial on how to create a function with the console can be found in the AWS documentation [here](https://docs.aws.amazon.com/lambda/latest/dg/getting-started.html#getting-started-create-function).

AWS directs you to an online code editor that has some boiler plate code that returns a status code and a body with "Hello from lambda!" 

```python
import json

def lambda_handler(event, context):
    # TODO implement
    return {
        'statusCode': 200,
        'body': json.dumps('Hello from Lambda!')
    }
```

To ensure that a function is available it needs to be saved (cmd+s) and then deployed by clicking the button labeled "Deploy" found above the code editor. The function was tested after deployment and returned the following, as expected: 

```json
{
  "statusCode": 200,
  "body": "\"Hello from Lambda!\""
}
```

Of course the goal was to do more with the function than just return a status code and message. The function needed to communicate with the OpenAI API and run something that OpenAI calls "chat completions". All Lambda functions require a __lambda handler__ that takes an event and a context. For the website the event will be a request with a json payload containing the resume and job description. At this point in the process, it was only necessary to have the function return *anything* from the OpenAI API. The [OpenAI documentation](https://platform.openai.com/docs/quickstart?context=python) provides an example where the system acts as poet and is prompted to compose a poem. Combining the lambda requirements and the basic code from OpenAI the below code was created for testing.

```python
import json
from openai import OpenAI

def lambda_handler(event, context):
    # TODO implement
    client = OpenAI()
    completion = client.chat.completions.create(
    model="gpt-3.5-turbo",
    messages=[
        {
            "role": "system",
            "content": "You are a poetc assistant, skilled in explaining complex programming concepts with creative flair.",
        },
        {
            "role": "user",
            "content": "Compose a poem that explains the concept of recursion in programming.",
        },
		],
	)
   
    return {
        'statusCode': 200,
        'body': completion.choices[0].message.content
    }
```

Now, those readers with a keen eye and experience with Python might be curious how the the above code will import a library that has not been installed at any point during the process. Additionally, the description of Lambda functions from AWS stated that Lambda "lets you run code for virtually any type of application or backend service __without provisioning or managing servers__." Where would the `openai` library be installed? The answer is something called "[Layers](https://docs.aws.amazon.com/lambda/latest/dg/chapter-layers.html)". 

Without utilizing a layer testing the above code returned the following error, in which it clearly states that there is no module name openai.

```python
Response
{
  "errorMessage": "Unable to import module 'lambda_function': No module named 'openai'",
  "errorType": "Runtime.ImportModuleError",
  "requestId": "a7f0ddcc-de64-4b72-9356-6eead34085ca",
  "stackTrace": []
}
```

Layers are described in the documentation this way:

>A Lambda layer is a .zip file archive that contains supplementary code or data. Layers usually contain library dependencies, a [custom runtime](https://docs.aws.amazon.com/lambda/latest/dg/runtimes-custom.html), or configuration files.

__Python version 3.9__ has been mentioned twice thus far in this write up. First in the technology section and again in the runtime settings earlier in this section. Creating the required `.zip` file for the layer is the reason for using this specific version of python. As stated in the documentation

>The first step to creating a layer is to bundle all of your layer content into a .zip file archive. Because Lambda functions run on [Amazon Linux](https://docs.aws.amazon.com/lambda/latest/dg/lambda-runtimes.html), your layer content must be able to compile and build in a Linux environment. If you build packages on your local Windows or Mac machine, you’ll get output binaries for that operating system by default. These binaries may not work properly when you upload them to Lambda.

The final two sentences are most certainly true as attempts to create the required file on a Mac ended with errors about missing modules called pydantic_core. If you are without access to a linux based server one can be spun up in various cloud services for very little money (e.g. [AWS EC2](https://aws.amazon.com/pm/ec2/), [Digital Ocean](https://try.digitalocean.com/cloud/)). It is important that the Python version that is used to install libraries is the same Python version that is selected for the runtime of the Lambda function. Version 3.9 was chosen because the author used the AWS browser based terminal [CloudShell](https://aws.amazon.com/cloudshell/) to install libraries, zip archive them, and copy them to an S3 bucket for Lambda's access.

Before anything can be copied to a S3 Bucket one had to be created. It is worth mentioning that all three AWS services used in this project are all regional. If the reader is following along you should create the S3 bucket used to store the `.zip` file and the following API Gateway in the same region as the Lambda function. Information about AWS Regions can be found [here](https://aws.amazon.com/about-aws/global-infrastructure/regions_az/).

Once the S3 bucket was created ([documentation](https://docs.aws.amazon.com/AmazonS3/latest/userguide/creating-bucket.html)) the following commands were executed in CloudShell, again taking care that the shell matches the region. In these commands the `openai` library is installed and archived in the proper format as stated by the [packaging layers documentation](https://docs.aws.amazon.com/lambda/latest/dg/packaging-layers.html).

```bash
# Make a directory for the packages
mkdir packages  
cd packages

# Create a python virtual environment and activate it
python3 -m venv venv  
source venv/bin/activate

# Create the required python named dirctory
mkdir python  
cd python

# install openai libary
pip install openai -t .

# Remove any unnecessary files
rm -rf *dist-info

# Return to packages directory
rm -rf *dist-info

# Archive the python directory
zip -r lambda-package.zip python

# Use aws cli to copy archived directory to S3 Bucket
aws s3 cp lambda-package.zip  s3://your-s3-bucket-name/
```


After the archive was copied to the bucket a layer was added to the Lambda function. Access to creating layers can be found in the console by navigating to the Lambda function and selecting the layers option from the menu on the left and then selecting "Create layer". 

1. Give the layer a name
2. select Upload a file from Amazon S3 and enter the uri for the `.zip` file (*e.g.* s3://your-s3-bucket-name//lambda-package.zip)
3. select __x86_64__
4. Choose compatible runtimes Python 3.9
5. Create

The Layer was then added to the function by navigating to the layer in the console and scrolling down to the layers section and clicking "Add layer".

1. Select Custom layer option
2. select the layer that was just created 

Now that the function had access to the required Python libraries there was another crucial piece that needed to be added. The OpenAI API requires an API Key. This can be obtained by creating an OpenAI account. An API key can be generated by logging into an OpenAI account and navigating to [this page](https://platform.openai.com/api-keys).

To allow the Lambda function access to the key it was put in the environment variables section of the Lambda function. This section can be found in the __configuration tab__ found just below the function visual layout in the AWS Console. Add the API key with the key name `OPENAI_API_KEY` and the value of the Open API key obtained from OpenAI. 

Two lines were then added to the function code to use the newly created `OPENAI_API_KEY`. The os library was imported at the top, and the environment variable was added within the code block.

```python
import json
# NEW LINE ADDED
import os
from openai import OpenAI

def lambda_handler(event, context):
    # Give the function access to the API Key
    os.environ.get('OPENAI_AI_KEY')
    client = OpenAI()
    completion = client.chat.completions.create(
    model="gpt-3.5-turbo",
    messages=[
        {
            "role": "system",
            "content": "You are a poetc assistant, skilled in explaining complex programming concepts with creative flair.",
        },
        {
            "role": "user",
            "content": "Compose a poem that explains the concept of recursion in programming.",
        },
		],
	)
   
    return {
        'statusCode': 200,
        'body': completion.choices[0].message.content
    }
```

Running the test again with this code should return a poem "that explains the concept of recursion in programming".

This code was modified later to fit the needs of the project, but before that was done an API Gateway was created so that the website has an endpoint to reach.

## API Gateway
Creating an API Gateway is a relatively easy process, but before we get into the details let's take a look at how AWS describes API Gateways. From the main [AWS API Gateway](https://aws.amazon.com/api-gateway/) site:

>Amazon API Gateway is a fully managed service that makes it easy for developers to create, publish, maintain, monitor, and secure APIs at any scale. APIs act as the "front door" for applications to access data, business logic, or functionality from your backend services. Using API Gateway, you can create RESTful APIs and WebSocket APIs that enable real-time two-way communication applications. API Gateway supports containerized and serverless workloads, as well as web applications.

According to this description an API Gateway is exactly what was needed for this small project. We needed something to act as "front door" to access our business logic (the Lambda function). We needed to have a RESTful API endpoint to our serverless workload. 

The following steps were done in the AWS Console to create an API Gateway.

__From the main API gateway page of the console__
Create API
- select Rest API
	- Build
	- Select New API
	- name the API (this will appear in the url)
	- select API endpoint type: Regional


A stage is needed to deploy, so that was created next Create a stage

__Sign in to the API Gateway console at [https://console.aws.amazon.com/apigateway](https://console.aws.amazon.com/apigateway)__.
- Choose a REST API. 
- In the main navigation pane, choose **Stages** under an API.
- From the **Stages** navigation pane, choose **Create stage**.
- For **Stage name**, enter a name, for example, `prod`.
- (Optional). For **Description**, enter a stage description.
- For **Deployment**, select the date and time of the existing API deployment you want to associate with this stage.
- Under **Additional settings**, you can specify additional settings for your stage.
- Choose **Create stage**.

After the API Gateway was created the next step was to create a resource and add a method ("POST") to the resource. From the [documentation tutorial](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-create-api-as-simple-proxy-for-lambda.html#api-gateway-create-api-as-simple-proxy-for-lambda-build)

>__To create a resource__

>1. Select the **/** resource, and then choose **Create resource**.
 >   
>2. Keep **Proxy resource** turned off.
>    
>3. Keep **Resource path** as `/compare-resume`.
 >   
>4. For **Resource name**, enter `helloworld`.
>    
>5. **CORS (Cross Origin Resource Sharing)** turned on.
 >   
>6. Choose **Create resource**.

After this a POST method was created (examples in the same documentation). CORS was enabled again on the resource (all methods) and the API was deployed to the production stage.

From the menu on the left of the API Gateway console __Stages__ has a the invoke url. This is the url that will be passed to the `fetch()` function in our webpage Javascript 
## Changing the Lambda Function to Desired Request From OpenAI
Now that there is an API endpoint to handle post requests, the Lambda function needed to be updated to handle the json payload that will be coming from the POST request. Below you will find a simplified version of the final function used. The reader can adjust the request to whatever suits their needs. The changes happen in the list that called messages that is sent to OpenAI. Asking OpenAI to act as a friendly recruiter and is prompted to review the resume and job description for analysis. One key thing to note that all Lambda functions receive an event as a parameter. The data can be accessed by using the following syntax `event['key_name_in_json_payload']` so in this case it is `event['resume']`.

Another note is that the Lambda function returns a body that is 
```
completion.choices[0].message.content
```

This syntax is how the response from the OpenAI API is accessed.

```python
import json
import os
from openai import OpenAI

def lambda_handler(event, context):
 
    api_key = os.environ.get('OPENAI_AI_KEY')
    
    client = OpenAI()
    
    try:
        system_message = f"You are a helpful assistant specialized in {event['jobDesc']}."
        messages = [
                {
                    "role": "system", 
                    "content": system_message,
                },
                {
                    "role": "user",
                    "content": f"Is this resume suitable for the job? Job description: {event['jobDesc']}, Resume: {event['resume']}",
                },
            ]
        completion = client.chat.completions.create(
        model="gpt-3.5-turbo",
        messages = messages
        )
        
        response = completion.choices[0].message.content
        
        return {
            'statusCode': 200,
            'body': response
        }
    except Exception as e:
        error_message = str(e)
        
        return {
            'statusCode': 200,
            'body': json.dumps({"Error": error_message}) 
        }
```


## Add AlpineJS and Fetch a Response
After the API endpoint was set up, it was time to update the webpage to make a call to the endpoint and display its response. There are many ways to handle API calls with Javascript, vanilla, Vue, React, just to name a few. The author decided on a Javascript framework called [AlpineJS](https://alpinejs.dev/) for its relative simplicity. Adding AlpineJS in a script tag in the `<head>` of the page we were able to access AlpineJS directives, which is where all of the Javascript will be placed. Variables and functions were set within the `x-data` directive attached to the `<main>` element of the page. Those variables include:

- a Boolean variable to show the `<div>` element that will contain the API response
	- Additional directives were added to this element, namely `x-show`
- two variables that will hold our data (resume, jobDesc)
	- these variables were modeled with the `<textarea>` to sync their values with `x-model`
- a reply variable that will hold the response from the API
	- the `x-text` directive was added to the `<div>` that will display the response
- finally an `async` function that will make call to the API `getData(resume,jobDesc)` being sure to pass in the necessary data from the form
	- a `@click` handler was added to the button on the form to call the function so that the data can be submitted to the API

The `getData()` function is shown below and then again in context of the whole page. This function uses the Javascript `fetch()` API to make a request to the API endpoint. Awaiting the response it assigns its body to our `reply` variable, allowing it to be displayed on the page. Additionally the `showResponse` variable is set to true. The `x-show` directive from AlpineJS allows one to show or display an element based on a condition.

```javascript
async getData(resume, jobDesc) {
			const response = await fetch('https://your-aws-amazon-api-endpoint', {
				method: 'POST',
				body: JSON.stringify({
					'resume': resume,
					'jobDesc': jobDesc
				})
			})
			const data = await response.json()
			this.showResponse = true
			this.reply = data['body']
			
		}
```

Full page example:

```html
<!DOCTYPE html>
<html>

<head>
	<meta charset="UTF-8">
	<meta name="viewport" content="width=device-width, initial-scale=1.0">
	<title>Job Fit Analysis</title>
	<!-- Added Line -->
	<script defer 			src="https://cdn.jsdelivr.net/npm/alpinejs@3.x.x/dist/cdn.min.js"></script>
	<link href="output.css" rel="stylesheet">
</head>

<body>
	<!-- Added x-data to <main>, x-model to <textarea> -->
	<!-- added @click to <button>, x-show, x-text to <div -->
	<main x-data="{
		showResponse: false,
		resume: '',
		jobDesc: '',
		reply: '',
		async getData(resume, jobDesc) {
			const response = await fetch('https://your-aws-amazon-api-endpoint', {
				method: 'POST',
				body: JSON.stringify({
					'resume': resume,
					'jobDesc': jobDesc
				})
			})
			const data = await response.json()
			this.showResponse = true
			this.reply = data['body']
			
		}
		
	}" >
			<h1>
				Job fit analysis
			</h1>
			<form>   <label for="resume">Enter your resume:</label>
				<br>   
				<textarea x-model="resume" id="resume" name="resume" rows="10" cols="50"></textarea>   
				<br><br>   
				<label for="job-desc">Enter the job description:</label>
				<br>    
				<textarea x-model="jobDesc" id="job-desc" name="job-desc" rows="10" cols="50"></textarea>      
				<br><br>   
				<button @click="getData(resume,jobDesc)" type="button" >Process</button> 
			</form>
			<div x-show="showResponse" x-text="reply"></div>
	</main>
</body>
</html>
```

## Deploy Website
The final step of the project was to make the website available to the public. There are many options to host a static website, but since all of the other services were created with AWS it was decided to serve the site from an S3 bucket. Here is a tutorial on how to serve a static site from [AWS S3 bucket](https://docs.aws.amazon.com/AmazonS3/latest/userguide/WebsiteHosting.html).

## Summary
Utilizing many of the services that AWS has to offer the author was able to create a website that accepts data from a user (resume, and job description) and posts it to an API endpoint for OpenAI to process. A response is returned and displayed on the page. Although the set-up described is complicated, it is a good exercise in using these AWS services. The author learned quite a lot about these services.

## Challenges
As with any new endeavor it was not all smooth sailing, and things did not work on the first try. Particular challenges were the concept of [CORS](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS), Lambda Layers, API Gateway Stages. All were more or less addressed by searching on the internet. Relevant documentation was provided where possible.

Finally the below link contains information about CORS while serving a static website from and AWS S3 bucket.

[AWS S3 - CORS documentation](https://docs.aws.amazon.com/AmazonS3/latest/userguide/cors.html)