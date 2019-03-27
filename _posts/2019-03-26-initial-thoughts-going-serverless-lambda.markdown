---
layout: post
title:  "Initial Thoughts on Going Serverless with AWS Lambda"
date:   2019-03-26 17:40:56
categories: lambdas
---

#### FaaS and AWS Lambda

The growing availability of flexible "rental contracts" of cloud computing resources has made it extremely cost-effective to host low volume applications without any downtime. For a simple application that's only used by a few thousand people per day or so, you could probably get away by serving it from an AWS t2.nano EC2 instance for just [$4.75/month](https://aws.amazon.com/about-aws/whats-new/2015/12/introducing-t2-nano-the-smallest-lowest-cost-amazon-ec2-instance/). And to combat those sporadic jumps in volume you're bound to see from time to time, the major cloud vendors offer a myriad of options to temporarily upgrade your computational resources for a fee.

For users who desire even more flexibility, the next logical step for the cloud service providers was to offer some sort of "pay per utilization" service, where customers no longer have to pay anything for idle servers. Google, Microsoft, and Amazon all built proprietary platforms for this, which are now commonly referred to as FaaS (Functions as a Service). The one we'll focus on in this article is Amazon's Lambda FaaS, which requires other components of the AWS ecosystem to function properly.

What Lambda does is that it allows you to arbitrarily invoke a piece of code based on customizable triggers that are emitted from the other AWS services. That "piece of code" can be written in an ever-expanding list of languages, including dynamically compiled ones (JavaScript, Python) and compiled favorites Java and C#. 

While Amazon doesn't publicize the internals of its FaaS, documentation implies that these functions are used in ephemeral "instances" that should be treated as stateless functions. In essence, a Lambda function will perform its knowledge in isolation without any knowledge of other Lambdas. This should mean that persistence and caching are impossible in Lambdas, but I've seen some [outside speculation](https://medium.com/@tjholowaychuk/aws-lambda-lifecycle-and-in-memory-caching-c9cd0844e072) and evidence from my own experimentation that suggests that Lambda instances aren't actually stateless.

In the context of moving an application from a dedicated server to a "serverless" network of AWS services and configurations, the standard practice seems to be to use the triggers from API Gateway (another AWS service) to invoke Lambda functions after receiving HTTP requests. These functions can essentially mock the "Controller" functionality of a traditional [MVC](https://en.wikipedia.org/wiki/Model%E2%80%93view%E2%80%93controller) architecture.

#### Cost is shockingly ambiguous

A t2.nano is only $0.0065 per hour ($0.000001806 per second).

Lambda costs $0.0000002 per request, and $0.00001667 for every Gigabyte-second. But this is only after the first million requests and 400,000 Gigabyte-seconds each month.

For low volume users in the AWS US East region (<1,000,000 requests per month and <111 Gigabyte-hours), Lambda is theoretically free. The most expensive service would actually be the API Gateway at [$3.50/million](https://aws.amazon.com/api-gateway/pricing/) requests and potential costs such as file storage for static files. For someone running a simple, low-traffic website off this infrastructure I'd imagine that the hosting cost could be lower than a dollar each month. This is, of course lower than the $4.95/month for a t2.nano.

Predicting the cost of either service gets astronomically more difficult once you start scaling things up. Lambda comes with scaling out of the box, but EC2 scaling depends on a ton of factors beyond just CPU/memory stats and user counts. In a real-world scenario, cost comparison could probably only be performed through empiricial experimentation.

#### Performance Matters

While the cost considerations might clearly favor Lambda in some scenarios, performance is also crucial even for low-volume applications. As opposed to an EC2 instance which can expose your application directly to the rest of the internet, a call to a Lambda has to travel through the API Gateway, and through all the other AWS services you may be using in your business logic.

A simple use case: serve the customer with a static html page...

```html
<!doctype html>

<html lang="en">
<head>
  <meta charset="utf-8">

  <title>Lambda Test</title>
  <meta name="description" content="Lambda Test">
  <meta name="author" content="Brian">

</head>

<body>
  Hello World
</body>
</html>
```

I created a Lambda function *serveStatic*, a corresponding API Gateway construct to intercept HTTP requests to trigger *serveStatic*, and a bucket to host that HTML as a file named *index.html*

Inside the lambda function, I wrote the following Python script to return the contents of that HTML file to the API Gateway as part of a JSON object.

```python
import json
import boto3

def lambda_handler(event, context):
    result = None
    try:
        result = boto3.resource('s3').Object('my_bucket_name_here', 'index.html').get()['Body'].read().decode("utf-8")
    except botocore.exceptions.ClientError as e:
        if e.response['Error']['Code'] == "404":
            print("The object does not exist.")
            result = 'Fail'
        else:
            raise
    return {
        'statusCode': 200,
        'body': result,
        "headers": {
            'Content-Type': 'text/html',
        }
    }

```

API Gateway provided me a URI exposed to HTTP requests. After sending a default GET request through my Google Chrome browser at that URI, I was able to see the "Hello World" rendered properly in the browser.

So what I had was a very simple website hosted through a combination of API Gateway, Lambda, and S3. The very basics of an old-fashioned website! But what about the performance?

```
ab -n 100 -c 10 https://{my unique identifier here}.execute-api.us-east-2.amazonaws.com/default/serveStatic
```

I used Apache Bench to test the performance of a hundred subsequent retrievals of that html file and got the following data:

```
Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:      201  349 145.8    312    1394
Processing:   141  211  56.8    199     387
Waiting:       60  106  40.4     94     288
Total:        350  560 164.0    522    1721
```

Now let's compare it to a generic https URL:

```
ab -n 100 -c 10 https://amazon.com/
```

this yielded:

```
Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:      210  265  38.5    259     404
Processing:    54   72  20.4     62     146
Waiting:       54   70  19.1     62     145
Total:        282  336  40.3    328     487
```

I did confirm that the results from amazon.com were similar to a number of other websites I frequent.

So to interpret these results, we have to not only look at the total time it took to finish requests but at the individual components contributing to it.

```
Connect and Waiting times:

The amount of time it took to establish the connection and get the first bits of a response

Processing time:

The server response time—i.e., the time it took for the server to process the request and send a reply

Total time:

The sum of the Connect and Processing times

[src](https://stackoverflow.com/questions/2820306/definition-of-connect-processing-waiting-in-apache-bench)
```

While it would be a lot better to compare the data from calling the Lambda function to that of a server sending the exact same HTML file, we can at least see in this scenario that the serverless architecture (Lambda) had to spend a lot more time processing the request before sending back a reply. 

Also, the standard deviation in total response time for the serverless architecture was over four times larger than that of amazon.com. There was also a request that took 1721 seconds, while the longest request to amazon.com was 487 milliseconds. For a business trying to offer a service, 1.7 seconds could mean anything from a subpar customer experience to a lost sale.

So not only is Lambda going to be slower, but the performance speed will probably be *much* less predictable than that of a dedicated server. However, it can be extremely cost-effective and it requires no configuration from the customer to be scalable.

#### Takeaway

While the cost efficiency of Lambda (and FaaS in general) is very attractive, I'm not sold on the serverless movement yet. Lambda does have many use cases (especially for heavy users of the AWS ecosystem), but doesn't seem solid enough to replace a true server for any type of web service that interacts directly with the people using it.

Questions? Comments? Please [let me know](mailto:brian.l.wilson@protonmail.com). Sayonara!
