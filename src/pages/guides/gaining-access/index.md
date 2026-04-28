---
title: Guides - Gaining Access
description: How to gain access to APIs 
---

# Authenticate and access Brand Intelligence APIs

This document provides a step-by-step tutorial for making calls to Adobe Brand Intelligence APIs. At the end of this tutorial, you will have an `{ACCESS_TOKEN}` to authenticate all Brand Intelligence API requests.

Brand Intelligence uses OAuth Server-to-Server credentials issued through Adobe Developer Console. All requests require an IMS Bearer token in the `Authorization` header. Tokens are valid for 24 hours and must be refreshed before expiry.

## Prerequisites

In order to successfully make calls to Brand Intelligence APIs, you must have the following:

- An organization with access to Adobe Brand Intelligence.
- An Admin Console administrator that is able to add you as a developer and a user for a product profile.

## Gain developer and user access

Before creating integrations on Adobe Developer Console, your account must have developer in Adobe Admin Console.

See the Admin Console documentation for specific instructions on how to [manage developer access for product profiles](https://helpx.adobe.com/enterprise/admin-guide.html/enterprise/using/manage-developers.ug.html).

Once you are assigned as a developer, you can start creating integrations in [Adobe Developer Console](https://www.adobe.com/go/devs_console_ui). These integrations are a pipeline from external apps and services to Adobe APIs.

## Creating API Credentials

The following steps walk you through setting up OAuth Server-to-Server credentials for Brand Intelligence.

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

Next, select the OAuth Server-to-Server authentication type to generate access tokens and access the Brand Intelligence API. Give your credential a meaningful name in the Credential name text field before selecting Next.

![image](images/oath.png "Console")

If the IMS Org has multiple instances, then select the instance you want the API client to be associated with

![image](images/instance-selector.png "Console")

Once selected, you will be sent back to the Project page where your new Oauth Server-to-Server credential is listed.

![image](images/new-credential.png "Console")

Clicking into the details of that credential you can see important information that includes how you generate the access token.

![image](images/details.png "Console")


## Using your access token

### Generate a token

With your credentials in hand, request a token from Adobe IMS:

```bash
export CLIENT_ID=<your_client_id>
export CLIENT_SECRET=<your_client_secret>

curl --location 'https://ims-na1.adobelogin.com/ims/token/v3' \
  --header 'Content-Type: application/x-www-form-urlencoded' \
  --data-urlencode 'grant_type=client_credentials' \
  --data-urlencode "client_id=$CLIENT_ID" \
  --data-urlencode "client_secret=$CLIENT_SECRET" \
  --data-urlencode 'scope=openid,AdobeID'
```

**Example response:**

```json
{
  "access_token": "eyJhbGciOiJSUzI1NiIsIng1dSI6...",
  "token_type": "bearer",
  "expires_in": 86399
}
```

### Make authenticated requests

Include the token in every Brand Intelligence API request:

```bash
--header "Authorization: Bearer <your_access_token>"
```

Brand Intelligence resolves your tenant and permissions from the token. There is no separate API key.

### Token expiry and refresh

Each token is valid for 24 hours (`expires_in: 86399` seconds). Implement a mechanism to detect approaching expiry and request a new token before the current one expires to maintain uninterrupted access.

<InlineAlert variant="warning" slots="text"/>

Never commit or log your Client ID, Client Secret, or access tokens. Store credentials securely server-side and keep them out of version control.
