# Week 2 — Deploy a Node.js App on AWS Elastic Beanstalk
Deploy a Node.js/Express web application (DumbBudget) on AWS Elastic Beanstalk
**DumbBudget** — a simple personal budget tracking app

## What I Did
### 1. Forked & Cloned the Repository

Forked the source code from https://github.com/DumbWareio/DumbBudget

### 2. Packaged the Application
Unlike the previous week .NET Node.js doesn't need compilation. I Zipped the source code excluding node_modules because EB installs 
dependencies itself

zip -r ../dumbBudgetApp.zip . -x "*.git*" -x "node_modules/*"

### 3. Deployed to Elastic Beanstalk
- Platform: Node.js 20 on Amazon Linux 2023
- Uploaded dumbBudgetApp.zip via AWS Console



