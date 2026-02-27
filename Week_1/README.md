# Week 1 — Deploy a Scalable App on Elastic Beanstalk

Deploy ASP.NET Core Web API on AWS Elastic Beanstalk.


## What I Did

### 1. Forked the Repository
Forked the source code from https://github.com/William-eng/aws-codepipeline-dotnet-demo

### 2. Packaged the Application
I compiled and published the app using

dotnet publish -c Release -o ./publish

Which produces compiled files ready to run on a server.

### 3. Zipped the Output
zip -r ../deploy.zip .

The zip contains the compiled app.

### 4. Deployed to Elastic Beanstalk
- Platform: .NET Core on Linux (.NET 8)
- Uploaded the .NET application packaged-dotnetApp.zip via the AWS Console
