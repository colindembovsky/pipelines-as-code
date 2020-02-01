# Pipelines as Code
## Intro
I remember my first job after graduating from college – I worked for a small Internet Service Provider (ISP) that provided services such as email and anti-virus to its customers. We were using C++ and CORBA – this was before web services and REST. 
I remember Friday afternoons spent watching the build scripts slowly build our large code base. Woe to the developer that broke the build! When the build succeeded, we would all cheer, and then we’d step outside into the African sun and we would all braai (barbeque) and celebrate another successful build.

I remember how long it took me to wrap my hear around the build process. The scripts were complex, our source control was basic and though most of the process was automated, you still had to have a lot of "tribal knowledge" to get the build to work. However primitive the process was, it was my first taste of automated builds.

My second job was for a financial services company with about 20 developers. There was no such thing as automated build and deploy – we just right-click Published from Visual Studio. As you can imagine, deployment errors and failures were very common: I remember how many times someone overwrote the production web.config with the web.config from their local machine!
I knew there was a better way – and looking around I found a beta product that looked promising. It was called "Team Foundation Server (TFS) 2005". It took about a week to install in those days, but it came with source control, a build engine, work and test management.

## A Trip Down Memory Lane
The build system was a runner that executed MSBuild files – in those days it was a standard way to build .NET applications. But customizing the build process (in XML) was not particularly elegant.

When TFS 2010 shipped, it came with a Windows Workflow Foundation-based build engine. There was a User Interface (UI) editor in Visual Studio for editing the XAML definitions, and I learned how to customize the builds: it was a laborious process, requiring .NET code and uploading binaries to a shared folder under source control, adding that folder to the agent and referencing the assemblies within the custom templates – not many teams bothered to customize because of the difficulty, and those that did ended up spending too much time on "plumbing" to make customization worthwhile.

It’s also worth noting that while my team was (slowly) getting better at automated builds, there wasn’t a good way to deploy automatically. You could add a script in to push compiled code somewhere, but approvals were handled manually or through an external ticketing system. We ended up building our own release management tool.

When Microsoft released TFS 2015, it launched yet another build engine – officially called TeamBuild, but often referred to as Build vNext. This build system had a web-based UI over a JSON file and was an orchestration engine rather than a build engine. You could also create your own custom tasks if you needed to – but adding in a custom script was simple, so most custom logic ended up in simple scripts. Using TeamBuild, teams could quickly create complex workflows for building, testing and deploying their code.
In 2013, Microsoft acquired a tool called InRelease from InCycle. It was rebranded to Release Management and incorporated into TFS 2013. InRelease (and the first wave of Release Management) was a desktop application that used TFS 2013 as the backend system. Now teams could use the desktop designer to model deployment workflows, including approvals – the designer was similar to the XAML designer from XAML builds. 

These releases could be configured to trigger off build completion events, and for the first time teams had a fairly sophisticated toolchain that allowed building, testing and deploying applications. At around the same time, Microsoft release Visual Studio Online, a cloud-hosted version of Team Foundation Server. In 2015, the desktop Release Management product was superseded by an integrated web-based UI. Visual Studio Online was then rebranded to Visual Studio Team Services and later to Azure DevOps Services. Team Foundation Server was rebranded to Azure DevOps Server in late 2018.

For the remainder of this chapter, I’ll refer to Azure DevOps, but this includes Azure DevOps Server too, so whether you’re on the cloud service or hosting your own Azure DevOps Server, there’s no difference as far as pipelines as code is concerned.
While you could export the build and release definitions as JSON files, these artifacts were stored "somewhere" in the backend database of Azure DevOps and not in source control. You could see the history of changes to the definitions – but only if you looked at them in the web designer, and it was difficult to trace changes or even create any sort of approval process for changes to definitions.

Other competing build systems, like CircleCI, allowed teams to create build processes using files stored in source control. Finally, late in 2017 Microsoft released YAML builds. These builds are YAML files stored in source control alongside your application code – which means they benefit from all the source control processes that application code benefit from, such as Pull Requests.
In May 2019, Microsoft added stages to YAML files allow teams to codify the entire end-to-end build, test and deploy workflow in a YAML file. At last, teams using Azure DevOps were able to codify their entire pipeline.

## Why Pipelines?
Before we launch into examining Pipelines, we should know why pipelines as code are important. Pipelines as code:
1. benefit from source control practices (such as Pull Requests)
1. can be audited for changes just like any other files in the repository
1. don’t require accessing a separate system or UI to edit
1. can fully codify the build, test and deploy process for code
1. can be templatized to empower teams to create standard processes across multiple repos

While pipelines as code – in our case, YAML builds – are powerful, they also do have a learning curve and some gotchas. Throughout this chapter I’ll call out some of these gotchas and make some recommendations about how you should think about your YAML builds. 

## Basics of Pipelines
### Agents and Queues
Before we jump into pipelines themselves, we must consider where these pipelines execute. The Azure DevOps build system is still really an orchestration engine: when a build is triggered, it finds an "agent" and tells the agent to execute the jobs (we’ll cover jobs later) defined in the pipeline file.

The agent is written in .NET Core, so it runs happily wherever .NET Core runs – Windows, Mac and Linux. Agents are registered with a queue in Azure DevOps: there are two types of queue in Azure DevOps: hosted and private.

Every Azure DevOps account has a Hosted queue with a single agent that can run one job at a time and an amount of free build minutes. Pricing is beyond the scope of this chapter, but you can purchase additional "hosted pipelines" in Azure DevOps. When you purchase an additional hosted pipeline, you’re really removing the build minutes limit and adding concurrency: one pipeline can run one job at a time, two pipelines can run two jobs simultaneously. When a pipeline is triggered or manually queued, Azure DevOps spins up an agent in Azure (either Windows, Linux or Mac, depending on which image you’ve configured to handle the pipeline) and runs the steps in the pipeline. The image is predefined by Microsoft and comes with a set of tools and SDKs preinstalled. Once the build completes, the instance is deleted.

> **TIP**: To see the list of installed software on the images, navigate to [https://github.com/actions/virtual-environments/tree/master/images](https://github.com/actions/virtual-environments/tree/master/images). Click into the platform folder and examine the README.md files.

There are also private pipelines and agents. If your build requires dependencies that are not installed on the hosted agent you have two choices: add a step that installs the dependency on a hosted agent. If this only take a few seconds, you’re good to go. Remember, each build starts off with a clean instance, so this step will be executed on every build – so long running install processes won’t work here. Your other option is to create a build machine, install your dependencies and then install the Azure DevOps agent on this machine – this is a private agent.

If you need to access resources that are not publicly accessible, such as Artifactory or SonarQube on your corporate network, or you’re deploying to private servers, then you’ll also need to create a private agent.

When you install an agent, you register the agent with a private queue. You can install as many agents onto a queue as you want – but you should only install one agent per core on a build machine. When a pipeline is queued onto a private queue, Azure DevOps will find an idle agent on that queue and offload the job to that agent. However, irrespective of how many agents you have on a queue, you still must pay for concurrency. If you want to run ten jobs at the same time, you’ll need 10 pipelines and at least 10 agents. If you only have one queue and 10 agents, you still only get one job running at a time!

> **TIP**: If you have multiple build machines and want to target specific machines for specific jobs, you can specify demands on the agent and in the pipeline. When the pipeline is queued, Azure DevOps will look for idle agents on the target queue that match the specified demands. More information on demands can be found here: [https://docs.microsoft.com/en-us/azure/devops/pipelines/process/demands?view=azure-devops&tabs=yaml](https://docs.microsoft.com/en-us/azure/devops/pipelines/process/demands?view=azure-devops&tabs=yaml)

## Anatomy of a Pipeline

### Hello World
Let’s start off slowly and create a simple pipeline that echos "Hello World!" to the console. This wouldn’t be a proper technical chapter if it didn’t include a Hello World example. Don’t worry – we’ll get more sophisticated quickly!

```yml
name: 1.0$(Rev:.r)

# simplified trigger (implied branch)
trigger:
- master

# equivalent trigger
# trigger:
#   branches:
#     include:
#     - master
  
variables:
  name: colin
  
pool:
  vmImage: ubuntu-latest
  
jobs:
- job: helloworld
  steps:
  - script: echo "Hello, $(name)"
```

If you run this, you'll see output similar to this:

![Hello World Log](images/hello-world-log.png "Hello World Pipeline log")

While this first example is fairly simple, it shows the anatomy of a basic pipeline. Most pipelines will have these components:

1. Name – though often this is skipped (if it is skipped, a date-based name is generated automatically)
1. Trigger – more on triggers later, but without an explicity trigger, there’s an implicit "trigger on every commit to any path from any branch in this repo"
1. Variables – these are "inline" variables (more on other types of variables later)
1. Job – every pipeline must have at least one job
1. Pool – you configure which pool (queue) the job must run on
1. Checkout – the "checkout: self" tells the job which repository (or repositories if there are multiple checkouts) to checkout for this job
1. Steps – the actual tasks that need to be executed: in this case a "script" task (script is an alias) that can run inline scripts

We’ll cover some other components later, such as:
1. Resources – references to other repos, either for code or for templates
1. Stages – jobs can be grouped together into logical "stages"
1. Templates – reusable jobs or sets of steps
1. Deployments – special jobs that execute against an environment
1. Download – for downloading pipeline artifacts

### Name
The variable name is a bit misleading, since the name is really the build number format. If you do not explicitly set a name format, you’ll get an integer number. This is a monotonically increasing number for run triggered off this pipeline, starting at 1. This number is stored in Azure DevOps. You can make use of this number by referencing `$(Rev)`.

To make a date-based number, you can use the format `$(Date:yyyy-mm-dd-HH-mm)` to get a build number like `2020-01-16-19-22`. To get a semantic number like `1.0.x`, you can use something like `1.0$(Rev:.r)`

> Note how the period between the last two digits is inside the `$()` dereference.

You can also dynamically set the build number if you need it to be calculated as part of the build. This is useful if you use GitFlow or some other mechanism to create build numbers. We’ll cover this scenario later when we explore dynamic variables later on.

### Triggers
As mentioned before, if there is no explicit `triggers` section, then it is implied that any commit to any path in any branch will trigger this pipeline to run. You can be more explicit though using filters such as `branches` and/or `paths`.

Let’s consider this trigger:

```yml
trigger:
  branches: 
    include:
    - master
```

This trigger is configured to queue the pipeline only when there is a commit to the master branch. What about triggering for any branch except master? You guessed it: use `exclude` instead of `include`:

```yml
trigger:
  branches: 
    exclude:
    - master
```

> **TIP**: You can get the name of the branch from the variables `Build.SourceBranch` (for the full name like `refs/heads/master`) or `Build.SourceBranchName` (for the short name like `master`).

What about a trigger for any branch with a name that starts with `topic/` and only if the change is in the `webapp` folder?

```yml
trigger:
  branches: 
    include:
    - feature/*
  paths:
    include:
    - webapp/**
```

Of course, you can mix includes and excludes if you really need to. You can also filter on `tags`.

> **TIP**: Don't forget one overlooked trigger: `none`. If you never want your pipeline to trigger automatically, then you can use `none`. This is useful if you want to create a pipeline that is only manually triggered.

There are other triggers for other events such as:
1. Pull Requests (PRs), which can also filter on branches and paths
1. Schedules, which allow you to specify cron expressions for scheduling pipeline runs
1. Pipelines, which allow you to trigger pipelines when other pipelines complete, allowing pipeline chaining

You can find all the documentation on triggers here: [https://docs.microsoft.com/en-us/azure/devops/pipelines/build/triggers?view=azure-devops&tabs=yaml](https://docs.microsoft.com/en-us/azure/devops/pipelines/build/triggers?view=azure-devops&tabs=yaml)

> **TIP**: If you have a small repo that contains only a single application or component, then you probably don't need branch filters. Branch filters are much more important for mono repos (that is a repo that contains multiple applications or components). If you have all your source in a single repo, then you may still want multiple pipelines that build parts of the repo. This is when you should use path filters.

### Jobs
A job is a set of steps that are excuted by an agent in a queue (or pool). Jobs are atomic – that is, they are executed wholly on a single agent. You can configure the same job to run on multiple agents at the same time, but even in this case the entire set of steps in the job are run on every agent. If you need some steps to run on one agent and some on another, you’ll need two jobs.

A job has the following attributes besides its name:
1. displayName – a friendly name
1. dependsOn - a way to specify dependencies and ordering of multiple jobs
1. condition – a binary expression: if this evaluates to true, the job runs; if false, the job is skipped
1. strategy - used to control how jobs are parallelized
1. continueOnError - to specify if the remainder of the pipeline should continue or not if this job fails
1. pool – the name of the pool (queue) to run this job on
1. workspace - managing the source workspace
1. container - for specifying a container image to execute the job in - more on this later
1. variables – variables scoped to this job
1. steps – the set of steps to execute
1. timeoutInMinutes and cancelTimeoutInMinutes for controlling timeouts
1. services - sidecar services that you can spin up to augment this job, such as spinning up a database container for integration tests

There are two major types of job: `job` and `deployment`. Deployments are a specialization of jobs that add some extra metadata and are intended for - you guessed it - _deployment_ jobs.

### Checkout
Classic builds implicitly checkout any repository artifacts, but pipelines require you to be more explicit using the `checkout` keyword:
- Jobs check out the repo they are contained in automatically unless you specify `checkout: none`.
- Deployment jobs do not automatically check out the repo, so you'll need to specify `checkout: self` for deployment jobs if you want to get access to files in the YML file's repo.

### Download
Downloading artifacts requires you to use the `download` keyword. Downloads also work the opposite way for jobs and dpeloyment jobs:
- Jobs do not download anything unless you explicitly define a `download`
- Deployment jobs implicitly perform a `download: current` which downloads any pipeline artifacts that have been created in the current pipeline. To prevent this, you must specify `download: none`.

### Resources
What if your job requires source code in another repository? You’ll need to use `resources`. Resources let you reference:
1. other repositories
1. pipelines
1. builds (classic builds)
1. containers (for container jobs)
1. packages

To reference code in another repo, specify that repo in the `resources` section and then reference it via its alias in the `checkout` step:

```yml
resources:
  repositories:
  - repository: appcode
    type: git
    name: otherRepo

steps:
- checkout: appcode
```

### Steps are Tasks
Steps are the actual "things" that execute, in the order that they are specified in the job. Each step is a task: there are out of the box (OOB) tasks that come with Azure DevOps, many of which have aliases, and there are tasks that get installed to your Azure DevOps account via the marketplace.

Creating custom tasks is beyond the scope of this chapter, but you can see how to create your own custom tasks here [https://docs.microsoft.com/en-us/azure/devops/extend/develop/add-build-task?view=azure-devops](https://docs.microsoft.com/en-us/azure/devops/extend/develop/add-build-task?view=azure-devops).

## Variables
It would be tough to achieve any sort of sophistication in your pipelines without variables. There are several types of variables, though this classification is mine and pipelines don’t distinguish between these types. However, I’ve found it useful to categorize pipeline variables to help teams understand some of the nuances that occur when dealing with them.

Every variable is really a key:value pair. The key is the name of the variable, and it has a value.

To dereference a variable, simply wrap the key in `$()`. Let’s consider this simple example:

```yml
variables:
  name: colin
steps:
-	script: echo "Hello, $(name)!"
```

This will write `Hello, colin!` to the log.

### Inline Variables
These are variables that are hard coded into the pipeline YML file itself. Use these for specifying values that are not sensitive and that are unlikely to change. A good example is an image name: let’s imagine you have a pipeline that is building a Docker container and pushing that container to a registry. You are probably going to end up referencing the image name in several steps (such as tagging the image and then pushing the image). Instead of using a value in-line in each step, you can create a variable and use it multiple times. This keeps to the DRY (Do not repeat yourself) principle and ensures that you don’t inadvertently misspell the image name in one of the steps.

TODO: var, docker build image tag, docker push image push

> **Note**: obviously you cannot create "secret" inline variables. If you need a variable to be secret, you’ll have to use pipeline variables, variable groups or dynamic variables.

### Predefined Variables
There are several predefined variables that you can reference in your pipeline. Examples are:
- Source branch: `Build.SourceBranch`
- Build reason: `Build.Reason`
- Artifact staging directory: `Build.ArtifactStagingDirectory`

You can find a full list of predefined variables here [https://docs.microsoft.com/en-us/azure/devops/pipelines/build/variables?view=azure-devops&tabs=yaml](https://docs.microsoft.com/en-us/azure/devops/pipelines/build/variables?view=azure-devops&tabs=yaml).

### Pipeline Variables
Pipeline variables are specified in Azure DevOps in the pipeline UI when you create a pipeline from the YML file. These allow you to abstract the variables out of the file. You can specify defaults and/or mark the variables as "secrets" (we’ll cover secrets a bit later). This is useful if you plan on triggering the pipeline manually and want to set the value of a variable at queue time.

> **Note**: if you specify a variable in the YML variables section, you cannot create a pipeline variable with the same name. If you plan on using pipeline variables, you must **not** specify them in the "variables" section.

When should you use pipeline variables? This is useful if you plan on triggering the pipeline manually and want to set the value of a variable at queue time. Imagine you sometimes want to build in `DEBUG` other times in `RELEASE`: you could specify `buildConfiguration` as a pipeline variable when you create the pipeline, giving it a default value of `debug`:

![Pipeline Variable](images/pipeline-var.png "Adding a pipeline variable")

If you specify `Let users override this value when running this pipeline` then users can change the value of the pipeline when they manully queue it. Specifying `Keep this value secret` will make this value a secret (Azure DevOps will mask the value).

Let's look at a simple pipeline that consumes the pipeline variable:

```yml
name: 1.0$(Rev:.r)

trigger:
- master

pool:
  vmImage: ubuntu-latest
  
jobs:
- job: echo
  steps:
  - script: echo "BuildConfiguration is $(buildConfiguration)"
```

Running the pipeline without editing the variable produces the following log:

![Debug Value](images/pipeline-var-run1.png "Default value in a run")

> **Tip**: If the pipeline is not manually queued, but triggered, any pipeline variables default to the value that you specify in the parameter when you create it.

If we update the value when we queue the pipeline to `release`, of course the log reflects the new value:

![Update the variable](images/pipeline-var-update1.png "Click Update when queueing the run")

![Enter the new value](images/pipeline-var-update2.png "Specify the new value")

![Release Value](images/pipeline-var-run2.png "Overriding the value in a run")

> **Note**: referencing a pipeline variable is exactly the same as referencing an inline variable – once again, the distinction is purely for discussion.

### Secrets
At some point you’re going to want a variable that isn’t visible in the build log: a password, an API Key etc. As I mentioned earlier, inline variables are never secret. You must mark a pipeline variable as secret in order to make it a secret, or you can create a dynamic variable that is secret.

"Secret" in this case just means that the value is masked in the logs. It is still possible to expose the value of a secret if you really want to. A malicious pipeline author could `echo` a secret to a file and then open the file to get the value of the secret.

All is not lost though: you can put controls in place to ensure that nefarious developers cannot simply run updated pipelines – you should be using Pull Requests and Branch Policies to review changes to the pipeline itself (an advantage to having pipelines as code). The point is, you still need to be careful with your secrets!

### Dynamic Variables and Logging Commands
Dynamic variables are variables that are created and/or calculated at run time. A good example is using the `az cli` to retrieve the connection string to a storage account so that you can inject the value into a web.config. Another example is dynamically calculating a build number in a script.

To create or set a variable dynamically, you can use [logging commands](https://docs.microsoft.com/en-us/azure/devops/pipelines/scripts/logging-commands?view=azure-devops&tabs=bash). Imagine you need to get the username of the current user for use in subsequent steps. Here’s how you can create a variable called `currentUser` with the value:

```yml
- script: |
    curUser=$(whoami)
    echo "##vso[task.setvariable variable=currentUser;]$curUser"
```

> **Tip**: when writing bash or PowerShell commands, don’t confuse `$(var)` with `$var`. `$(var)` is interpolated by Azure DevOps when the step is executed, while `$var` is a bash or PowerShell variable. I often use `env` to create environment variables rather than dereferencing variables inline. For example, I could write:

```yml
- script: echo $(Build.BuildNumber)
```

but I can also use environment variables:

```yml
- script: echo $buildNum
  env:
    buildNum: $(Build.BuildNumber)
```

This may come down to personal preference, but I’ve avoided confusion by consistently using env for my scripts!

> **Tip**: `env` works for PowerShell tasks too! However, you have to dereference using `$env:VAR`.

To make the variable a secret, simple add `issecret=true` into the logging command:

```yml
echo "##vso[task.setvariable variable=currentUser;issecret=true]$curUser"
```

You could do the same thing using PowerShell:
```yml
- powershell: |
    Write-Host "##vso[task.setvariable variable=currentUser;]$env:UserName"
```

> **Tip**: There are two flavors of PowerShell: `powershell` is for Windows and `pwsh` is for PowerShell Core which is cross-platform (so it can run on Linux and Mac!). 

One special case of a dynamic variable is a calculated build number. For that, calculate the build number however you need to and then use the `build.updatebuildnumber` logging command:

```yml
- script: |
    buildNum=$(...)  # calculate the build number somehow
    echo "##vso[build.updatebuildnumber]$buildNum"
```

Other logging commands are documented here: [https://docs.microsoft.com/en-us/azure/devops/pipelines/scripts/logging-commands?view=azure-devops&tabs=bash#build-commands](https://docs.microsoft.com/en-us/azure/devops/pipelines/scripts/logging-commands?view=azure-devops&tabs=bash#build-commands)

### Variable Groups
Creating inline variables is fine for values that are not sensitive and that are not likely to change very often. Pipeline variables are useful for pipelines that you want to trigger manuall. But there is another option that is particularly useful for multi-stage pipelines (we'll cover these in more detail later).

Imagine you have a web application (that connects to a database) that you want to build and then push to DEV, QA and Prod environments. Let's consider just one config setting - the database connection string. Where should you store the value for the connection string? Perhaps you could store the DEV connection string in source control, but what about QA and Prod? You probably don't want those passwords stored in source control.

You could create them as pipeline variables - but then you'd have to prefix the value with an environment or something to distinguish the QA value from the Prod value. What happens if you add in a STAGING environment? What if you have other settings like API Keys? This can quickly become a mess.

This is what Variable Groups are designed for. You can find variable groups in the `Library` hub in Azure DevOps:

![Accessing the Library](images/library.png "Accessing the Library")

The image above shows two variable groups: one for DEV and one for QA. Let's create a new one for Prod, specifying the same variable name (`ConStr`) but this time entering in the value for Prod:

![Adding a Variable Group](images/prod-var-group.png "Creating the Prod variable group")

> **Note**: Security is beyond the scope of this chapter - but you can specify who has permission to view/edit variable groups, as well as which pipelines are allowd to consume them. You can of course mark any value in the variable group as secret by clicking the padlock icon next to the value.

> **Tip**: The trick to making variable groups work for environment values is to keep the names the same in each variable group. That way the only setting you need to update between environments is the variable group name. I suggest getting the pipeline to work completely for one environment, and then `Clone` the variable group - that way you're assured you're using the same variable names.

#### KeyVault
You can also integrate variable groups to Azure KeyVaults. When you create the variable group, instead of specifying values in the variable group itself, you connect to a KeyVault and specify which keys from the vault should be synchronized when the variable group is instantiated in a pipeline run:

![KeyVault Variable Group](images/key-vault.png "Creating a KeyVault variable group")

#### Consuming Variable Groups
Now that we have some variable groups, we can consume them in a pipeline. Let's consider this pipeline:

```yml
trigger:
- master

pool:
  vmImage: ubuntu-latest
  
jobs:
- job: DEV
  variables:
  - group: WebApp-DEV
  - name: environment
    value: DEV
  steps:
  - script: echo "ConStr is $(ConStr) in enviroment $(environment)"

- job: QA
  variables:
  - group: WebApp-QA
  - name: environment
    value: QA
  steps:
  - script: echo "ConStr is $(ConStr) in enviroment $(environment)"

- job: Prod
  variables:
  - group: WebApp-Prod
  - name: environment
    value: Prod
  steps:
  - script: echo "ConStr is $(ConStr) in enviroment $(environment)"
```

When this pipeline runs, we see the DEV, QA and Prod values from the variable groups:

![Value in DEV](images/var-group-dev.png "Value in DEV")

![Value in Prod](images/var-group-prod.png "Value in Prod")

> **Note**: The format for inline varialbes alters slightly when you have variable groups - you have to use the `- name/value` format.

This example demonstrates a couple of things:
1. Integration with variable groups
1. Job variables - that is, jobs can have variables too, not just pipelines
1. You can specify multiple jobs in a single pipeline - in fact, as we'll see later, pipelines can have multiple stages, and each stage can have multiple jobs.

What you may have noticed is that every job has the same steps - so there is plenty of copy/paste going on here! Surely there's a better way to repeat common steps? Yes, there is a btter way - we are creating code, after all! Enter templates...

## Templates
Templates are like _functions_ - they allow reuse and much better maintainability. There are two types of templates: _step_ templates and _job_ templates. 

### Step Templates
Let's consider the previous example pipeline. Each job had exactly the same `steps` section:

```yml
  steps:
  - script: echo "ConStr is $(ConStr) in enviroment $(environment)"
```

What if we created a template for the common steps so that we only had to maintain it in a single place? All we have to do is place the steps into a separate steps YML file - and we can even parameterize the template:

```yml
# templates/steps.yml
parameters:
  owner: ''

steps:
- script: echo "ConStr is $(ConStr) in enviroment $(environment) and is owned by ${{ parameters.owner }}"
```

This is familiar, but slightly different. The primary difference is the way that parameters are dereferenced, using `${{}}`, which is different from the way variables are deferenced using `$()`. We'll discuss the differences between parameters and variables in more detail later.

Given this template, we can update the main pipeline:
```yml
trigger:
- master

pool:
  vmImage: ubuntu-latest
  
jobs:
- job: DEV
  variables:
  - group: WebApp-DEV
  - name: environment
    value: Dev
  steps:
  - template: templates/webapp-steps.yml
    parameters:
      owner: Dylan

- job: QA
  variables:
  - group: WebApp-QA
  - name: environment
    value: QA
  steps:
  - template: templates/webapp-steps.yml
    parameters:
      owner: Bob

- job: Prod
  variables:
  - group: WebApp-Prod
  - name: environment
    value: Prod
  steps:
  - template: templates/webapp-steps.yml
    parameters:
      owner: Sally
```

As expected, the output for the QA environment is showing all the QA-specific values, both for the variables as well as the parameter:

![Step Template Run](images/step-template-run.png "Values in QA for the step template run")

We can also create job templates.

### Job Templates
Job templates are essentially the same as step templates - but they allow us to specify entire jobs, not just sets of steps, as a template. This is particularly useful for deployment scenarios, where we want to use the same job to deploy to multiple environments, just with environment-specific values. This is touching on multi-stage pipelines, which we'll get to shortly. For now, let's look at what you can do with job templates.

TODO: from here!!

### Parameters vs Vars
explain difference
string vs object

### Expressions
show examples
if
loop
inject steps

## Multi-Stage Pipelines
Intro; why this is necessary; discussion of CI/CD and differences
### Deployment jobs
show example

### Environments; 
authorization, checks, k8s environments

### Endpoints; 
Azure endpoint, generic endpoint

## Advanced Topics
### Container jobs; discuss, show example
### Decorators
### Pipelines API
### Schemas; ?? show schema, discuss why you would need it (control of inheritance etc.)

## Lessons Learned
Auditability vs segregation of duty; 
discuss coding everything vs coding just steps and having vars separate

