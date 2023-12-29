Motivation: 

1. **Learning**. The fastest and most effective way to learn anything is by doing it. Big companies don't use Heroku or Netlify to run their online businesses. AWS is a global supercomputer you can leverage at scale to do virtually anything you want. It’s broken down into Lego blocks of functionality that you can snap together in any configuration you want. Learning all the pieces, how to snap them together, take them apart, then snap them together again differently takes practice.

2. **Value**. AWS is a pay-as-you-go service much like water or gas, you pay for what you use. In my experience, it's been worth it - so far only costing me a few dollars a month for top quality services. After using free services from smaller hosting companies (that probably use AWS under the covers), strict limits and lack of flexibility makes AWS a breath of fresh air and worth the cost.

## Architecture

*The following is the Architecture for the new blog*

![Architecture Diagram](https://s3.ap-southeast-2.amazonaws.com/blog.crewsj.net/shared_images/architecture_diagram_01.png "Architecture Diagram")

I want to drive website content via changes to external files, such as JSON and Markdown files. That way, I won't need to deploy React to S3 every time new content is added. This is especially important when working with a CDN like Cloudfront, as content is cached here which needs to be invalidated on every deployment.

For this first iteration I will pull the JSON and markdown using [Github Pages](https://pages.github.com/), here - [https://github.com/jimcrews/crewsj.github.io](https://github.com/jimcrews/crewsj.github.io). (I might implement a database and API down the line if my motivation is there).

## Route 53
Register a new domain or transfering an existing one. Route 53 offers a lot of amazing features such as intelligent routing and health checks capabilities. It also integrates with other AWS services using Alias records. This way, even if the IP addresses of the underlying resources should change, traffic will still be sent to the correct endpoint. I'll be using an Alias record to point to my Cloudfront distribution.

I registered the domain *crewsj.net*


## S3
Create a new bucket, and make it publicly accessible. The bucket name must match your domain name exactly. I created the bucket *blog.crewsj.net* because I intend to create a DNS record and use *blog* for my website.

After creating the new bucket, go into properties and enable "Static Website Hosting".


## React App
Create a website using React. I'll be using vite to create a new React app using the Typescript template.

```
npm create vite@latest react-s3-blog -- --template react-ts
```

Once your happy with your website, run ```npm run build```, and copy the contents of ```dist``` into your S3 bucket.


## Certificate Manager
To use https, we need a certificate. I created a wildcard certificate: *.crewsj.net. You will be required to validate your domain. Choosing DNS validation works seamlessly with Route 53.


## Cloudfront
Create a distribution using the origin of your S3 bucket. Under Viewer Protocol Policy choose Redirect HTTP to HTTPS. Scroll all the way down to the Alternate Domain Names (CNAMEs) field and type in your domain name without http(s), i.e. blog.crewsj.net. Under Settings, choose Custom SSL Certificate for CloudFront SSL encryption.

## S3 revisited
Now that Cloudfront is setup, create a new Alias record to point to your distribution.

