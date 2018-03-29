# Windows Containers
Windows containers, Classic ASP, COM+, IIS and 32-bit code. Kubernetes. What else could you want?

This artical and code explores setting up a Windows Docker container based on Windows Server Core 1709. 

Getting your own app up and running inside a container can be a non-trivial task, especially if it's an older app. You'll need to make sure you have the ability to automatically install/deploy the app in non-interactive mode as Windows Server Core has no GUI!

Rather than bog down in all that, this article explores some of the basic building blocks of getting legacy Windows apps up and running inside a Windows Docker container before delivering that container to a Kubernetes cluster on Azure. 

### What's Covered Here?

We'll cover how to set up the following inside a Docker container. 

- COM+ (32-bit)
- VB6 based Classic ASP site
- Configuring IIS
    - Configure IIS features and app pool
- Windows Services

### Getting Started

You'll need Windows 10 (latest build) as well as the latest build of Docker for Windows to support Windows based containers. 

#### The Docker Bits

- [Windows 10 Pro 64-bit](https://www.microsoft.com/en-au/software-download/windows10). The current version of Docker for Windows runs on 64bit Windows 10 Pro, Enterprise and Education (1607 Anniversary Update, Build 14393 or later)
- [Docker for Windows](https://docs.docker.com/docker-for-windows/)
- [Visual Studio Code](https://code.visualstudio.com/)
- [Hyper Terminal](https://github.com/zeit/hyper). Not essential, but highly recommend as there will be a bit of terminal based work. 

#### The Kubernetes Bits

- [Node.js](https://nodejs.org/en/download/current/). Ensure you install current. 
- [acs-engine](https://github.com/Azure/acs-engine)
- [acs-engine Yeoman Generator](https://github.com/jakkaj/generator-acsengine)
- [Kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/)

## Prerequisites

This article assumes basic Docker knowledge. It also assumes basic PowerShell knowledge. 

### Let's go!

#### Switch to Windows Containers

Right click on the Docker icon in the system tray and select "Switch to Windows containers". If you try and pull an image for the wrong OS you'll get an error. 

#### Pull Windows Server Core

First up - before you do anything else, pull the Windows Server Core 1709 image to your local machine - it's going to take a little while!

```
docker pull microsoft/windowsservercore:1709
```

This will ensure you have a local cache of the Server Core image ready for when you build your Dockerfile. 

### Start the Dockerfile

This container will be based on Windows Server Core. We'll start super simple just to make sure you can get a container up and running. 

Create a new file called "Dockerfile". 

In it place: 

```Dockerfile
FROM microsoft/iis
RUN echo "This is my IIS based app" > c:\inetpub\wwwroot\index.html
```

This image is based on [microsoft/iis](https://hub.docker.com/r/microsoft/iis/). See the [Dockerfile](https://github.com/Microsoft/iis-docker/blob/master/windowsservercore-1709/Dockerfile) here. You can spot that it's based on "microsoft/windowsservercore:1709". Lucky you already pulled that image!

Of note in the microsoft/iis Dockerfile is the last line - `ENTRYPOINT ["C:\\ServiceMonitor.exe", "w3svc"]`. Docker containers run until the process they are watching exits. In this case, the process to be watched is IIS, but you don't just add a `CMD` or `ENTRYPOINT` command to fire up IIS - it's managed by windows. [ServiceMonitor](https://github.com/Microsoft/IIS.ServiceMonitor) is a tool that does just this - it watches any Windows service until it exits. 

ServiceMonitor is installed during the container build process by running some PowerShell that downloads the exe to the container. This is common practice for utilities (downloading them) - and indeed it's utilitsed further on in this article. 

```PowerShell
RUN powershell -Command `
    Add-WindowsFeature Web-Server; `
    Invoke-WebRequest -UseBasicParsing -Uri "https://dotnetbinaries.blob.core.windows.net/servicemonitor/2.0.1.2/ServiceMonitor.exe" -OutFile "C:\ServiceMonitor.exe"
```

#### Note the Escape!

Of note is that normally in Dockerfiles the escape line character is "\". This is hard work when dealing with PowerShell scripts where it has meaning. So the first line in the Dockerfile is `# escape=`` which will change the escape char to make things a little easier. 

All in all however, the Docker file you need to create doesn't have to worry about any of that... just yet. You can get all the greatness in the microsoft/iis container including escapes, and ServiceMonitor just by inheriting your file from theirs. 

```Dockerfile
FROM microsoft/iis
RUN echo "This is my IIS based app" > c:\inetpub\wwwroot\index.html
```

Now that you have the Dockerfile ready it's time to build it. 

From a terminal navigate to the folder that your Dockerfile is in and type:

```
docker build -t simpleiis .
```

![Building a container](https://user-images.githubusercontent.com/5225782/38065067-03a822a4-334d-11e8-8575-0c56be210fc1.gif)

Once that's built, you can fire it up to test. 

### Run the test container

Note that the container exposes port 80. You can tell this by looking in the [microsoft/iis](https://github.com/Microsoft/iis-docker/blob/master/windowsservercore-1709/Dockerfile) Dockerfile - it says `EXPOSE 80`. For now, Docker is smart enough to expose that automatically if you ask it. 

```
docker run --rm -t -P simpleiis
```

![Docker Run](https://user-images.githubusercontent.com/5225782/38065394-db302cc0-334e-11e8-9d33-88d1d2322392.gif)

The `--rm` switch removes the container when it exits. The `-P` switch tells Docker to map exposed ports automatically. The `-t` command is not directly used here, but later on it may become handy - it maps everything the container outputs from the running process so it can be displayed later to assist with debugging. It's a good idea to get used to using it. 

Once it's running type `docker ps -a` to show running containers. Find the local port of the container that's been exposed - in this example look for 0.0.0.0:24890. Visit localhost:25890. Yours will differ. 

That's it for the simple container!

To stop your container type `docker stop <containerid>`. 

### Let's get more advanced

This container is great - it demonstrates how easy it is to get IIS up and running inside a Windows container. Let's build something more substantial and customisable. 

This section is based on this [Dockerfile](https://github.com/CSEANZ/WindowsContainers/blob/master/Docker/WindowsAndIIS/Dockerfile) by [Regan Murphy](https://twitter.com/nzregs). 

#### Kick things off

```Dockerfile
# escape=`
FROM microsoft/windowsservercore:1709
```

Sets the escape char to ``` and inherits from the "microsoft/windowsservercore:1709" base image. It does not inherit from "microsoft/iis" as this one is more customisable. 

#### Prep the environment

```Dockerfile
SHELL ["powershell", "-command", "$ErrorActionPreference = 'Stop'; $ProgressPreference = 'SilentlyContinue';"]
```

This line makes Windows Powershell the default prompt when using the `RUN` command in your Dockerfile. Makes things nice and easy!

#### Install container tools

```Dockerfile
# Install ContainerTools
ENV ContainerToolsVersion=0.0.1
RUN Install-PackageProvider -Name NuGet -MinimumVersion 2.8.5.201 -Force; `
    Set-PsRepository -Name PSGallery -InstallationPolicy Trusted; `
    Install-Module -Name ContainerTools -MinimumVersion $Env:ContainerToolsVersion -Confirm:$false
```

[ContainerTools](https://github.com/noelbundick/ContainerTools) is an alternate method to watching services, logs and to force the container to wait. We're not using it directly in this example, but check it out. It's work knowing about / trying out. 

#### Enable IIS Windows features

```Dockerfile
# Install IIS, add features, enable 32bit on AppPool
RUN Install-WindowsFeature -name Web-Server; `
    Add-WindowsFeature Web-Static-Content, Web-ASP, WoW64-Support; `
    Import-Module WebAdministration; `
    set-itemProperty IIS:\apppools\DefaultAppPool -name "enable32BitAppOnWin64" -Value "true"; `
    Restart-WebAppPool "DefaultAppPool"
```
Here we're turning on IIS with various features (like Classic ASP! In the modern container world!) as well as creating the default app pool which supports 32bit mode. 

```Dockerfile
# Download the com components and test pages from github
RUN [Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12; `
    Invoke-WebRequest -Uri https://github.com/nzregs/vb6MathLib/raw/master/MathLibNZRegs.dll -OutFile c:\inetpub\wwwroot\MathLibNZRegs.dll; `
    Invoke-WebRequest -Uri https://github.com/nzregs/vb6MathLib/raw/master/msvbvm60.dll -OutFile c:\inetpub\wwwroot\msvbvm60.dll; `
    Invoke-WebRequest -Uri https://github.com/nzregs/vb6MathLib/raw/master/default.asp -OutFile c:\inetpub\wwwroot\default.asp; `
    Invoke-WebRequest -Uri https://github.com/nzregs/vb6MathLib/raw/master/test.asp -OutFile c:\inetpub\wwwroot\test.asp; `
    $regsvr = [System.Environment]::ExpandEnvironmentVariables('%windir%\SysWOW64\regsvr32.exe'); `
    Start-Process $regsvr  -ArgumentList '/s', "c:\inetpub\wwwroot\msvbvm60.dll" -Wait; `
    Start-Process $regsvr  -ArgumentList '/s', "c:\inetpub\wwwroot\MathLibNZRegs.dll" -Wait
```
This section is downloading some COM components and ASP pages from Regans [public GitHub](https://github.com/nzregs) and saving them to the local path before running regsvr32 on them to register them as COM components with Windows. These .dll files are VB6 based components. 

```Dockerfile
RUN powershell -Command `
    Add-WindowsFeature Web-Server; `
    Invoke-WebRequest -UseBasicParsing -Uri "https://dotnetbinaries.blob.core.windows.net/servicemonitor/2.0.1.2/ServiceMonitor.exe" -OutFile "C:\ServiceMonitor.exe"

EXPOSE 80

ENTRYPOINT ["C:\\ServiceMonitor.exe", "w3svc"]
```

This section downloads ServiceMonitor (like before), exposes port 80 from the container, and sets the entry point to have ServiceMonitor watch w3svc, the IIS service. 

### Build the custom container

Just like before, build this dockerfile (switch to the path in terminal first). 

```
docker build -t classicaspiis .
docker run --rm -t -P classicaspiis
docker ps -a
```

Find the port in the output listing and navigate to that path in your browser. 

![ASP in a container!](https://user-images.githubusercontent.com/5225782/38066956-026b932c-3356-11e8-87ed-b5c1f02f7a83.PNG)

That's it,  you're running Classic ASP, COM+, 32bit and the rest inside a Windows container. 

## Kubernetes and Windows Containers

Azure has a variety of options when it comes to containers and orchestrators - including Kubernetes. 

One service is the [Azure Container Service (managed) or AKS](https://docs.microsoft.com/en-us/azure/aks/). This services sets up a cluster for you and manages the running, updating and general maintenance of that cluster for you. The problem is that right now AKS doesn't support Windows containers. That means we need a more customised approach - but fear not... this is an easy process. 

Underneath the hood, AKS uses something called the [acs-engine](https://github.com/Azure/acs-engine). This engine allows you to generate [ARM](https://docs.microsoft.com/en-gb/azure/azure-resource-manager/resource-group-overview) templates which will go and set up a cluster for you. 

### Set up a Kubernetes cluster on Azure

To help with this process we'll use a [Yeoman Generator](http://yeoman.io/) to help get started. 

Full generator instructions are located [here](https://github.com/jakkaj/generator-acsengine). Head there to see the full requirements and getting started guide. Remember to check out the [instructional video](https://www.youtube.com/watch?v=J3CZkL7rt6Y&feature=youtu.be) if you get stuck. 

Make sure you:

- Install Node.ks 
- Install the generator using npm
- Generate a service principal
- Get your Azure subscription id
- Pick a region
- Pick a dns name
- Pick a resource group name

**Note:** We really suggest you choose a new unique resource group name and don't reuse an existing one. This way you can easily clean up your cluster later. Also, it's a good idea to create your Container Registry in a different group to your cluster so you can create and delete clusters without destroying your private registry. 

Make a new folder and run the generator by typing `yo acsengine` and enter the required fields. 

This will produce an acs-engine template as well as some Powershell scripts to help get the cluster started. 

Run the scripts as per the [instructions](https://github.com/jakkaj/generator-acsengine) ([video](https://www.youtube.com/watch?v=J3CZkL7rt6Y&feature=youtu.be)) and soon enough you'll have a cluster. 

### Create a container registry

In order to utilise your new containers in Kubernetes you'll need a container registry. [Docker Hub](https://hub.docker.com/) is one example of a registry. Azure provides a private container resistry you can use - which thanks to your service principle will be accessible to your Kubernets cluster. Check out the [Quickstarts](https://docs.microsoft.com/en-gb/azure/container-registry/) to set one up.

Once you done that, grab the admin user and password from the "Access keys" tab. You may need to enable the Admin user mode to grab them.

Also copy the name of the registry - e.g "someregistry.azurecr.io". 

At the command login and enter your credentials. 

```
docker login someregistry.azurecr.io
```

### Push the image to the private registry

Pushing images is easy. You're logged in already, you just need to tag the image correctly to push it to the registry. It's based on convention of "reponame/containername". In this case it woudl be "someregistry.azurecr.io/classicaspiis". You can also put a version number on that so you can deploy newer versions with the same name e.g. "someregistry.azurecr.io/classicaspiis:1". 

You do not have to rebuild the container to "tag" it with a new name. 

```
docker tag classicaspiis someregistry.azurecr.io/classicaspiis:1
```

Now type `docker images` to see the newly tagged image name as well as the original. Note they both have the same IMAGE ID. Cool eh?

```
docker push someregistry.azurecr.io/classicaspiis:1
```

This will push the image up to the container registry ready for consumption inside Kubernetes. 

### Check the progress of your cluster

After a while your cluster will be ready. 

"kubectl" is the most commonly used tool to investigate and alter Kubernetes clusters. There are others (including [graphical](https://kubernetic.com/) ones) , but for now this will do. 

Before kubectl can explore your cluster, it needs to be configured. By default there is a "config" file in ~/.kube/config. This file may or may not exist on your machine. In the case of the yo generator you just ran, there is a script (`4_set_kubectl_config.ps1`) that will temporarily set up an environment var in your current Powershell session for kubectl to use. If you like you can take the file this powershell scipt points to and manually integrate it in to ~/.kube/config.

The acs-engine has output a range of config files for use depending on the region you selected - look in "_output\<dnsname>\kubeconfig" for the files. `4_set_kubectl_config.ps1` will automatically be referencing the correct one.  

**Warning:** the files acs-engine outputs are .json, kubectl defaults to .yaml - you may need to use something like [this](https://kubernetes-v1-4.github.io/docs/user-guide/kubectl/kubectl_config_view/) with the `--flatten` option to help out with the migration. 

Once this is done, type

```
kubectl cluster-info 
```
Check that the cluster outputs are what you expect. 

You can now create some [pods](https://kubernetes.io/docs/concepts/workloads/pods/pod/)!

### Deploy your container

At a high level containers are deployed in pods, by deployments and exposed by services. 

In the files generated by the yo generator, navigate to the "kube" path. In there are two files, one each for Windows and Linux. 

Open "kube.windows.yaml". Find the line:

```yaml
image: jakkaj/reganasp:1
```

Replace it with the image you uploaded to the private registry e.g. `someregistry.azurecr.io/classicaspiis:1`. 

Type `kubectl apply -f kube.windows.yaml'. This will push up the desired state to the cluster which will create the pods and services. 

To see how to pod is going type `kubectl get pods`. 

To see the public ip for the service type `kubectl get svc`. 

If you have any problems, list the pods, then ask for more details on that pod. 

```
kubectl get pods
kubectl describe pod <somepodid>
```

Once the pod is listed as "Ready", grab the external ip from `kubectl get svc` and navigate to it. You will see your Classic ASP site running in you new Kubernetes cluster. 

### Cleaning Up

If you want to remove your cluster you can run `powershell\x_delete_resource_group.ps1`. Please remember that this will kill everything in the resource group - including the container registry if you created it here. It will also kill anything that was **already in the group**. If you didn't create the resource group just for this exercise, don't delete it until you're sure what's in there!

 

## Useful Links

- [Useful PowerShell commands](https://github.com/noelbundick/ContainerTools)

- [Docker for Windows](https://docs.docker.com/docker-for-windows/)
- [Visual Studio Code](https://code.visualstudio.com/)
- [Windows 10 Pro 64-bit](https://www.microsoft.com/en-au/software-download/windows10). The current version of Docker for Windows runs on 64bit Windows 10 Pro, Enterprise and Education (1607 Anniversary Update, Build 14393 or later)
- [Hyper Terminal](https://github.com/zeit/hyper)
- [Node.js](https://nodejs.org/en/download/current/). Ensure you install current. 
- [acs-engine](https://github.com/Azure/acs-engine)
- [acs-engine Yeoman Generator](https://github.com/jakkaj/generator-acsengine)
- [Kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/)
- [Azure Container Service (managed) or AKS](https://docs.microsoft.com/en-us/azure/aks/)
- [Azure Resource Manager](https://docs.microsoft.com/en-gb/azure/azure-resource-manager/resource-group-overview) 
- [Kubernetes Pods](https://kubernetes.io/docs/concepts/workloads/pods/pod/)
- [Kubernetic Kubernets GUI](https://kubernetic.com/)
- [Kubectl config view documentation](https://kubernetes-v1-4.github.io/docs/user-guide/kubectl/kubectl_config_view/)