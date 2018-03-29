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



### Useful links

Noel Bundick has a bunch of useful commands on his GitHub - check them out [here](https://github.com/noelbundick/ContainerTools). 





### Links

- [Useful PowerShell commands](https://github.com/noelbundick/ContainerTools)

- [Docker for Windows](https://docs.docker.com/docker-for-windows/)
- [Visual Studio Code](https://code.visualstudio.com/)
- [Windows 10 Pro 64-bit](https://www.microsoft.com/en-au/software-download/windows10). The current version of Docker for Windows runs on 64bit Windows 10 Pro, Enterprise and Education (1607 Anniversary Update, Build 14393 or later)
- [Hyper Terminal](https://github.com/zeit/hyper)
- [Node.js](https://nodejs.org/en/download/current/). Ensure you install current. 
- [acs-engine](https://github.com/Azure/acs-engine)
- [acs-engine Yeoman Generator](https://github.com/jakkaj/generator-acsengine)
- [Kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/)
