---
layout: post
title: 'II. Installation'
date: 2021-03-22 21:19:16 -0400
categories: project documentation
---

As described above, the application is nearly entirely self-contained. This design decision was made to meet project requirements, and to greatly simplify the forthcoming section.
An approach involving a ‘microservice’ architecture or deploying the application’s separate components in a ‘serverless’ fashion would have provided a substantially more difficult process to replicate, so the decision was made here to greatly simplify the deployment of the project.

Three major components are required for the complete application’s deployment—the application server (the node process), a/the MongoDB instance, an AWS account (for S3), and, if desired, a
Heroku account (or another VM on which to run the application)--Heroku was chosen here, as it’s the simplest to deploy and quickly associated automated deployments with a GitHub account. For
the MongoDB instance, MongoDB’s own Atlas service provides the easiest option for quickly deploying an instance. Credentials for AWS and MongoDB are stored in an env file. The format is as
follows:

```
MLAB_URL="your-instances-connection-string"
JWT_SECRET="your-secret-here"
NODE_ENV="development"
S3_BUCKET_NAME="your-s3-bucket-name"
AWS_ACCESS_KEY_ID="your-access-key"
AWS_SECRET_ACCESS_KEY="your-secret-access-key"
PUBLIC_S3_URL=”your-s3-buckets-public-url”
PORT=”choose-a-port”
```

This file contains valuable credentials, and must not be committed to a source code repository. It’s wise to add the ‘.env’ file to your ‘.gitignore’ immediately after initialization to avoid any mistakes. Configuration for the Atlas instance is quite straightforward and is best suited to reading their documentation, available here. For aid during development and debugging, MongoDB offers a desktop client that allows for exploration, editing, and deletion of documents—this is quite handy and simple to use. Additionally, the MongoDB VSCode extension greatly simplifies quick viewing of documents. If the port environment variable is not set, the server will run on port 3000.

---

AWS S3 is used for the file-sharing functionality in the application. When a user shares a file via the UI, the data from the file is sent to the application server, which then takes the contents and sends them to designated S3 bucket (handling file uploads via the client directly may be possible; this decision was made to avoid having credentials explicitly on the client—a poor decision for obvious reasons). After a successful upload, a URL is built and returned to the room the client made the request from. For all of the this to be accomplished the following is required: a publicly accessible S3 bucket (so the content can be accessed by any client), an IAM user with read/write permissions to the bucket, and programmatic access to the bucket (so the uploads can take place in a programmatic fashion).

The following screenshots outline the process and correspond to the listed steps.

First, Create a new bucket, and make it publicly accessible. Choose the appropriate region (us-east-1, here), and allow public access.

![AWS 1](/assets/images/aws1.PNG)

Second, allow public access. The most straightforward way to allow public access is to edit the bucket policy, as so, as this allows globally public access to the bucket. The following policy allows public access to each object in the bucket.

![AWS 2](/assets/images/aws2.PNG)

Next, you’ll now need to create a programmatic IAM user with r/w access to the new bucket. First create the user with programmatic access, and grant the permissions to your previously created
bucket.
![AWS 3](/assets/images/aws3.PNG)

![AWS 4](/assets/images/aws4.PNG)

Upon creation, save the CSV file, as it contains the credentials mentioned above, granting programmatic access. With the user created, add these credentials in the corresponding locations to
the environment variable file.

![AWS 5](/assets/images/aws5.PNG)

---

For installation of the application server, clone the repository, install the dependencies,
and start the server. The application has been tested and developed using Node.js versions 12.20.1
and 15.3.0; modern JavaScript features are used throughout, so a minimum version of 12 is suggested.

![NPM 1](/assets/images/npm1.PNG)

With the dependencies installed, either run the server, or start the dev server (using Nodemon, for hot reloading).

![NPM 2](/assets/images/npm2.PNG)

![NPM 3](/assets/images/npm3.PNG)

This same deployment process should be followed if deploying the application on a Linux/Windows instance—change the PORT environment variable to 443 (the application can be run without HTTPS,
but setting samesite secure HTTP-only cookies requires HTTPS). Also, it’s 2021 and getting an SSL certificate is simpler than ever.
