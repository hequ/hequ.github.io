---
title: Deploy static website blog to aws
date: 2021-03-13 00:00:00 +0200
categories: azure typescript datacollector-api
layout: post
---

If you plan to create a website, you have a ton of options which technology to use or to which service to deploy. There are some affordable and accessible options like Netlify or Surge if you want to deploy your site with ease. Or you can always go with a WordPress page if you want to have a ready-made solution. I first thought to go with Netlify as Surge does not offer SSL for custom domains on a free plan. However, I decided that I wanted to set up things myself and maybe learn a bit or two about domains and SSL, so I wrapped the sleeves and started playing with the AWS.

I’ve used AWS a lot before, so I was not afraid to look for options there. I searched google for deploying the static website and found that AWS supports this nicely. In short, the idea is to put the static site to S3, set up the CloudFront to cache the contents, and then set up the custom domain name.

This article shows how to create a scalable infrastructure for your site, using S3 and CloudFront. I also tell you how to get a free SSL certificate for your custom domain With the help of AWS.

## Preparing your local environment

To follow this guide, you need to have AWS CLI installed on your system. I’m using a Mac, so getting AWS CLI is as easy as running homebrew. It should also create a credentials file to your home directory. Inside this credentials file, are your AWS credentials, which you use with AWC CLI. If you are running some other operating system, you need to check instructions here on how to install AWS CLI.

```s
brew install awccli should create ~/.aws/credentials file
```

Create a programmatic AWS user
You need to have a programmatic user to access AWS services through your CLI tools. So to create one, you need to login to AWS console and go to IAM. You create a new user and select the ‘programmatic access’ option to allow the user to access AWS through APIs only. It’s always good to create a separate user that you use only with AWC CLI and keep your primary user account separate from this programmatic user account.

Once you have created the programmatic user, you need to save the credentials to ~/.aws/credentials like this:

```conf
[your-profile-name]
aws_access_key_id=client-id
aws_access_key_access_key=client-secret
```

## Create S3 Bucket

You need an S3 bucket for your website, and if you don’t have one, go to AWS Console and select S3. There you can create a new bucket. When creating the bucket, unselect the box, which makes your bucket private. Because your website is public, you need to have the bucket contents visible to the internet. Because the bucket is open, take extra care not to deploy any sensitive files into this bucket!

Turn off the private bucket, but remember not to deploy anything sensitive!

When you have created the bucket, you need to tweak its properties to use it as a website host. First, select the static website hosting option from the properties tab, and enter your website’s entry file. This box also shows the actual endpoint, which you need to use to access your site! Be aware that this endpoint is different to your S3 bucket URL. When you access your website in S3, you are using different URL than when you are accessing your files inside the S3. A bit confusing, and I’m not sure what’s the logic behind this, but this is how it works.

Static website hosting screen contains the endpoint to your website

By default, your S3 bucket contents are not visible to anyone. To fix this, we need to add read permissions for all users of the bucket. Select a Bucket Policy tab and paste the following snippet to the editor.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PublicReadGetObject",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::your-s3-bucket-name/*"
    }
  ]
}
```

## Upload your website to S3 Bucket

Now the bucket is ready to serve our website. We need to upload the actual contents into it. You can use the following index.htmlsnippet as a website if you don’t have one ready yet.

```html
<html>
  <head>
    <title>S3 Test Page</title>
  </head>
  <body>
    <h1>Hello world from S3</h1>
  </body>
</html>
```

Now, let’s upload that snippet into our newly created bucket:

```s
aws s3 --profile your-aws-profile sync index.html s3://your-bucket-name
```

If your bucket name is the-bucket and your S3 bucket is in eu-north-1 region, your website should now be running at http://the-bucket.s3-website.eu-north-1.amazonaws.com. You can check the correct URL from Properties -> Static Website Hosting setting. Test that your site is running correctly.

## Create your domain and adjust its settings

I manage my domains in NameCheap, but if you want to stay in AWS land, you can use Route53 to manage your custom domains. When you have registered your domain name, you need to create a CNAME record that points to your newly created CloudFront distribution. Launch your hosting provider console and create the following records:

Create a CNAME record with @ as host and CloudFront URL as the value. This will map your domain to cloudfront.

## Create an SSL certificate

We can use AWS certificate manager to obtain a certificate. This is useful because if we let AWS create the certificate for us, then AWS also updates the certificate automatically so we don’t have to remember to do it.

To create a certificate, open the certificate manager and click “Request a certificate”. Select public certificate and add your domainname when asked. For the validation method, select DNS validation. DNS validation is preferred, because certificate manager can renew your certificate automatically as long as the DNS entry is in place. If you select email validation, you are being sent an email when you need to renew the certificate.

When you have requested the certificate, you need to go to your domain providers console and add the provided CNAME to your domain. Note that you need to remove your domain name from the certificate manager provided host value. So if certificate manager tells you to add a CNAME of: xyz.yourdomain.com with a value “abcd” then you go to your domain provider console and add a CNAME for “xyz” with a value “abcd”. If you add the whole domain, then the DNS validation will not work.

## Setup CloudFront CDN

The modern web is all about speed. You want your content to be as fast and accessible as possible. AWS provides one of the best CDN’s in the market, and we are going to use it to make our new website fast. If you are not yet familiar with what CDN is, here’s a small primer. CDN stands for Content Delivery Network, and it is a service that saves your content around the world, so no matter where you live, your site is always available to you from a server nearby. In the heart of CDN is its ability to save your content and serve it to clients without hitting your website. This functionality is called caching. CDN can also do other neat tricks like compressing your site contents, so delivering your site is even faster.

To use CloudFront, you select it from the AWS console and create a new distribution. Distribution is the logical unit inside CloudFront in which you describe how your content is going to handled. Now, create a new Web distribution. You need to set the following settings:

The origin domain name needs to be the URL of your website. When you click the field, CloudFront suggests your S3 bucket automatically but don’t select it. Instead, type in your website URL.
For viewer protocol policy, select redirect HTTP to HTTPS as it forces clients to use HTTPS, which is safer.
Enable the Compress Objects Automatically to make CloudFront compress all the contents before transferring. Compressing will make the CDN even faster by minimizing the transferred file size.
For the price class, select what makes the most sense when you think about your users. If you need to cover every country, select the highest price class. But if your visitors live mostly e.g., in Europe or the US, you manage just fine with the cheapest PriceClass.
Add your domain to the Alias field and select the certificate you created earlier in the custom certificate field. This enables CloudFront to use your custom domain.
Hit save and wait when the distribution is created. This might take some time as your site is now spread to all the edge nodes around the world. However, your website can usually be used before the creation is fully completed. Your custom domain should be now set up, so give it a try!

## Future considerations

With the above setup, I was able to achieve the goals I had in mind:

- The site is easy to update with aws-cli. Only one command.
- Costs are low. Essentially this setup is almost free if you are eligible to the AWS free tier. I’ll update the real cost later, but it looks like this setup is free, because, at the moment, I don’t have anything else running in my AWS account.
- I have a custom domain set up with a valid SSL certificate.
- Some future improvements might be creating a CloudFormation template so that the established AWS infrastructure would be described in code and easily repeatable.

I hope you liked this article. If you did, please drop me a message!
