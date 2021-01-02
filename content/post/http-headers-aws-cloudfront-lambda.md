---
title: "Http Headers Aws Cloudfront Lambda"
date: 2019-01-29T20:30:47Z
draft: false
tags:
  - note-to-self
  - how-to
  - AWS
  - CloudFront
  - Lambda
---

All website need appropriate headers setting, even though this is a static site I wanted to start off from a strong security position.

Too often security is an added extra and can't be properly implement because it would break existing functionality.

Amazon have a [good tutorial](https://aws.amazon.com/blogs/networking-and-content-delivery/adding-http-security-headers-using-lambdaedge-and-amazon-cloudfront/)

<!--more-->

My Lambda function is

```js
"use strict";
exports.handler = (event, context, callback) => {
  //Get contents of response
  const response = event.Records[0].cf.response;
  const headers = response.headers;

  //Set new headers
  headers["strict-transport-security"] = [
    {
      key: "Strict-Transport-Security",
      value: "max-age=63072000; includeSubdomains; preload",
    },
  ];
  headers["content-security-policy"] = [
    {
      key: "Content-Security-Policy",
      value:
        "default-src 'none'; img-src 'self'; script-src 'self'  ; style-src 'self'; font-src 'self'; object-src 'none'; frame-ancestors 'none'; base-uri 'self'; form-action 'none';",
    },
  ];
  headers["x-content-type-options"] = [
    { key: "X-Content-Type-Options", value: "nosniff" },
  ];
  headers["x-frame-options"] = [{ key: "X-Frame-Options", value: "DENY" }];
  headers["x-xss-protection"] = [
    { key: "X-XSS-Protection", value: "1; mode=block" },
  ];
  headers["referrer-policy"] = [
    { key: "Referrer-Policy", value: "same-origin" },
  ];

  //Return modified response
  callback(null, response);
};
```

I also set these headers in a local Apache instance to be able to check locally

Apache Conf

```ApacheConf
<IfModule mod_ssl.c>
	<VirtualHost seanburlington:443>
		ServerAdmin webmaster@localhost
		DocumentRoot /home/sean/git/seanburlington/public
<Directory /home/sean/git/seanburlington/public>
        Options Indexes FollowSymLinks
        AllowOverride None
        Require all granted
Header add Content-Security-Policy "default-src 'none'; img-src 'self'; script-src 'self'; style-src 'self'; object-src 'none'; frame-ancestors 'none'; base-uri 'self'; form-action 'none';"
Header add Referrer-Policy "same-origin"
Header add Strict-Transport-Security "max-age=63072000; includeSubdomains; preload"
Header add X-Content-Type-Options "nosniff"
Header add X-Frame-Options "DENY"
Header add X-XSS-Protection "1; mode=block"
</Directory>
		ErrorLog ${APACHE_LOG_DIR}/error.log
		CustomLog ${APACHE_LOG_DIR}/access.log combined
		SSLEngine on
		SSLCertificateFile	/etc/ssl/certs/ssl-cert-snakeoil.pem
		SSLCertificateKeyFile /etc/ssl/private/ssl-cert-snakeoil.key
		<FilesMatch "\.(cgi|shtml|phtml|php)$">
				SSLOptions +StdEnvVars
		</FilesMatch>
		<Directory /usr/lib/cgi-bin>
				SSLOptions +StdEnvVars
		</Directory>
	</VirtualHost>
</IfModule>

```

And in case the tutorial doesn't exist at some point in the future I've made a copy here.

<pre class="hidemost">


Adding HTTP Security Headers Using Lambda@Edge and Amazon CloudFront
by Jake Wells | on 05 DEC 2017 

Whenever you navigate to a website, your browser requests a web page, and the server responds with the content along with HTTP headers. Headers such as cache-control are used by the browser to determine how long to cache content for, others such as content-type are used to indicate the media type of a resource and therefore how to interpret such resource. In this post, you will learn how to add response headers that are specifically targeted to improve the security and privacy of both viewers and content providers. I’ll also show you how you can add these headers to your website using Lambda@Edge and Amazon CloudFront.

Adding security response headers is often achievable by modifications to your application configuration. In this blog we will focus on how to achieve the same result when you have an application that can’t be modified at the origin (e.g., a web site hosted in Amazon S3).

What are security headers?

Security headers are a group of headers in the HTTP response from a server that tell your browser how to behave when handling your site’s content. For example, X-XSS-Protection is a header that Internet Explorer and Chrome respect to stop pages loading when they detect cross-site scripting (XSS) attacks. The following is a list of each header we’ll be implementing with a link to more information.

Strict Transport Security

Content-Security-Policy

X-Content-Type-Options

X-Frame-Options

X-XSS-Protection

Referrer-Policy

Additional details on each of these security headers can be found in Mozilla’s Web Security Guide.

Lambda@Edge Overview

Lambda@Edge provides the ability to execute a Lambda function at an Amazon CloudFront Edge Location. This capability enables intelligent processing of HTTP requests at locations that are close (for the purposes of latency) to your customers. To get started, you simply upload your code (Lambda function written in Node.js) and pick one of the CloudFront behaviors associated with your distribution.

You can run a Lambda@Edge function in response to four different CloudFront events. For the purpose of this blog post, we’ll just be focusing on the Origin Response event. If you’re interested in looking more broadly at Lambda@Edge, Jeff Barr does a great job of providing an overview in his blog post here.

Origin Response – This event is triggered after the origin returns a response to a request. It has access to the response from the origin.

The following diagram illustrates the available triggers for a CloudFront distribution We’re focusing on number 6.:



Here is how the process works:

Viewer navigates to website.
Before CloudFront serves content from the cache it will trigger any Lambda function associated with the Viewer Request trigger for that behavior.
CloudFront serves content from the cache if available, otherwise it goes to step 4.
Only after CloudFront cache ‘Miss’, Origin Request trigger is fired for that behavior.
S3 Origin returns content.
After content is returned from S3 but before being cached in CloudFront, Origin Response trigger is fired.
After content is cached in CloudFront, Viewer Response trigger is fired and is the final step before viewer receives content.
Viewer receives content.
Solution overview

The solution uses a simple single page website, hosted in an Amazon S3 bucket and using Amazon CloudFront. I’ll show you how to create a new Lambda@Edge function, how to associate it with your CloudFront distribution, and how to monitor its execution with Amazon CloudWatch Logs. I will be making use of the origin response trigger to execute our Lambda@Edge function. This will allow CloudFront to cache this response after the security headers are added, which means the Lambda@Edge function will only need to be triggered upon a CloudFront ‘Miss’ and security headers will be returned for all future ‘Hits.’

I will assume that you already have an S3 bucket configured for your website, with a CloudFront distribution configured for serving your content. For the purpose of my demo, I’ve set up an S3 bucket, used it as an origin for my distribution, and uploaded a basic index.html file with the text “Hello World! Do I have security headers yet?”

The following diagram illustrates the sequence of events for triggering our Lambda@Edge function:



Here is how the process works:

Viewer requests website www.example.com.
If the object is cached already, CloudFront returns the object from the cache to the viewer, otherwise it moves on to step 3.
CloudFront requests the object from the origin, in this case an S3 bucket.
S3 returns the object, which in turn causes CloudFront to trigger the origin response event.
Our Add Security Headers Lambda function triggers, and the resulting output is cached and served by CloudFront.
Before I start, I use Mozilla Observatory to rate my website, the following screenshot shows the rating without the security headers. This doesn’t necessarily mean the content being served is insecure, it does however provide a needle on a gauge by which we can look to improve.



Let’s get started. I’ll create a new Lambda function and associate it with my CloudFront distribution. Make sure you have a CloudFront distribution before following the next instructions.

First, I go to the Lambda Console, and I ensure that I’m in the US-East-1 N. Virginia Region by selecting US East (N. Virginia) from the drop-down list at the top right. Then I select Create Function to create a new Lambda function.



Note: I need to select US-East-1 as the location where I create the Lambda function. Otherwise I am unable to connect to a CloudFront trigger. However after I’ve finished with the setup, the function will be replicated to all other Regions.

Next, I am presented with the option to select a blueprint or Author from scratch. If I type in ‘CloudFront’ I am presented with a range of different pre-built functions, but for this solution, I choose Author from scratch because I’ll be using code provided here for this function.



Now that I have started to create a new Lambda function, I need to configure the trigger for it. To do this, I choose the dotted grey box and then choose CloudFront. Note: If you can’t see CloudFront as a trigger option, make sure you’re in the US-East-1 Region, as required in step one. The options presented here are:

Distribution ID: I’m selecting the distribution I created earlier, that serves content from my S3 bucket.

Cache Behavior: I select ‘*’ which is the default behavior. Since in this case I am not creating additional behaviors, this will apply to all requests. If I had created multiple behaviors, this would only be triggered if none of the other behaviors match.

CloudFront Event: As discussed earlier, I want this to trigger after the origin has returned the object but before the cache, so I select Origin Response.

Enable trigger and replicate: I check this box to enable CloudFront as a trigger for a Lambda function. Upon Lambda function creation, this option automatically creates a version of my function and replicates it across multiple Regions.



After I choose Next, I’m presented with the Configure Function page. I choose a meaningful name and description. The Runtime for Lambda@Edge must be Node.js 6.10.

I’m using the following code, which gets the contents of the response, sets the new headers, then returns the updated response that includes the new security headers.

'use strict';
exports.handler = (event, context, callback) => {
    
    //Get contents of response
    const response = event.Records[0].cf.response;
    const headers = response.headers;

//Set new headers 
 headers['strict-transport-security'] = [{key: 'Strict-Transport-Security', value: 'max-age=63072000; includeSubdomains; preload'}]; 
 headers['content-security-policy'] = [{key: 'Content-Security-Policy', value: "default-src 'none'; img-src 'self'; script-src 'self'; style-src 'self'; object-src 'none'"}]; 
 headers['x-content-type-options'] = [{key: 'X-Content-Type-Options', value: 'nosniff'}]; 
 headers['x-frame-options'] = [{key: 'X-Frame-Options', value: 'DENY'}]; 
 headers['x-xss-protection'] = [{key: 'X-XSS-Protection', value: '1; mode=block'}]; 
 headers['referrer-policy'] = [{key: 'Referrer-Policy', value: 'same-origin'}]; 
    
    //Return modified response
    callback(null, response);
};
After pasting this code into my function, I leave my handler as the default‘index.handler and choose to Create a new role from template(s). For the function to execute I need to ensure I select the Basic Edge Lambda permissions from the Policy templates drop-down list, which will go ahead and generate a role for me upon Lambda function creation: 

In the Advanced Settings tab, ensure that 128 MB is the allocated memory and that 3 seconds is configured for timeout (maximum allowed for Lambda@Edge). Then choose Next. Review the details and choose Create Function.

Note: If this was a Viewer Request or Viewer Response trigger, the maximum Timeout would be 1 second.



This creates the function, attaches the trigger to the distribution, and also initiates global replication of the function. The status of my distribution changes to In Progress for the duration of the replication (typically 5 to 8 minutes):



As soon as the status changes back to Deployed I access the root of my distribution, and I see the index.html file:

  

More importantly, to make sure the headers are being added, I open a browser’s Web Developer toolbar, choose the Network tab, and Reload the page. When I choose the GET request for index.html, I’m presented with the added security headers in the response from CloudFront. I’ve highlighted the ones that my function added:



I re-run my website through Mozilla Observatory. The following results show we’ve moved from an F rating to an A+.



Monitoring and debugging

Although I didn’t experience any errors in my execution here, it’s important to know where to go if there are any problems. In the same way that I monitor any Lambda function, I can use Amazon CloudWatch Logs to monitor the execution of Lambda@Edge functions. The slight difference here is that the logs are stored in the Region closest to the location where the function is executed. So, for my test, I need to look at CloudWatch Logs in the London Region because I’m visiting the website from London. I’ll need to change the Region to view the CloudWatch Logs for my Lambda function, according to where my viewers are located.

In this screenshot, I’ve forced an error to show you the log output:



I find it helpful to test my Lambda function directly in the Lambda console before I enable it to be triggered and replicate. That way I save the time it takes to create a new version, assign a trigger, visit the website then view the logs. To do this, I need to configure a test event in the Lambda function in the way I normally would for Lambda, and pass it a sample request or response specific to CloudFront. After I choose Save and Test, I’m presented with the output and any errors. That way I can quickly fix them.



Conclusion

In this post, I showed you how to use Lambda@Edge to improve the security of your website by adding security headers to the origin response trigger of a CloudFront distribution behavior. I demonstrated creating a Lambda@Edge function, associating it with a trigger on a CloudFront distribution, then proving the result and monitoring the output. This is a very simple example of what can be achieved with Lambda@Edge, and I’m sure you can come up with far more creative ways to use it!

If you have any questions about this blog please post them in the Comments section below. If you have any awesome ideas of creative ways you can use Lambda@Edge, please share them in the AWS Lambda Forums.

If you’re new to AWS Lambda@Edge, I encourage you to try out some of the blueprints in the Lambda console and refer to Getting Started with Amazon CloudFront and Getting Started with AWS Lamdba@Edge


 documentation for more information.


</pre>
