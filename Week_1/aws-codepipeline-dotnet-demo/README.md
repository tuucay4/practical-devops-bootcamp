# .NET Application: Local Execution & Elastic Beanstalk Packaging Guide

This repository contains a .NET web application designed for AWS deployment. This README focuses on the steps required to run the app locally and package it into a plain ZIP file for manual deployment to the **AWS Elastic Beanstalk Management Console** (without using Docker).

---

## 📋 Prerequisites

- **[.NET 8.0 SDK or 9.0 SDK](https://dotnet.microsoft.com)** (ensure your version matches the project target).
- **AWS Account** with permissions to create Elastic Beanstalk environments.
- A terminal (PowerShell, Bash, or Command Prompt).

---

## 🚀 1. Running Locally

Before packaging for the cloud, verify the application works on your machine.

1.  **Clone the repository:**
    ```bash
    git clone https://github.com


    ```
2.  **Restore and Build**

          dotnet restore

3.  **Run the Project**

    dotnet run --project AWSCodePipelineDemo/AWSCodePipelineDemo.csproj

The app will typically start at http://localhost:5000 or https://localhost:5001.
Open your browser to verify the application is active.
Press Ctrl+C to stop the server.
