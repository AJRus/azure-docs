---
title: Develop module for Windows devices - Azure IoT Edge | Microsoft Docs
description: This tutorial walks through setting up your development machine and cloud resources to develop IoT Edge modules using Windows containers for Windows devices
author: kgremban
manager: philmea
ms.author: kgremban
ms.date: 04/20/2019
ms.topic: tutorial
ms.service: iot-edge
services: iot-edge
ms.custom: mvc
---

# Tutorial: Develop IoT Edge modules for Windows devices

Use Visual Studio 2017 to develop and deploy code to Windows devices running IoT Edge.

In the quickstart, you created an IoT Edge device using a Windows virtual machine and deployed a pre-built module from the Azure Marketplace. This tutorial walks through what it takes to develop and deploy your own code to an IoT Edge device. This tutorial is a useful prerequisite for all the other tutorials, which will go into more detail about specific programming languages or Azure services. 

This tutorial uses the example of deploying a **C module to a Windows device**. This example was chosen for its simplicity, so that you can learn about the development tools without worrying about whether you have the right libraries installed. Once you understand the development concepts, then you can choose your preferred language or Azure service to dive into the details. 

In this tutorial, you learn how to:

> [!div class="checklist"]
> * Set up your development machine.
> * Use the IoT Edge tools for Visual Studio 2017 to create a new project.
> * Build your project as a container and store it in an Azure container registry.
> * Deploy your code to an IoT Edge device. 

[!INCLUDE [quickstarts-free-trial-note](../../includes/quickstarts-free-trial-note.md)]


## Key concepts

This tutorial walks through the development of an IoT Edge module. An *IoT Edge module*, or sometimes just *module* for short, is a container that contains executable code. You can deploy one or more modules to an IoT Edge device. Modules perform specific tasks like ingesting data from sensors, performing data analytics or data cleaning operations, or sending messages to an IoT hub. For more information, see [Understand Azure IoT Edge modules](iot-edge-modules.md).

When developing IoT Edge modules, it's important to understand the difference between the development machine and the target IoT Edge device where the module will eventually be deployed. The container that you build to hold your module code must match the operating system (OS) of the *target device*. For Windows container development, this concept is simpler because Windows containers only run on Windows operating systems. But you could, for example, use your Windows development machine to build modules for Linux IoT Edge devices. In that scenario, you'd have to make sure that your development machine was running Linux containers. As you go through this tutorial, keep in mind the difference between *development machine OS* and the *container OS*.

This tutorial targets Windows devices running IoT Edge. Windows IoT Edge devices use Windows containers. We recommend using Visual Studio 2017 to develop for Windows devices, so that's what this tutorial will use. You can use Visual Studio Code as well, although there are differences in support between the two tools.

The following table lists the supported development scenarios for **Windows containers** in Visual Studio Code and Visual Studio 2017.

|   | Visual Studio Code | Visual Studio 2017 |
| - | ------------------ | ------------------ |
| **Azure services** | Azure Functions <br> Azure Stream Analytics |   |
| **Languages** | C# (debugging not supported) | C <br> C# |
| **More information** | [Azure IoT Edge for Visual Studio Code](https://marketplace.visualstudio.com/items?itemName=vsciot-vscode.azure-iot-edge) | [Azure IoT Edge Tools for Visual Studio 2017](https://marketplace.visualstudio.com/items?itemName=vsc-iot.vsiotedgetools) |

This tutorial teaches the development steps for Visual Studio 2017. If you would rather use Visual Studio Code, refer to the instructions in [Use Visual Studio Code to develop and debug modules for Azure IoT Edge](how-to-vs-code-develop-module.md).

## Prerequisites

A development machine:

* Windows 10 with 1809 update or newer.
* You can use your own computer or a virtual machine, depending on your development preferences.
* Install [Git](https://git-scm.com/). 
* Install the Azure IoT C SDK for Windows x64 through vcpkg:

   ```powershell
   git clone https://github.com/Microsoft/vcpkg
   cd vcpkg
   .\bootstrap-vcpkg.bat
   .\vcpkg install azure-iot-sdk-c:x64-windows
   .\vcpkg --triplet x64-windows integrate install
   ```

<!--vcpkg only required for C development-->

An Azure IoT Edge device on Windows:

* We recommend that you don't run IoT Edge on your development machine, but instead use a separate device. This distinction between development machine and IoT Edge device more accurately mirrors a true deployment scenario, and helps to keep the different concepts straight.
* If you don't have a second device available, use the quickstart article to create an IoT Edge device in Azure with a [Windows virtual machine](quickstart.md).

Cloud resources:

* A free or standard-tier [IoT hub](../iot-hub/iot-hub-create-through-portal.md) in Azure. 

## Install container engine

IoT Edge modules are packaged as containers, so you need a container engine on your development machine to build and manage the containers. We recommend using Docker Desktop for development because of its many features and popularity as a container engine. With Docker Desktop on a Windows computer, you can switch between Linux containers and Windows containers so that you can easily develop modules for different types of IoT Edge devices. 

Use the Docker documentation to install on your development machine: 

* [Install Docker Desktop for Windows](https://docs.docker.com/docker-for-windows/install/)

  * When you install Docker Desktop for Windows, you're asked whether you want to use Linux or Windows containers. For this tutorial, use **Windows containers**. For more information, see [Switch between Windows and Linux containers](https://docs.docker.com/docker-for-windows/#switch-between-windows-and-linux-containers).


## Set up Visual Studio and tools

Use the IoT extensions for Visual Studio 2017 to develop IoT Edge modules. These extensions provide project templates, automate the creation of the deployment manifest, and allow you to monitor and manage IoT Edge devices. In this section, you install Visual Studio and the IoT Edge extension, then set up your Azure account to manage IoT Hub resources from within Visual Studio. 

1. If you don't already have Visual Studio on your development machine, [Install Visual Studio 2017](https://docs.microsoft.com/visualstudio/install/install-visual-studio?view=vs-2017) with the following workloads: 

   * Azure development
   * Desktop development with C++
   * .NET Core cross-platform development

1. If you do already have Visual Studio 2017 on your development machine, make sure that its version is 15.7 or higher. Follow the steps in [Modify Visual Studio](https://docs.microsoft.com/visualstudio/install/modify-visual-studio?view=vs-2017) to add the required workloads if you don't have them already.

2. Download and install the [Azure IoT Edge Tools](https://marketplace.visualstudio.com/items?itemName=vsc-iot.vsiotedgetools) extension for Visual Studio 2017. 

3. When your installations are complete, open Visual Studio.

4. Select **View** > **Cloud Explorer**. 

5. Select the profile icon in the cloud explorer and sign in to your Azure account if you aren't signed in already. 

6. Once you sign in, your Azure subscriptions are listed. Select the subscriptions that you want to access through the cloud explorer and then select **Apply**. 

7. Expand your subscription, then **IoT Hubs**, then your IoT hub. You should see a list of your IoT devices, and can use this explorer to manage them. 

   ![Access IoT Hub resources in Cloud Explorer](./media/tutorial-develop-for-windows/cloud-explorer-view-hub.png)

[!INCLUDE [iot-edge-create-container-registry](../../includes/iot-edge-create-container-registry.md)]

## Create a new module project

The Azure IoT Edge Tools extension provides project templates for all supported IoT Edge module languages in Visual Studio 2017. These templates have all the files and code that you need to deploy a working module to test IoT Edge, or give you a starting point to customize the template with your own business logic. 

1. Run Visual Studio as an administrator.

2. Select **File** > **New** > **Project**. 

3. In the new project window, select the **Azure IoT** project type and choose the **Azure IoT Edge** project. Rename the project and solution, or accept the default **AzureIoTEdgeApp1**. Select **OK** to create the project. 

   ![Create a new Azure IoT Edge project](./media/tutorial-develop-for-windows/new-project.png)

4. In the IoT Edge application and module window, configure your project with the following values: 

   | Field | Value |
   | ----- | ----- |
   | Application platform | Uncheck **Linux Amd64**, and check **WindowsAmd64**. |
   | Select a template | Select **C Module**. | 
   | Module project name | Accept the default **IoTEdgeModule1**. | 
   | Docker image repository | An image repository includes the name of your container registry and the name of your container image. Your container image is prepopulated from the module project name value. Replace **localhost:5000** with the login server value from your Azure container registry. You can retrieve the login server from the Overview page of your container registry in the Azure portal. <br><br> The final image repository looks like \<registry name\>.azurecr.io/iotedgemodule1. |

   ![Configure your project for target device, module type, and container registry](./media/tutorial-develop-for-windows/add-application-and-module.png)

5. Select **OK** to apply your changes. 

Once your new project loads in the Visual Studio window, take a moment to familiarize yourself with the files that it created: 

* An IoT Edge project called **AzureIoTEdgeApp1.Windows.Amd64**.
    * The **Modules** folder contains pointers to the modules included in the project. In this case, it should be just IoTEdgeModule1. 
    * The **deployment.template.json** file is a template to help you create a deployment manifest. A *deployment manifest* is a file that defines exactly which modules you want deployed on a device, how they should be configured, and how they can communicate with each other and the cloud. 
* An IoT Edge module project called **IoTEdgeModule1**.
    * The **main.c** file contains the default C module code that comes with the project template. The default module takes input from a source and passes it along to IoT Hub. 
    * The **module.json** file hold details about the module, including the full image repository, image version, and which Dockerfile to use for each supported platform.

### Provide your registry credentials to the IoT Edge agent

The IoT Edge runtime needs your registry credentials to pull your container images onto the IoT Edge device. Add these credentials to the deployment template. 

1. Open the **deployment.template.json** file.

2. Find the **registryCredentials** property in the $edgeAgent desired properties. 

3. Update the property with your credentials, following this format: 

   ```json
   "registryCredentials": {
     "<registry name>": {
       "username": "<username>",
       "password": "<password>",
       "address": "<registry name>.azurecr.io"
     }
   }

4. Save the deployment.template.json file. 

### Review the sample code

The solution template that you created includes sample code for an IoT Edge module. This sample module simply receives messages and then passes them on. The pipeline functionality demonstrates an important concept in IoT Edge, which is how modules communicate with each other.

Each module can have multiple *input* and *output* queues declared in their code. The IoT Edge hub running on the device routes messages from the output of one module into the input of one or more modules. The specific language for declaring inputs and outputs varies between languages, but the concept is the same across all modules. For more information about routing between modules, see [Declare routes](module-composition.md#declare-routes).

1. In the **main.c** file, find the **SetupCallbacksForModule** function.

2. This function sets up an input queue to receive incoming messages. It calls the C SDK module client function [SetInputMessageCallback](https://docs.microsoft.com/azure/iot-hub/iot-c-sdk-ref/iothub-module-client-ll-h/iothubmoduleclient-ll-setinputmessagecallback). Review this function and see that it initializes an input queue called **input1**. 

   ![Find the input name in the SetInputMessageCallback constructor](./media/tutorial-develop-for-windows/declare-input-queue.png)

3. Next, find the **InputQueue1Callback** function.

4. This function processes received messages and sets up an output queue to pass them along. It calls the C SDK module client function [SendEventToOutputAsync](https://docs.microsoft.com/azure/iot-hub/iot-c-sdk-ref/iothub-module-client-ll-h/iothubmoduleclient-ll-sendeventtooutputasync). Review this function and see that it initializes an output queue called **output1**. 

   ![Find the output name in the SendEventToOutputAsync constructor](./media/tutorial-develop-for-windows/declare-output-queue.png)

5. Open the **deployment.template.json** file.

6. Find the **modules** property of the $edgeAgent desired properties. 

   There should be two modules listed here. The first is **tempSensor**, which is included in all the templates by default to provide simulated temperature data that you can use to test your modules. The second is the **IotEdgeModule1** module that you created as part of this project.

   This modules property declares which modules should be included in the deployment to your device or devices. 

7. Find the **routes** property of the $edgeHub desired properties. 

   One of the functions if the IoT Edge hub module is to route messages between all the modules in a deployment. Review the values in the routes property. The first route, **IotEdgeModule1ToIoTHub**, uses a wildcard character (**\***) to include any message coming from any output queue in the IoTEdgeModule1 module. These messages go into *$upstream*, which is a reserved name that indicates IoT Hub. The second route, **sensorToIotEdgeModule1**, takes messages coming from the tempSensor module and routes them to the *input1* input queue of the IotEdgeModule1 module. 

   ![Review routes in deployment.template.json](./media/tutorial-develop-for-windows/deployment-routes.png)


## Build and push your solution

You've reviewed the module code and the deployment template to understand some key deployment concepts. Now, you're ready to build the IotEdgeModule1 container image and push it to your container registry. With the IoT tools extension for Visual Studio, this step also generates the deployment manifest based on the information in the template file and the module information from the solution files. 

### Sign in to Docker

Provide your container registry credentials to Docker on your development machine so that it can push your container image to be stored in the registry. 

1. Open PowerShell or a command prompt.

2. Sign in to Docker with the Azure container registry credentials that you saved after creating the registry. 

   ```cmd
   docker login -u <ACR username> -p <ACR password> <ACR login server>
   ```

   You may receive a security warning recommending the use of `--password-stdin`. While that best practice is recommended for production scenarios, it's outside the scope of this tutorial. For more information, see the [docker login](https://docs.docker.com/engine/reference/commandline/login/#provide-a-password-using-stdin) reference.

### Build and push

Your development machine now has access to your container registry, and your IoT Edge devices will too. It's time to turn the project code into a container image. 

1. Right-click the **AzureIotEdgeApp1.Windows.Amd64** project folder and select **Build and Push IoT Edge Modules**. 

   ![Build and push IoT Edge modules](./media/tutorial-develop-for-windows/build-and-push-modules.png)

   The build and push command starts three operations. First, it creates a new folder in the solution called **config** that holds the full deployment manifest, built out of information in the deployment template and other solution files. Second, it runs `docker build` to build the container image based on the appropriate dockerfile for your target architecture. Then, it runs `docker push` to push the image repository to your container registry. 

   This process may take several minutes the first time, but is faster the next time that you run the commands. 

2. Open the **deployment.windows-amd64.json** file in the newly created config folder. (The config folder may not appear in the Solution Explorer in Visual Studio. If that's the case, select the **Show all files** icon in the Solution Explorer taskbar.)

3. Find the **image** parameter of the IotEdgeModule1 section. Notice that the image contains the full image repository with the name, version, and architecture tag from the module.json file.

4. Open the **module.json** file in the IotEdgeModule1 folder. 

5. Change the version number for the module image. (The version, not the $schema-version.) For example, increment the patch version number to **0.0.2** as though we had made a small fix in the module code. 

   >[!TIP]
   >Module versions enable version control, and allow you to test changes on a small set of devices before deploying updates to production. If you don't increment the module version before building and pushing, then you overwrite the repository in your container registry. 

6. Save your changes to the module.json file.

7. Right-click the **AzureIotEdgeApp1.Windows.Amd64** project folder again, and again select **Build and Push IoT Edge modules**. 

8. Open the **deployment.windows-amd64.json** file again. Notice that a new file wasn't created when you ran the build and push command again. Rather, the same file was updated to reflect the changes. The IotEdgeModule1 image now points to the 0.0.2 version of the container. This change in the deployment manifest is how you tell the IoT Edge device that there's a new version of a module to pull. 

9. To further verify what the build and push command did, go to the [Azure portal](https://portal.azure.com) and navigate to your container registry. 

10. In your container registry, select **Repositories** then **iotedgemodule1**. Verify that both versions of the image were pushed to the registry.

    ![View both image versions in container registry](./media/tutorial-develop-for-windows/view-repository-versions.png)

### Troubleshoot

If you encounter errors when building and pushing your module image, it often has to do with Docker configuration on your development machine. Use the following checks to review your configuration: 

* Did you run the `docker login` command using the credentials that you copied from your container registry? These credentials are different than the ones that you use to sign in to Azure. 
* Is your container repository correct? Does it have your correct container registry name and your correct module name? Open the **module.json** file in the IotEdgeModule1 folder to check. The repository value should look like **\<registry name\>.azurecr.io/iotedgemodule1**. 
* If you used a different name than **IotEdgeModule1** for your module, is that name consistent throughout the solution?
* Is your machine running the same type of containers that you're building? This tutorial is for Windows IoT Edge devices, so your Visual Studio files should have the **windows-amd64** extension, and Docker Desktop should be running Windows containers. 

## Deploy modules to device

You verified that the built container images are stored in your container registry, so it's time to deploy them to a device. Make sure that your IoT Edge device is up and running. 

1. Open the Cloud Explorer in Visual Studio and expand the details for your IoT hub. 

2. Select the name of the device that you want to deploy to. In the **Actions** list, select **Create Deployment**.

   ![Create deployment for single device](./media/tutorial-develop-for-windows/create-deployment.png)


3. In the file explorer, navigate to the config folder of your project and select the **deployment.windows-amd64.json** file. This file is often located at `C:\Users\<username>\source\repos\AzureIotEdgeApp1\AzureIotEdgeApp1.Windows.Amd64\config\deployment.windows-amd64.json`

   Do not use the deployment.template.json file, which doesn't have the full module image values in it. 

4. Expand the details for your IoT Edge device in the Cloud Explorer to see the modules on your device.

5. Use the **Refresh** button to update the device status to see that the tempSensor and IotEdgeModule1 modules are deployed your device. 


   ![View modules running on your IoT Edge device](./media/tutorial-develop-for-windows/view-running-modules.png)

## View messages from device

The IotEdgeModule1 code receives messages through its input queue and passes them along through its output queue. The deployment manifest declared routes that passed messages from tempSensor to IotEdgeModule1, and then forwarded messages from IotEdgeModule1 to IoT Hub. The Azure IoT Edge tools for Visual Studio allow you to see messages as they arrive at IoT Hub from your individual devices. 

1. In the Visual Studio cloud explorer, select the name of the IoT Edge device that you deployed to. 

2. In the **Actions** menu, select **Start Monitoring D2C Message**.

3. Watch the **Output** section in Visual Studio to see messages arriving at your IoT hub. 

   It may take a few minutes for both modules to start. The IoT Edge runtime needs to receive its new deployment manifest, pull down the module images from the container runtime, then start each new module. If you 

   ![View incoming device to cloud messages](./media/tutorial-develop-for-windows/view-d2c-messages.png)

## View changes on device

If you want to see what's happening on your device itself, use the commands in this section to inspect the IoT Edge runtime and modules running on your device. 

The commands in this section are for your IoT Edge device, not your development machine. If you're using a virtual machine for your IoT Edge device, connect to it now. In Azure, go to the virtual machine's overview page and select **Connect** to access the remote desktop connection. On the device, open a command or PowerShell window to run the `iotedge` commands.

* View all modules deployed to your device, and check their status:

   ```cmd
   iotedge list
   ```

   You should see four modules: the two IoT Edge runtime modules, tempSensor, and IotEdgeModule1. All four should be listed as running.

* Inspect the logs for a specific module:

   ```cmd
   iotedge logs <module name>
   ```

   IoT Edge modules are case-sensitive. 

   The tempSensor and IotEdgeModule1 logs should show the messages they're processing. The edgeAgent module is responsible for starting the other modules, so its logs will have information about implementing the deployment manifest. If any module isn't listed or isn't running, the edgeAgent logs will probably have the errors. The edgeHub module is responsible for communications between the modules and IoT Hub. If the modules are up and running, but the messages aren't arriving at your IoT hub, the edgeHub logs will probably have the errors. 

## Next steps

In this tutorial, you set up Visual Studio 2017 on your development machine and deployed your first IoT Edge module from it. Now that you know the basic concepts, try adding functionality to a module so that it can analyze the data passing through it. Choose your preferred language: 

> [!div class="nextstepaction"] 
> [C](tutorial-c-module-windows.md)
> [C#](tutorial-csharp-module-windows.md)