# Hello VAF example (extended)

This extended example of the Hello VAF example illustrates the available communication bindings 
of the MICROSAR Adaptive Communication Platform (MSRA). The example covers the full journey, including
configuration, implementation, compilation, and execution. It shows, in a nutshell, how application
logic (in form of application modules) can be combined with different communication logic
(represented by platform modules).


## Expected result
The result of this demo is a fully configured integration project, which is created using the
Vector Application Framework. The resulting application consists of two parts, each represented by
one binary executable. That is, on the one hand, the *DemoExecutable* and, on the other hand, the
*TestExecutable. They communicate with each other using three different types of communication
mechanisms as supported by the MICROSAR Adaptive Communication Platform. That is: IPC, SOME/IP, and
Zero Copy (ZC). Executable-internal communication is also used but directly handled by the
Application Framework. Means, this communication channel does not depend from the platform.
Eventually, communication daemons (as provided by MSRA) and the example application get executed and
will log some information about the exchanged data to the standard output of the terminal window.


## Overview
The application logic in this project is encapsulated by three application modules. During
integration, instances of these modules get assigned to executables. AppModule1 and AppModule2 to
DemoExecutable; AppModule3 to TestExecutable.

| Application Modules	|  
| --------------------- |  
| AppModule1	        |  
| AppModule2	        |  
| AppModule3	        |  

| Executables             |
| ----------------------- |
| DemoApplication         |
| TestApplication	      |

The data exchange between the listed application modules is realized in various ways. Each
mechanism, as listed below, addresses a dedicated communication case.

| Communication Mechanisms                                                          | 
| --------------------------------------------------------------------------------- |
| IPC (Intra ECU, MICROSAR Adaptive)                                                |
| Zero Copy (Intra ECU, MICROSAR Adaptive)                                          |
| SOME/IP (Inter ECU, MICROSAR Adaptive)                                            |
| Internal communication channel (Intra executable, Vector Application Framework) |


>**ℹ️ Note** This demo is just a showcase that illustrates the functionality and features of
> Vector Application Framework and MICROSAR Adaptive. The demonstrated usage of IPC, Zero Copy,
> and SOME/IP for communication between executables that get executed on the same machine, for
> example, does not depict reality. For this case, one would rather go with only IPC or Zero Copy
> only, which is tailored for local communication on the same ECU. Network communication between
> ECUs instead is a use case for SOME/IP protocol.

The figure below illustrates the building blocks and communication relations of this demo once again.
* Communication within the same executable via Vector Application Framework (cf. AppModule1 ↔ AppModule2).
* Communication between different executables via MSRA communication mechanisms (cf. AppModule1 ↔
  AppModule3, AppModule2 ↔ AppModule3).

![hello-vaf-extended](../figures/hello-extended.svg)


## Step-by-step instructions
Before you start with this Hello VAF Extended demo project, make sure to meet the necessary
[Prerequisites](https://github.com/vectorgrp/application-framework/blob/master/README.md#prerequisites)
as listed in the top-level README of this repository.

This example is structured into multiple steps as follows:
* Step 1: [Set up project](#step-1-set-up-project)
* Step 2: [Create application modules](#step-2-create-application-modules)
* Step 3: [Configure communication between application modules](#step-3-configure-communication-between-application-modules)
* Step 4: [Generate code for application modules](#step-4-generate-code-for-application-modules)
* Step 5: [Implement logic of application modules](#step-5-implement-logic-of-application-modules)
* Step 6: [Create and configure application executables](#step-6-create-and-configure-application-executables)
* Step 7: [Connect application modules](#step-7-connect-application-modules)
* Step 8: [Generate source code for the integration project](#step-8-generate-source-code-for-the-integration-project)
* Step 9: [Generate platform-specific source code](#step-9-generate-platform-specific-source-code)
* Step 10: [Build executables](#step-10-build-executables)
* Step 11: [Run executables](#step-11-run-executables)


>**ℹ️ Note** There are two options on how to follow this step-by-step guideline. You can either use
> the project and task navigation bar as provided by the Visual Studio Code (VS Code) extension
> *Vector Application Framework (VAF)* or directly use the CLI commands via terminal window. In this
> README, the CLI commands are used and described. To run tasks via the extension, select the
> project in the VAF Navigation area and run a task from the VAF Tasks view by clicking the
> respective item.

### Step 1: Set up project 
To get started, you need to create a so-called integration project:
1. Do so by using the following CLI command:
```bash
vaf project init integration
```

2. Choose a suitable name, e.g., HelloVafExtended:
```bash
Enter your project name: HelloVafExtended
```

3. Choose a suitable directory to store your project in. Click `Enter` to select the current directory.

The Application Framework CLI tool created your integration project, which provides the general
structure for this sample application.

### Step 2: Create application modules 
Before specifying the executables of this demo as illustrated above, you first need to create
application module projects. Existing app-modules can also be imported. Here, you will
create application module projects in place:
1. Navigate to your newly created integration project in your terminal.
```bash
cd HelloVafExtended
```

2. Create the application module AppModule1 by using the following CLI command:
```bash
vaf project create app-module
```

3. Choose a suitable name for the application module AppModule1:
```bash
Enter the name of the app-module: AppModule1
```

4. Choose a namespace for the application module:
```bash
Enter the namespace of the app-module: demo
```

5. Choose the default path to the project root directory, i.e., the directory to store your
   application module project:
```bash
Enter the path to the project root directory: .
```

6. Repeat steps 2-5 to create the application modules AppModule2 and AppModule3.

The CLI tool created AppModule1, AppModule2, and AppModule3 as separate folders, which can be found
in the `src/application_modules` directory of your integration project.

### Step 3: Configure communication between application modules 
To specify the communication between the newly created application modules, you need to create
communication interfaces for your application modules. This can be done in the configuration files
(.py) of the respective application modules. This file is also referred to as Configuration as Code
(CaC) file as the configuration is noted as a Python script.

#### Configure AppModule1 
1. Open the `src/application_modules/app_module1/model/app_module1.py` configuration file of the
   created AppModule1 inside your integration project.

2. Implement interfaces and tasks for your application module in your configuration file using the
   Application Framework model API by extending the configuration file with the following lines of
   code:

For `src/application_modules/app_module1/model/app_module1.py`:
``` python
app_module1 = vafpy.ApplicationModule(name="AppModule1", namespace="demo")

ic_data_string = vafpy.ModuleInterface(name="ICDataInterfaceSTRING", namespace="ic_string")
ic_data_string.add_data_element("ICDataSTRING", datatype=BaseTypes.STRING)
app_module1.add_provided_interface("AppProviderIC", ic_data_string)

zc_data_uint8 = vafpy.ModuleInterface(name="ZCDataInterfaceUINT8", namespace="zc_uint8")
zc_data_uint8.add_data_element("ZCDataUINT8", datatype=BaseTypes.UINT8_T)
app_module1.add_provided_interface("MSRAProviderZC", zc_data_uint8)

ipc_data_uint8 = vafpy.ModuleInterface(name="IPCDataInterfaceUINT8", namespace="ipc_uint8")
ipc_data_uint8.add_data_element("IPCDataUINT8", datatype=BaseTypes.UINT8_T)
app_module1.add_provided_interface("MSRAProviderIPC", ipc_data_uint8)

someip_data_uint8 = vafpy.ModuleInterface(name="SOMEIPDataInterfaceUINT8", namespace="someip_uint8")
someip_data_uint8.add_data_element("SOMEIPDataUINT8", datatype=BaseTypes.UINT8_T)
app_module1.add_provided_interface("MSRAProviderSOMEIP", someip_data_uint8)

periodic_task = vafpy.Task(name="PeriodicTask", period=timedelta(milliseconds=200))
app_module1.add_task(task=periodic_task)
```

3. Save and exit the configuration file.

This step complete means you have set up the communication interfaces for AppModule1.

#### Configure AppModule2
1. Open the `src/application_modules/app_module2/model/app_module2.py` configuration file of the
   created AppModule2 inside your integration project.

2. Implement interfaces and tasks for your application module in your configuration file using the
   Application Framework model API by extending the configuration file with the following lines of
   code:

For `app-module2.py`:
``` python
app_module2 = vafpy.ApplicationModule(name="AppModule2", namespace="demo")

ic_data_string = vafpy.ModuleInterface(name="ICDataInterfaceSTRING", namespace="ic_string")
ic_data_string.add_data_element("ICDataSTRING", datatype=BaseTypes.STRING)
app_module2.add_consumed_interface("AppConsumerIC", ic_data_string)

zc_data_int8 = vafpy.ModuleInterface(name="ZCDataInterfaceINT8", namespace="zc_int8")
zc_data_int8.add_data_element("ZCDataINT8", datatype=BaseTypes.INT8_T)
app_module2.add_consumed_interface("MSRAConsumerZC", zc_data_int8)

ipc_data_int8 = vafpy.ModuleInterface(name="IPCDataInterfaceINT8", namespace="ipc_int8")
ipc_data_int8.add_data_element("IPCDataINT8", datatype=BaseTypes.INT8_T)
app_module2.add_consumed_interface("MSRAConsumerIPC", ipc_data_int8)

someip_data_int8 = vafpy.ModuleInterface(name="SOMEIPDataInterfaceINT8", namespace="someip_int8")
someip_data_int8.add_data_element("SOMEIPDataINT8", datatype=BaseTypes.INT8_T)
app_module2.add_consumed_interface("MSRAConsumerSOMEIP", someip_data_int8)

periodic_task = vafpy.Task(name="PeriodicTask", period=timedelta(milliseconds=200))
app_module2.add_task(task=periodic_task)
```

3. Save and exit the configuration file.

You have now set up the interfaces for communication with AppModule2.

#### Configure AppModule3
1. Open the `src/application_modules/app_module3/model/app_module3.py` configuration file of the
   created AppModule3 inside your integration project.

2. Implement interfaces and tasks for your application module in your configuration file using the
   Application Framework model API by extending the configuration file with the following lines of
   code:

For `app-module3.py`:
``` python
app_module3 = vafpy.ApplicationModule(name="AppModule3", namespace="demo")

zc_data_uint8 = vafpy.ModuleInterface(name="ZCDataInterfaceUINT8", namespace="zc_uint8")
zc_data_uint8.add_data_element("ZCDataUINT8", datatype=BaseTypes.UINT8_T)
app_module3.add_consumed_interface("MSRAConsumerZC", zc_data_uint8)

ipc_data_uint8 = vafpy.ModuleInterface(name="IPCDataInterfaceUINT8", namespace="ipc_uint8")
ipc_data_uint8.add_data_element("IPCDataUINT8", datatype=BaseTypes.UINT8_T)
app_module3.add_consumed_interface("MSRAConsumerIPC", ipc_data_uint8)

someip_data_uint8 = vafpy.ModuleInterface(name="SOMEIPDataInterfaceUINT8", namespace="someip_uint8")
someip_data_uint8.add_data_element("SOMEIPDataUINT8", datatype=BaseTypes.UINT8_T)
app_module3.add_consumed_interface("MSRAConsumerSOMEIP", someip_data_uint8)

zc_data_int8 = vafpy.ModuleInterface(name="ZCDataInterfaceINT8", namespace="zc_int8")
zc_data_int8.add_data_element("ZCDataINT8", datatype=BaseTypes.INT8_T)
app_module3.add_provided_interface("MSRAProviderZC", zc_data_int8)

ipc_data_int8 = vafpy.ModuleInterface(name="IPCDataInterfaceINT8", namespace="ipc_int8")
ipc_data_int8.add_data_element("IPCDataINT8", datatype=BaseTypes.INT8_T)
app_module3.add_provided_interface("MSRAProviderIPC", ipc_data_int8)

someip_data_int8 = vafpy.ModuleInterface(name="SOMEIPDataInterfaceINT8", namespace="someip_int8")
someip_data_int8.add_data_element("SOMEIPDataINT8", datatype=BaseTypes.INT8_T)
app_module3.add_provided_interface("MSRAProviderSOMEIP", someip_data_int8)

periodic_task = vafpy.Task(name="PeriodicTask", period=timedelta(milliseconds=200))
app_module3.add_task(task=periodic_task)
```

3. Save and exit the configuration file.

You now have completed the interfaces definitions for all involved app-modules.

### Step 4: Generate code for application modules
The code generation step for all application modules can now be triggered in order to:
* Generate the implementation stubs, i.e., the *.cpp and *.h files in the implementation subdirectory
  of the application module project.
* Configure the CMake build environment with a release and debug preset. This enables the use of the
  VS Code CMake Tools extension and IntelliSense features.

To trigger the code generation, repeat the following steps for each application module:
1. Navigate to the dedicated application module project inside the src/application_modules directory
   of your integration project in your terminal.
2. Run the following CLI command:
``` bash
vaf project generate
```

The Application Framework generators produce, amongst others, module interface files, source files,
core support files, and datatype header files.

### Step 5: Implement logic of application modules 
To realize the functionality of the described demo, some logic is needed for the periodic tasks and
event handlers of each application module:
1. We have prepared some sample code with basic functionality. It illustrates the usage of the API
and mimics some data exchange between the application modules. You can treat it as a starting
point for your own implementation. The files are located in the `/opt` directory of the Docker
container. Copy them to the workspace with the following command:
```bash 
cp -r /opt/vaf/Demo/HelloVaf/HelloExtended ./HelloVafExtendedReference
```

2. For each app-module in `src/application_modules` you can now either add your own implementation
   or use the prepared sample code from `HelloVafExtendedReference/app/src` and
   `HelloVafExtendedReference/test-app/src` by copying complete files or snippets to the respective
   source and header files.
3. Compilation can be triggered from within the app-module project folder. This step allows to check
   for implementation errors and yields a static library per application module.
```bash
vaf make build
```

4. If a compilation error occurs, fix it and rerun the previous step to confirm.
5. If compilation succeeds, apply the model changes from the application module to the integration
project. Navigate to the integration project root directory and run the command below:
```bash
vaf model update
```

Select the affected application module(s) from the list and continue such that the integration
project is updated accordingly.

### Step 6: Create and configure application executables
In this step, application modules get instantiated and assigned to executables. Also the
communication ports get mapped to each other.
1. Open the `model/hello_vaf_extended.py` configuration file inside your integration project.
2. Extend it with the following lines of code:
``` python
# --------------------------------------------------------------------------------------------------------------------------
#  Executable definitions
# --------------------------------------------------------------------------------------------------------------------------
executable1 = Executable("DemoExecutable")
executable2 = Executable("TestExecutable")

# --------------------------------------------------------------------------------------------------------------------------
#  Application module instantiations
# --------------------------------------------------------------------------------------------------------------------------
executable1.add_application_module(AppModule1, [])
executable1.add_application_module(AppModule2, [])

executable2.add_application_module(AppModule3, [])
```

That is the first part for the configuration in the integration project complete. 

### Step 7: Connect application modules 
To connect application modules, the created communication interface instances, often referred to as
ports, need to be connected. To do so, extend the file `model/vaf/hello_vaf_extended.py` with the
following lines of code:
```python
# --------------------------------------------------------------------------------------------------------------------------
#  Wiring of application and platform modules
# --------------------------------------------------------------------------------------------------------------------------
executable1.connect_interfaces(
    AppModule1,
    Instances.AppModule1.ProvidedInterfaces.AppProviderIC,
    AppModule2,
    Instances.AppModule2.ConsumedInterfaces.AppConsumerIC,
)

executable1.connect_provided_interface_to_msra(
    AppModule1,
    Instances.AppModule1.ProvidedInterfaces.MSRAProviderIPC,
    deployment_type=vafmodel.MSRAConnectionPointDeploymentType.IPC,
)
executable1.connect_provided_interface_to_msra(
    AppModule1,
    Instances.AppModule1.ProvidedInterfaces.MSRAProviderSOMEIP,
    deployment_type=vafmodel.MSRAConnectionPointDeploymentType.SOME_IP,
)
executable1.connect_provided_interface_to_msra(
    AppModule1,
    Instances.AppModule1.ProvidedInterfaces.MSRAProviderZC,
    deployment_type=vafmodel.MSRAConnectionPointDeploymentType.ZERO_COPY,
)

executable1.connect_consumed_interface_to_msra(
    AppModule2,
    Instances.AppModule2.ConsumedInterfaces.MSRAConsumerIPC,
    deployment_type=vafmodel.MSRAConnectionPointDeploymentType.IPC,
)
executable1.connect_consumed_interface_to_msra(
    AppModule2,
    Instances.AppModule2.ConsumedInterfaces.MSRAConsumerSOMEIP,
    deployment_type=vafmodel.MSRAConnectionPointDeploymentType.SOME_IP,
)
executable1.connect_consumed_interface_to_msra(
    AppModule2,
    Instances.AppModule2.ConsumedInterfaces.MSRAConsumerZC,
    deployment_type=vafmodel.MSRAConnectionPointDeploymentType.ZERO_COPY,
)

executable2.connect_consumed_interface_to_msra(
    AppModule3,
    Instances.AppModule3.ConsumedInterfaces.MSRAConsumerIPC,
    deployment_type=vafmodel.MSRAConnectionPointDeploymentType.IPC,
)
executable2.connect_consumed_interface_to_msra(
    AppModule3,
    Instances.AppModule3.ConsumedInterfaces.MSRAConsumerSOMEIP,
    deployment_type=vafmodel.MSRAConnectionPointDeploymentType.SOME_IP,
)
executable2.connect_consumed_interface_to_msra(
    AppModule3,
    Instances.AppModule3.ConsumedInterfaces.MSRAConsumerZC,
    deployment_type=vafmodel.MSRAConnectionPointDeploymentType.ZERO_COPY,
)

executable2.connect_provided_interface_to_msra(
    AppModule3,
    Instances.AppModule3.ProvidedInterfaces.MSRAProviderIPC,
    deployment_type=vafmodel.MSRAConnectionPointDeploymentType.IPC,
)
executable2.connect_provided_interface_to_msra(
    AppModule3,
    Instances.AppModule3.ProvidedInterfaces.MSRAProviderSOMEIP,
    deployment_type=vafmodel.MSRAConnectionPointDeploymentType.SOME_IP,
)
executable2.connect_provided_interface_to_msra(
    AppModule3,
    Instances.AppModule3.ProvidedInterfaces.MSRAProviderZC,
    deployment_type=vafmodel.MSRAConnectionPointDeploymentType.ZERO_COPY,
)
```

This step completes the Configuration as Code part. Application modules are now connected and the
communication set up.

### Step 8: Generate source code for the integration project
To generate the source code artifacts for the integration project, follow the below steps: 
1. Navigate to your integration project root directory in a terminal window.
2. Run the following CLI command:
```bash
vaf project generate
```
3. Press `Enter` to generate with generation mode `PRJ`, i.e., for the integration project only.

> **ℹ️ Note** If something changes in a sub-project of the integration project, the entire
> integration project with all its dependencies can be regenerated using project generation mode
> `ALL`. This also includes the previously discussed `vaf model update` command.

### Step 9: Generate platform-specific source code
Now, only the platform-specific source files for the MICROSAR Adpative Communication Platform are missing. 
This step can be accomplished with the following commands:
1. Navigate to your integration project root directory in a terminal window.
2. Run the following CLI command:
```bash
vaf platform export msra -o ./model/msra
```

### Step 10: Build executables 
The executables as configured earlier need to be compiled and linked in order to obtain the binary
for execution on a Linux machine as follows:
1. Navigate to your integration project root directory in a terminal window.
2. Run the following CLI commands:
```bash
vaf make preset
vaf make build
vaf make install
```

If a compilation error occurs, fix it and repeat step 2 until it completes successful.

The executables (DemoExecutable and TestExecutable) get deployed along with the MSRA platform daemons to 
the `build/Release/install/opt` directory of the integration project.

### Step 11: Run executables
Finally, prepare and run your built executables to observe the communication between the implemented
application modules:
1. Navigate to your integration project root directory in a terminal window.
2. Source the setup.sh script in order to set necessary environment variables:
```bash
source setup.sh
```

3. Make sure to run this script in all the terminals that are used to run executables.

#### Run amsr_ipcservicediscovery_daemon 
1. Navigate to `build/Release/install/opt/amsr_ipcservicediscovery_daemon directory`, which contains
   the IPC Service Discovery Daemon executable.
2. Execute the `amsr_ipcservicedescovery_daemon` as follows:
```bash
./bin/amsr_ipcservicedescovery_daemon
```

#### Run amsr_someipd_daemon 
1. Navigate to the `build/Release/install/opt/amsr_someipd_daemon` directory, which contains the
   SOME/IP Daemon executable.
2. Execute the `amsr_someipd_daemon` binary with the specification of the configuration file path:
```bash
./bin/amsr_someipd_daemon -c ./etc/someipd_posix.json
```

#### Run DemoExecutable
1. Navigate to `build/Release/install/opt/DemoExecutable`.
2. Execute the included binary as follows:
```bash
./bin/DemoExecutable 
```

#### Run TestExecutable
1. Navigate to `build/Release/install/opt/TestExecutable`.
2. Execute the included binary as follows:
```bash
./bin/TestExecutable
```

Congratulations! You have successfully set up the extended example of Hello VAF. You should
now now be able to observe the communication between the implemented application modules.
