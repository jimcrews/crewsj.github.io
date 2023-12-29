Being proficient in a popular cloud platform is a hot and trending skill to have, and is more often becoming a requirement for new dev roles. So choosing AWS to build personal projects, I can hopefully learn more about the platform as well as whatever I'm using it for. There's a pretty steep learning curve though, so getting started can be difficult. AWS can be a scary place due to the sheer size of it, and the always present danger of accidentally running up a massive bill (as has happened to many and well documented on hackernews).. But hopefully, by setting up Budgets and Alerts, and not ignoring any AWS emails, I'll avoid this. 

On to the project.

## Architecture

*The following is the Architecture for my new blog*

![Architecture Diagram](https://s3.ap-southeast-2.amazonaws.com/blog.crewsj.net/shared_images/architecture_diagram_01.png "Architecture Diagram")

I want to drive website content via changes to external files, such as JSON and Markdown files. That way, I won't need to deploy React to S3 every time new content is added. This is especially important when working with a CDN like Cloudfront, as content is cached there which needs to be invalidated on every deployment.

For this first iteration I will pull the JSON and markdown using [Github Pages](https://pages.github.com/). JSON will describe each post, and will look like this:

```json
[
    {
        "id": 1,
        "title": "My First Post",
        "slug": "my-first-post",
        "cover": "https://my-image.com/image.png",
        "date": "2023-12-28",
        "markdown": "https://web.github.io/f1.md",
        "tldr": "post description here"
    },
    {
        "id": 2,
        "title": "My Second Post",
        "slug": "my-second-post",
        "cover": "https://my-image.com/image.png",
        "date": "2023-12-30",
        "markdown": "https://web.github.io/f2.md",
        "tldr": "A very big post"
    }
]
```

## Details
Working with React and AWS, The following sections describe each component of the application at a high level.

### Route 53
Register a new domain or transfering an existing one. Route 53 offers a lot of amazing features such as intelligent routing and health checks capabilities. It also integrates with other AWS services using Alias records. This way, even if the IP addresses of the underlying resources should change, traffic will still be sent to the correct endpoint. I'll be using an Alias record to point to my Cloudfront distribution.

I registered the domain *crewsj.net*

### S3
Create a new bucket, and make it publicly accessible. The bucket name must match your domain name exactly. I created the bucket *blog.crewsj.net* because I intend to create a DNS record and use *blog* for my website.

After creating the new bucket, go into properties and enable "Static Website Hosting".


### React App
Create a website using React. I'll be using vite to create a new React app using the Typescript template.

```
npm create vite@latest react-s3-blog -- --template react-ts
```

Once your happy with your website, run ```npm run build```, and copy the contents of ```dist``` into your S3 bucket.


### Certificate Manager
To use https, we need a certificate. I created a wildcard certificate: *.crewsj.net. You will be required to validate your domain. Choosing DNS validation works seamlessly with Route 53.


### Cloudfront
Create a distribution using the origin of your S3 bucket. Under Viewer Protocol Policy choose Redirect HTTP to HTTPS. Scroll all the way down to the Alternate Domain Names (CNAMEs) field and type in your domain name without http(s), i.e. blog.crewsj.net. Under Settings, choose Custom SSL Certificate for CloudFront SSL encryption.

### S3 revisited
Now that Cloudfront is setup, create a new Alias record to point to your distribution.

## Conclusion
That's it for now. Future improvements could include:
 - CI/CD using Github Actions for S3 deployment
 - Include a Database and API for web content delivery