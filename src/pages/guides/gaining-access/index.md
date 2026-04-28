---
title: Guides - Gaining Access
description: How to gain access to APIs 
---

# Authenticate and access Brand Intelligence APIs

This document provides a step-by-step tutorial in order to make calls to Adobe Brand Intelligence APIs. At the end of this tutorial, you will have generated or collected the following credentials that are required as headers in all Brand Intelligence API calls:

- `{ACCESS_TOKEN}`
- `{API_KEY}`

To maintain the security of your applications and users, all requests to Brand Intelligence APIs must be authenticated and authorized using standards such as OAuth. You can gather most of the required credentials in the initial one-time setup. The access token, however, must be refreshed every 24-hours.

## Prerequisites

In order to successfully make calls to Brand Intelligence APIs, you must have the following:

- An organization with access to Adobe Brand Intelligence.
- An Admin Console administrator that is able to add you as a developer and a user for a product profile.

## Gain developer and user access

Before creating integrations on Adobe Developer Console, your account must have developer in Adobe Admin Console.

See the Admin Console documentation for specific instructions on how to [manage developer access for product profiles](https://helpx.adobe.com/enterprise/admin-guide.html/enterprise/using/manage-developers.ug.html).

Once you are assigned as a developer, you can start creating integrations in [Adobe Developer Console](https://www.adobe.com/go/devs_console_ui). These integrations are a pipeline from external apps and services to Adobe APIs.

# Creating API Credentials

The following page includes details of how you go about setting up an OAuth client that can access the Workfront APIs

## Step 1: Developer Console Access

API setup is done through [Adobe Developer Console](http://developer.adobe.com). 

![image](images/admin-console.png "Console")

## Step 2: Creating an OAuth Client

After logging into the console you will see an option to create a Project. "Projects" are a way to managed different integrations or client apps. 

![image](images/create-project.png "Console")

After you create a Project you will see an option to add an API

![image](images/add-api.png "Console")

Adobe Brand Intelligence API's are associated with a single API named "Adobe Brand Intelligence" found under the "Adobe Firefly Services" category. 

![image](images/select-workfront.png "Console")

Select the OAuth Server-to-Server authentication type.

Next, select the OAuth Server-to-Server authentication type to generate access tokens and access the Experience Platform API. Give your credential a meaningful name in the Credential name text field before selecting Next.

![image](images/oath.png "Console")

If the IMS Org has multiple instances, then select the instance you want the API client to be associated with

![image](images/instance-selector.png "Console")

Once selected, you will be sent back to the Project page where your new Oauth Server-to-Server credential is listed.

![image](images/new-credential.png "Console")

Clicking into the details of that credential you can see important information that includes how you generate the access token.

![image](images/details.png "Console")
