# Bitbucket Webhook CICD Deployment
How to setup CI/CD based off a bitbucket webhook for auto deployment to your LIVE or DEV environments.
This is for auto deploying .NET applications, however this could be easily translated to other languages.


## Requirements
- A domain to create an A record to point towards your server hosting the API
- Ability to host an API to receive the webhook requests


## Server Software Requirements
- IIS or other web server software
- [GIT](https://git-scm.com/downloads/win) must be installed and added to the environment variable. IIS will need restarting `iisreset /restart` after installation
- [.NET](https://dotnet.microsoft.com/en-us/download/dotnet), you will need to install all of the .NET runtimes your deployed apps will use
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
The following examples are written in .NET

### Validate the received payload on the endpoint
```
string secret = "{webhook secret}";

using var reader = new StreamReader(HttpContext.Request.Body);
var payload = await reader.ReadToEndAsync();
// Dont pass the object into the actual API method as then we cant read the body to get the payload, this needs manually deserializing
// You will also need to create your own BitbucketPush class, this can be easily done by logging the payload and pasting it into an online converter
BitbucketPush push = JsonConvert.DeserializeObject<BitbucketPush>(payload);

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

### Check if you have an automated deployment for the repository & branch
```
//Get the repo name and branch
var repo = push.repository.full_name.Replace("{account name}/", ""); //Get the repo name by removing the account name
var branch = push.push.changes.First().@new.name; //branch the commit was pushed to

//Check if we have an automated deployment process for this project & branch
//This example assumes you have a list of objects with the deployment configs
var project = deployments.Where(x => x.Repo == repo && x.Branch == branch).FirstOrDefault();
if (project == null)
{
   //Repo / branch combination doesn't have a deployment process
   return;
}
```

### Call a powershell script
```
// Create the runspace
using var runspace = RunspaceFactory.CreateRunspace();
runspace.Open();

using PowerShell ps = PowerShell.Create(runspace);

//These three lines are only needed if the ExecutionPolicy is set to restricted on the machine
//ps.AddScript("Set-ExecutionPolicy Bypass -Scope Process -Force");
//await ps.InvokeAsync();
//ps.Commands.Clear();
 
ps.AddCommand("path\to\powershell.ps1");

var parameters = new Dictionary<string, object>()
{
   { "ExampleParamter", "test" },
};

// Add the parameters to the script
foreach (var param in parameters)
{
   ps.AddParameter(param.Key, param.Value);
}

// Error logging - any Write-Error inside the powershell scripts will be logged here
ps.Streams.Error.DataAdded += (object sender, DataAddedEventArgs e) =>
{
   string error = ((PSDataCollection<ErrorRecord>)sender)[e.Index].ToString();
};

// Start the script
var results = await ps.InvokeAsync();
runspace.Close();
```

## Auto deploy to IIS Powershell Script Example
```
# Powershell parameters
param (
    [string]$GitBranch, // development
    [string]$IISSiteName, //Test API
    [string]$IISFolder, // C:\IIS\wwwroot\TEST API
    [string]$RepoFolder, // C:\Repos\
    [string]$NetBuildPath, // Test.API\bin\Release\net7.0\publish
)

$ProjectName = [System.IO.Path]::GetFileNameWithoutExtension($GitURL)
$ProjectPath = "$RepoFolder\$ProjectName\"
$GitURL = "https://{username}:{app_password}@bitbucket.org/{user}/$ProjectName.git";

# Step 1 - Delete the existing folder if it exists then clone the repo & specified branch
if (Test-Path $ProjectPath) {
    Remove-Item "$ProjectPath\*" -Recurse -Force
}
git clone $GitURL --branch $GitBranch "$ProjectPath"


# Step 2 - Build the project
dotnet publish $ProjectPath -c Release

# Import the IISAdministration module as it's the newer IIS tool rather than WebAdministration
Import-Module IISAdministration

# Step 3 - Stop the IIS instance
Stop-IISSite -Name "$IISSiteName" -Confirm:$false # Remove the confirmation

# Wait 5 seconds in case of a slow IIS shutdown, this may need increasing if your apps have a slow shutdown
# Rather than doing this, you could implement a while loop using the Get-IISSite -Name "$IISSiteName" to wait until it has stopped
Start-Sleep -Seconds 5 


# Step 4 - Delete all the files in the hosted folder
Remove-Item "$IISFolder\*" -Recurse -Force


# Step 5 - Copy the build files to the hosted folder
Copy-Item -Path "$ProjectPath\$NetBuildPath\*" -Destination $IISFolder -Recurse -Force

# Wait an additional 3 seconds just in case the files are fully copied over
Start-Sleep -Seconds 3


# Step 6 - Start the IIS instance
Start-IISSite -Name "$IISSiteName"

#You can also add another check here to see if the IIS instance has started using Get-IISSite -Name "$IISSiteName"

# Step 7 - Exit script
exit 0
```


## Considerations
- Instead of using the GIT username and app password, it will be more secure to use SSH

## Author
- [SpikeThatMike](https://spikethatmike.dev)

