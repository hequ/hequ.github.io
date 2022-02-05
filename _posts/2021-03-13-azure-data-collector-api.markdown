---
title: How to use Azure Data Collector Api
date: 2021-03-13 00:00:00 +0200
categories: azure typescript datacollector-api
layout: post
---

This week I’ve been working on a task to send application logs into Azure Monitor. For those, who don’t know, Azure Monitor allows you to easily store logs from different Azure resources such as App Services or virtual machines running in Azure. Those logs are collected into what is called a Log Analytics Workspace. In the case of App Service, the collection of logs is pretty straightforward: you just enable the log collection from the App Service’s settings and that’s it. The logs will be automatically collected for you.

But there is another, what I think is a less known, way to collect logs from your applications. The Log Analytics Data Collector API. This API lets you send in the logs using a REST API which means that you can store logs even from services that are not running in Azure. Here I’ll show you a quick way how to use this API to store your logs, but bear in mind that, at the time of writing, this API is still in public preview, so changes might occur.

To use this API, you need to have a Log Analytics Workspace up and running. You can create one from the portal which is probably the easiest way. When you have access to the workspace, you need to get the WorkspaceID and the Primary or the Secondary SharedKey. These can be found from the portal as you navigate into your Log Analytics workspace, and check the Agents Management page.

You need these credentials to access the workspace. You should probably note that the SharedKeys in the portal is in base64 format, and you need to decode the key into UTF-8 format before you can use it to sign the HMAC signature. This is something that was not very clearly stated in the documentation, and if you don’t pay attention to the key format, it is easy to miss. If you use the key without first decoding it, it will produce an HMAC signature which the Data Collector API won’t accept.

The process to send logs is pretty straightforward, but there are a couple of quirks when creating the authorization header. I hope that the following examples will help you to create a working HMAC signature.

Data Collector API Request

```ts
const url = `https://${workspaceId}.ods.opinsights.azure.com/api/logs?api-version=2016-04-01`;
const response = await fetch(url, {
  method: "POST",
  headers: {
    Authorization: signature,
    "content-type": "application/json",
    "Log-Type": "SomeExternalSystemError",
    "x-ms-date": date,
  },
  body,
});
```

Nothing too hard there. The URL contains the workspaceId and you need to post the HMAC signature of the payload as an Authorization header. You need to also provide the Log-Type which is how you can separate different kinds of logs when you search for logs in Azure Log Analytics. The x-ms-date is the same date used when signing the payload.

Now here’s how you create the HMAC signature in node 14. The example is typescript, but it’s easy to change to plain javascript if needed.

HMAC Signature Creation

```ts
const getAuthorizationSignature = (
  body: string,
  date: string,
  primaryKey: string
) => {
  const stringToSign =
    "POST" +
    "\n" +
    Buffer.from(body).length +
    "\n" +
    "application/json" +
    "\n" +
    "x-ms-date:" +
    date +
    "\n" +
    "/api/logs";

  const hmac = crypto.createHmac("sha256", Buffer.from(primaryKey, "base64"));
  const digest = hmac.update(Buffer.from(stringToSign)).digest("base64");

  return `SharedKey ${workspaceId}:${digest}`;
};
```

The signed string has to contain the length of the HTTP request payload, content-type of the request, the x-ms-date header of the request, and the API path to call. Next, you create an HMAC signature of that payload. I use the node crypto standard library for this. You should notice that the createHmac() function needs the Primary SharedKey in UTF-8 format so I decode the base64 encoded string first using Buffer. Then you can use digest('base64') to create the base64 encoded HMAC signature. I’m returning the complete header value from this function which can be used as-is in the Authorization header.

That’s pretty much it. With these parts, you should be able to build your own function to call the Data Collector API. When you call the API, you should be able to find your message using Log Analytics queries within 30minutes. For the sake of completeness, I put the whole working code in the end so it’s easier to understand.

I hope you find this piece of information useful, and if you do, please let me know in the comments! Happy logging!

Full Working Code

```ts
const getAuthorizationSignature = (
  body: string,
  date: string,
  primaryKey: string
) => {
  const stringToSign =
    "POST" +
    "\n" +
    body.length +
    "\n" +
    "application/json" +
    "\n" +
    "x-ms-date:" +
    date +
    "\n" +
    "/api/logs";

  const hmac = crypto.createHmac("sha256", Buffer.from(primaryKey, "base64"));
  const digest = hmac.update(Buffer.from(stringToSign)).digest("base64");

  return `SharedKey ${workspaceId}:${digest}`;
};

const postLogMessage = async (dataToBeLogged: Record<string, string>) => {
  const workspaceId = "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx";
  const primaryKey =
    "Tm93IHdoZW4geW91IGFjdHVhbGx5IGRlY29kZWQgdGhpcyBtZXNzYWdlLCB5b3UgY2FuIGZvbGxvdyBtZSBpbiB0d2l0dGVyIGF0IGh0dHBzOi8vdHdpdHRlci5jb20va2lubnVuZW5faGVucmkvIDop";

  const date = new Date().toUTCString();
  const body = JSON.stringify(dataToBeLogged);
  const signature = getAuthorizationSignature(body, date, primaryKey);

  const url = `https://${workspaceId}.ods.opinsights.azure.com/api/logs?api-version=2016-04-01`;
  return await fetch(url, {
    method: "POST",
    headers: {
      Authorization: signature,
      "content-type": "application/json",
      "Log-Type": "SomeExternalSystemError",
      "x-ms-date": date,
    },
    body,
  });
};
```
