# Bitbucket Webhook CICD Deployment
How to setup CI/CD based off a bitbucket webhook for auto deployment to your LIVE or DEV environments
This is for auto deploying .NET applications, however this could be easily translated to other languages

## Requirements
- A server which can receive external requests
- Ability to host an API to receive the webhook requests

## Server Software Requirements
- IIS or other web server software
- [GIT](https://git-scm.com/downloads/win) must be installed and added to the environment variable and IIS will need restarting iisreset /restart
- [.NET](https://dotnet.microsoft.com/en-us/download/dotnet), you will need to install all of the .NET runtimes you apps will use
- [MSBuild](https://visualstudio.microsoft.com/downloads/#build-tools-for-visual-studio-2022) must be installed and necessary workloads added

## Create your webhook
- Sign into your Bitbucket account
- Go to the repository you want to setup the auto deployment for
- Go to the settings
- Go to Workflow / Webhooks
- Add Webhook
   - Title: CI/CD Deployment
   - URL: {YOUR SERVER URL WHICH WILL RECEIVE THE REQUESTS}
   - Secret: {GENERATE A CUSTOM SECRET OR CHOOSE YOUR OWN}
   - Status: ACTIVE
   - Triggers: PUSH
- Save

## Create an App password
This example will use an username and app password to clone your repository
- Sign into your Bitbucket account
- Click on your profile icon in the bottom left corner.
- Select Personal settings from the menu.
- In the sidebar, click on App passwords.
- Click the Create app password button.
- Enter a name for the app password (e.g., "CI/CD Deployment").
- Select Read permission for repositories, this is the only permission you will need
- Click Create.
- Copy the generated app password and store it in a secure location, you won’t be able to see it again.

## Create the API
The following examples are wrriten in .NET
```
string secret = "{webhook secret}";

using var reader = new StreamReader(HttpContext.Request.Body);
var payload = await reader.ReadToEndAsync();
// Dont pass the object into the actual API method as then we cant read the body to get the payload, this needs manually deserializing
// You will also need to create your own BitbucketRepoPush class, this can be easily done by logging the payload and pasting it into an online converter
BitbucketRepoPush push = JsonConvert.DeserializeObject<BitbucketRepoPush>(payload);

// Get the Bitbucket signature from the request headers
var bitbucketSignature = Request.Headers["X-Hub-Signature"].ToString();
if (string.IsNullOrEmpty(bitbucketSignature))
    return BadRequest("Missing signature");

// As of now Bitbucket is still using SHA-256 for the hash for the signature
using var sha256 = new HMACSHA256(Encoding.UTF8.GetBytes(secret));
var digest = sha256.ComputeHash(Encoding.UTF8.GetBytes(payload));
var calculatedSignature = "sha256=" + BitConverter.ToString(digest).Replace("-", "").ToLower();

 // Compare the calculated signature with the signature provided by bitbucket
if (calculatedSignature != bitbucketSignature)
{
    //Incorrect signature
    return Unauthorized("Invalid signature")
}

//Valid signature
```

## Auto deploy to IIS

## Considerations

## Author
- [SpikeThatMike](https://spikethatmike.dev)

