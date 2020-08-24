Unknown markup type 10 {
  type: [33m10[39m,
  start: [33m296[39m,
  end: [33m303[39m,
  href: [32m''[39m,
  title: [32m''[39m,
  rel: [32m''[39m,
  name: [32m''[39m,
  anchorType: [33m0[39m,
  creatorIds: [],
  userId: [32m''[39m
}
Unknown markup type 10 { type: [33m10[39m, start: [33m7[39m, end: [33m14[39m }

# Azure DevOps API Backup

Backup your projects with 99 lines of code

![Visual Studio](https://cdn-images-1.medium.com/max/7562/1*POyCLzcQi1idbHs4AELZzQ.png)*Visual Studio*

### Introduction

[Azure DevOps](https://azure.microsoft.com/en-us/services/devops/) — formerly Visual Studio Team Services — is a cloud service to manage source code and collaborate between development teams. It integrates perfectly with both Visual Studio and Visual Studio Code. While the code is perfectly safe on Azure infrastructure, there are cases where a centralized local backup of all projects and repositories is needed. These might include Corporate Policies, Disaster Recovery and Business Continuity Plans.

![Download as Zip](https://cdn-images-1.medium.com/max/5290/1*FkDrmeSD2LmCEUeAGPHFAQ.png)*Download as Zip*

As the image shows, we can download a repository from Azure DevOps as a Zip file, but this may not be practical if we have a considerable amount of projects and repositories and need them backed up on a regular basis. To do this, we can use the [Azure DevOps API](https://docs.microsoft.com/en-us/rest/api/azure/devops/?view=azure-devops-rest-5.1), as explained in the next steps.

### 1. Get an Azure DevOps API personal access token

On the top right corner of the Azure DevOps portal we have our account picture. Clicking on it reveals the account menu where we find a *Security* option. Inside *Security*, we have *Personal access tokens*. We click on *New token* to create one. For this example we only need to check the *Read, write & manage *permission for *Code*. When we name the token and click *Create*, we get the token value, but since it won’t be shown again, we must copy and save it elsewhere. We will need this token later, to access the API.

### 2. Clone the repository, configure arguments and dependencies

To get the sample code, we can clone [this GitHub repository](https://github.com/beralves/azure-devops-backup) and open it with [Visual Studio](https://visualstudio.microsoft.com). To test the solution in debug mode, we still have to configure a few arguments in the project properties. These arguments will define the access token — obtained on the previous step — , organization — i.e. the Azure DevOps domain — and the local directory where we want to write the backed up data. There is a fourth optional argument to state if we want to unzip the downloaded files — more on that later. Here’s how the argument list will look like:

    --token "our-auth-token" --organization "our-org"  --outdir "C:\our-directory" --unzip

This solution uses two external libraries we need to reference: RestSharp to facilitate the API calls and Newtonsoft JSON to deserialize the API responses into objects.

### 3. Analyze and run the code

We start by declaring the data structures for the API responses. Then, the main function cycles through three nested levels of hierarchy — project / repository / item — calling these API endpoints and deserializing the results into the corresponding structures. Hence, for each project, we get a list of the repositories it contains and for each repository, we get a list of the items it contains.

When we get to the repository level we don’t need to make individual API calls to download every single item on the repository. Instead, we can download all the items, packed into a Zip file with a single API call. In order to do this, we still need the item list, because we have to build a JSON structure containing every item on the list where the property *gitObjectType* equals “*blob”*. This JSON structure will then be posted to the */blobs* endpoint to obtain the Zip file as a response.

Note we are also saving the original JSON item list we got from the repository call. This will be needed to map the files inside the Zip package, because these are presented in a single flat directory and their names are the object ids and not their actual names and extension. This is where the --unzip argument enters. If it’s omitted, the process does not go further and we get a simple backup: for every repository of each project we get a Zip file with the content and a JSON file to map it.

If the --unzip argument is present, the program will create a directory for each repository based on the information provided by each Zip/JSON file pair. In this directory, we will get the original file and folder structure with real file names and extensions. Looping through all the items on the JSON list file, we consider a simple condition: if the item is a folder we create the directory according to the *item.path* property. Otherwise, we assume it’s a blob and we extract it from the Zip archive into the corresponding directory assigning the original file name and extension.

### Final thoughts

This is not an exhaustive method to retrieve every artifact on Azure DevOps. There’s a lot more to be done to make this a complete solution. As it is, it will only retrieve the files from the last commit of the default — usually Master — branch. However, it’s a good starting point to backup Azure DevOps projects and keep a local repository of these.

![](https://cdn-images-1.medium.com/max/2000/0*Piks8Tu6xUYpF4DU)

**Follow us on [Twitter](https://twitter.com/joinfaun) **🐦** and [Facebook](https://www.facebook.com/faun.dev/) **👥** and join our [Facebook Group](https://www.facebook.com/groups/364904580892967/) **💬**.**

**To join our community Slack **🗣️ **and read our weekly Faun topics **🗞️,** click here⬇**

![](https://cdn-images-1.medium.com/max/3200/0*oSdFkACJxs5iy1oR)

### If this post was helpful, please click the clap 👏 button below a few times to show your support for the author! ⬇
