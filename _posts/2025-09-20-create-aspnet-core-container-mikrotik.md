---
title: "Create an ASP.NET Core Container For a Mikrotik Router"
date: 2025-09-20T08:00:00-04:00
categories:
  - How-To
tags:
  - ASP.NET Core
  - Mikrotik
  - Container
---

In this post, I will walk through the process of building a container for the `React and ASP.NET Core` project template for a Mikrotik L009UiGS.

<!-- More -->

# Prepare the project

Update the default `dockerfile` as detailed in the comments within the file below.


```docker
# See https://aka.ms/customizecontainer to learn how to customize your debug container and how Visual Studio uses this Dockerfile to build your images for faster debugging.

# This stage is used when running from VS in fast mode (Default for Debug configuration)
# 1) (Optional) Optimize base image size by switching to a chiseled image
FROM mcr.microsoft.com/dotnet/aspnet:8.0-jammy-chiseled AS base
USER $APP_UID
WORKDIR /app
EXPOSE 8080
EXPOSE 8081


# This stage is used to build the service project
# 2) Add --platform argument
FROM --platform=$BUILDPLATFORM mcr.microsoft.com/dotnet/sdk:8.0 AS with-node
RUN apt-get update
RUN apt-get install curl
RUN curl -sL https://deb.nodesource.com/setup_20.x | bash
RUN apt-get -y install nodejs


# 3) Add --platform argument
FROM --platform=$BUILDPLATFORM with-node AS build
ARG BUILD_CONFIGURATION=Release
# 4) Save runtime in /tmp/rid for use later
ARG TARGETPLATFORM
RUN if [ "$TARGETPLATFORM" = "linux/arm/v7" ] ; then DOTNET_TARGET=linux-arm ; else DOTNET_TARGET=linux-x64 ; fi && echo $DOTNET_TARGET > /tmp/rid
WORKDIR /src
COPY ["Demo.Server/Demo.Server.csproj", "Demo.Server/"]
COPY ["demo.client/demo.client.esproj", "demo.client/"]
# 5) Pass runtime value to restore
RUN dotnet restore "./Demo.Server/Demo.Server.csproj" -r $(cat /tmp/rid)
COPY . .
WORKDIR "/src/Demo.Server"
# 6) Pass runtime value to build
RUN dotnet build "./Demo.Server.csproj" -c $BUILD_CONFIGURATION -o /app/build -r $(cat /tmp/rid)

# This stage is used to publish the service project to be copied to the final stage
# 7) Add --platform argument
FROM --platform=$BUILDPLATFORM build AS publish
ARG BUILD_CONFIGURATION=Release
# 8) Save runtime in /tmp/rid for use later
ARG TARGETPLATFORM
RUN if [ "$TARGETPLATFORM" = "linux/arm/v7" ] ; then DOTNET_TARGET=linux-arm ; else DOTNET_TARGET=linux-x64 ; fi && echo $DOTNET_TARGET > /tmp/rid
# 9) Pass runtime to publish
RUN dotnet publish "./Demo.Server.csproj" -c $BUILD_CONFIGURATION -o /app/publish /p:UseAppHost=false -r $(cat /tmp/rid)

# This stage is used in production or when running from VS in regular mode (Default when not using the Debug configuration)
FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "Demo.Server.dll"]
```

# Building the project

First, uncheck the "Use containerd for pulling and storing images" in the Docker Desktop settings (otherwise, in my testing, the exported .tar archive was formatted in such a way that the Mikrotik failed to extract the container image).

Next, to build the project, and export the container into a .tar file, run the following commands in the solution directory.

```
docker build . -f ./Demo.Server/Dockerfile --target final --platform=linux/arm/v7 -t demo:latest

docker save -o demo.tar demo:latest
```


# Setting up the router

## Formatting USB Storage

If you have `ext4` formatted USB storage already, skip this section.

Otherwise, insert a drive to format for the router. Run the following command on the router: 

```
/disk format usb1 file-system=ext4 mbr-partition-table=no
```

The drive format does appear to be important as later steps failed with a `fat32` formatted drive.

## Prepare drive

Create the following directory structure for container data, run the following commands on the rotuer:

```
/file add type=directory name=usb1/container
/file add type=directory name=usb1/container/image
/file add type=directory name=usb1/container/rootdir
/file add type=directory name=usb1/container/workdir
```

## Set up the container networking

Following some of the guidance outlined in the [Container documentation by Mikrotik](https://help.mikrotik.com/docs/spaces/ROS/pages/84901929/Container):
* create a virtual interface for the container
* a bridge for containers
* an IP address for the container
* an IP address for the container on the local network

These commands assume the following:
* The LAN address space is 192.168.0.0/24
* The virtual network 172.17.0.0/24 has not been configured
    * Also - this is an acceptable address space based on the routers current configuration
* 192.168.0.231 is ununsed on the LAN

With those assumptions, the commands ready an IP address of 172.17.0.231/24 for the container and forward traffic from 192.168.0.231 to 172.17.0.231, thus making the container accessible on the LAN.

If necessary change the values below, then execute the commands on the router.

```
/interface/veth/add name=veth1 address=172.17.0.231/24 gateway=172.17.0.1
/interface/bridge/add name=containers
/ip/address/add address=172.17.0.1/24 interface=containers
/interface/bridge/port add bridge=containers interface=veth1
/ip/address/add address=192.168.0.231/24 interface=ether1
/ip/firewall/nat/add action=src-nat chain=srcnat src-address=172.17.0.231 out-interface-list=WAN to-addresses=192.168.0.231
/ip/firewall/nat/add action=dst-nat chain=dstnat dst-address=192.168.0.231 in-interface-list=WAN to-addresses=172.17.0.231
```

## Env Variables

Below is an example of setting environment values. These values would likely appear within the `appsettings.json` and/or `appsettings.Development.json` files in the project.

```
/container/envs/add name=Demo key=DEMO_Authorization__Username value="admin"
/container/envs/add name=Demo key=DEMO_Authorization__Password value="admin"
```

Note the `name` value in the `/add` command is the name of the environment value group, this will be used when adding the container.

### Sample appsettings.json

```xml
{
  "Authorization": {
    "Username": "admin",
    "Password":  "password"
  },
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  }
}
```

### Reading configuration

The commands in the previous section assume the environment variables are read with the `DEMO_` prefix, adjust as needed.

```C#
builder.Configuration.AddEnvironmentVariables(prefix: "DEMO_").AddCommandLine(args);
```

## Install container

Transfer the container .tar archive into the `usb1/container/image` directory on the router. For example, using the following `scp` command:

```bash
scp demo.tar 192.168.0.1:/container/image/demo.tar
```

Once transferred, run the following command on the router

```
/container/add file=usb1/container/image/demo.tar interface=veth1 envlist=Demo root-dir=usb1/container/rootdir/demo logging=yes
```

Start the container.

## References

Mikrotik
* [Containers](https://help.mikrotik.com/docs/spaces/ROS/pages/84901929/Container)
* [L009UiGS Product Page](https://mikrotik.com/product/l009uigs_rm)