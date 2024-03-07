---
title: .Net
date: 2022-03-23
publishdate: 2022-03-24
weight: 20
toc: true
imageurl: /assets/img/framework-logos/net-logo.png
menu:
  main:
    weight: 20
---

To integrate .Net web services with API Toolkit, an SDK called the `ApiToolkit.Net` client for API Toolkit is utilized. It keeps track of incoming traffic, aggregates the requests, and then delivers them to the apitoolkit servers.
We'll concentrate on providing a step-by-step instruction for integrating an API toolkit into our Golang web service in this tutorial.

## Design decisions:

- The SDK relies on google cloud pubsub over grpc behind the scenes, to ensure that your traffic is communicated to APIToolkit for processing in the most efficient ways.
- Processing the live traffic in this way, allows :
  1. APIToolkit to perform all kinds of analysis and anomaly detection and monitoring on your APIs in real time.
  2. Users to explore their API live, via the api log explorer.

## How to Integrate with a DotNet Web Service:

1. Sign up / Sign in to the [API dashboard](https://app.apitoolkit.io)
   ![Sign up / Sign in](/signin.png)
2. [Create a project](/docs/dashboard/creating-a-project/)
3. [Generate an API key for your project](/docs/dashboard/generating-api-keys), and include a brief description of your work. And to prevent losing your key after it has been generated, remember to make a copy of it.
   ![API key generation](/api-keys-generation.png)
4. Initialize the middleware with the APItoolkit API key you generated above. Integrating only takes 3 lines of Go code:

## Installation

Run the following command to install the package into your .NET application:

```sh
dotnet add package ApiToolkit.Net
```

Now you can initialize APIToolkit in your application's entry point (eg Program.cs)

```csharp
var config = new Config
{
    Debug = true, # Set debug flags to false in production
    ApiKey = "{Your_APIKey}"
};
var client = await APIToolkit.NewClientAsync(config);

# Register the middleware to use the initialized client
app.Use(async (context, next) =>
{
    var apiToolkit = new APIToolkit(next, client);
    await apiToolkit.InvokeAsync(context);
});
```

The field `{Your_APIKey}` should be replaced with the api key which you generated from the apitoolkit dashboard.
In practice, you would set this field using

## Redacting/Masking fields

If you have fields which are too sensitive and should not be sent to APIToolkit servers, you can mark those fields to be redacted either via the APIToolkit dashboard, or via this client SDK. Redacting fields via the SDK means that those fields never leave your servers in the first place, compared to redacting it via the APIToolkit dashboard, which would redact the fields on the edge before further processing. But then the data still needs to be transported from your servers before they are redacted.

To mark a field for redacting via this SDK, you simply need to provide additional arguments to the APIToolkitService with the paths to the fields that should be redacted. There are 3 potential arguments which you can provide to configure what gets redacted.

- `RedactHeaders`: A list of HTTP header keys which should be redacted before data is sent out. eg COOKIE(redacted by default), CONTENT-TYPE, etc
- `RedactRequestBody`: A list of JSONpaths which will be redacted from the request body, if the request body is a valid json.
- `RedactResponseBody`: A list of JSONpaths which will be redacted from the response body, if the response body is a valid json.

Examples of valid jsonpaths would be:

`$.store.book`: Will replace the books field inside the store object with the string [CLIENT_REDACTED]
`$.store.books[*].author`: Will redact the author field in all the objects in the books list, inside the store object.

For more examples and introduction to json path, please take a look at: [https://support.smartbear.com/alertsite/docs/monitors/api/endpoint/jsonpath.html](https://support.smartbear.com/alertsite/docs/monitors/api/endpoint/jsonpath.html)

Here's an example of what your configuration in your entry point (Program.cs) would look like with the redacted fields configured:

```csharp
var config = new Config
{
    Debug = true, # Set debug flags to false in production
    ApiKey = "{Your_APIKey}",
    RedactHeaders = new List<string> { "HOST", "CONTENT-TYPE" },
    RedactRequestBody = new List<string> { "$.password", "$.payment.credit_cards[*].cvv", "$.user.addresses[*]" },
    RedactResponseBody = new List<string> { "$.title", "$.store.books[*].author" }
};
var client = await APIToolkit.NewClientAsync(config);

# Register the middleware to use the initialized client
app.Use(async (context, next) =>
{
    var apiToolkit = new APIToolkit(next, client);
    await apiToolkit.InvokeAsync(context);

```

It is important to note that while the `RedactHeaders` config field accepts a list of headers(case insensitive),
the `RedactRequestBody` and `RedactResponseBody` expect a list of JSONPath strings as arguments.

The choice of JSONPath was selected to allow you have great flexibility in describing which fields within your responses are sensitive.
Also note that these list of items to be redacted will be aplied to all endpoint requests and responses on your server.
To learn more about jsonpath to help form your queries, please take a look at this cheatsheet:
[https://lzone.de/cheat-sheet/JSONPath](https://lzone.de/cheat-sheet/JSONPath)

## Next Steps

1. Deploy your application or send test http requests to your service
2. Check API log explorer or Endpoints pages on the APIToolkit dashboard to see if your test request was processed correctly
   ![Endpoint-after-integration](/endpoint-screenshot.png)
3. Enjoy having our API comanage your backends and APIs with you.
