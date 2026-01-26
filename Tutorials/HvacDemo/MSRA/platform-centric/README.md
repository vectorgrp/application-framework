# HVAC demo, platform-centric scenario based on MSRA

The platform-centric use case describes a project setup, where some platform model is already
available and supposed to be used as starting point for the development of the application in
question. In this demo, it is an AUTOSAR Adaptive model that consists of a design and a deployment
part. The **design part** covers the definition of two service interfaces as follows:

| Service: HvacStatus     | Service: HvacControl      |
| ----------------------- | ------------------------- |
| Event: CompressorStatus | Event: CompressorState    |
| Event: ValveStatus      | Event: ValvePosition      |
| Event: FanRightSpeed    | Event: FanSpeed           |
| Event: FanLeftSpeed     | Method: ChangeTemperature |
| Method: SetDegree       |                           |

The **deployment part** defines one executable with a consumer port (r-port) of the HvacStatus interface
and a provider port (p-port) of the HvacControl interface:

* Executable: HvacExecutable
* RPort: HvacStatus
* PPort: HvacControl

This results in an initial situation based on the AUTOSAR Adaptive model as illustrated below:

![hvac_start](../../figures/hvac-start.svg)

## HvacExecutable (app)

As mentioned above, the executable and its external connections are known. The internal architecture
of the executable, however, is open and in the scope of the **Vector Application Framework (VAF)**.
Plan in this demo is to introduce two application modules. Each is supposed to connect one of the
external ports. Besides that, an internal communication channel between the modules is wanted. A
high-level illustration of this setup is given below:

![hvac](../../figures/hvac.svg)

### Import and definition of interfaces

At first, internal and external interfaces for the application modules need to be imported to the
VAF. For that, a **VAF interface project** can be used.

To get started, create a new project using the VAF command line tool:

``` bash
vaf project init interface
Enter your project name: Interfaces
```

Next, enter the project directory.

``` bash
cd Interfaces
```

The example AUTOSAR model is shipped as part of the container and located in: `/opt/vaf/Demo/HvacDemo/MSRA/platform-centric/app`.
Import the HVAC platform model by typing the following VAF command to a terminal:

``` bash
vaf platform derive msra
Enter the path to the ARXML model directory: /opt/vaf/Demo/HvacDemo/MSRA/platform-centric/app/model/msra
```

Two new files are added to the project folder by this command. `msra-derived-model.json` is the VAF
model file in JSON format. It contains all relevant information as imported from the original ARXML
files. `msra.py` is the Configuration as Code (CaC) support, which is needed to access the model
artifacts from the Python configuration.

Both, consolidation of external model information and definition of own datatypes and interfaces is
done in Python. For that, open the template file `interface.py`. To import the interfaces from the
previous platform derive step, uncomment line #4:

``` python
from .msra import *
```

The definition of own datatypes and interfaces is also done in the `interface.py` file by using elements
from the `vafpy` package. For the HVAC example, one extra interface for the direct data exchange of
the two app-modules is needed. The below interface definition contains one example for pub/sub
messaging using data elements and one example for a remote procedure call, referred to as operation:

``` python
my_interface = vafpy.ModuleInterface(name="DataExchangeInterface", namespace="demo")
my_interface.add_data_element(name="MyValue", datatype=BaseTypes.UINT32_T)
my_interface.add_operation(name="MyFunction", in_parameter={"MyValueIn": BaseTypes.UINT32_T}, out_parameter={"MyValueOut": BaseTypes.UINT32_T})
```

> **ℹ️ Note**  
> The Python IntelliSense feature in VS Code is extremely helpful when working with such
> configuration files.

Last step in the interface project is the export of the configuration to the VAF exchange format in
JSON. To do so, use the following command:

``` bash
vaf model generate
```

The exported JSON file gets stored to the subdirectory `./export` by default, along with its CaC
support file for later use in an application module project.

### Configuration and implementation of app-modules

Application modules are supposed to be self-contained. The corresponding **VAF app-module project**
allows to configure, implement, test, and maintain it stand-alone and thus separate from the later
integration step. This further allows to use app-modules in different integration projects.

To start a new application module project, type:

``` bash
vaf project init app-module
Enter the name of the app-module: AppModule1
Enter the namespace of the app-module: demo

cd AppModule1
```

In first place, the above-created data exchange file from the interface project needs to be imported
to make the model elements from there accessible in the app-module project. Use the following
command for that:

```bash
vaf project import
Enter the path to the exported VAF model JSON file: ../Interfaces/export/Interfaces.json
```

Next step is the configuration of the application module. For that, open the file
`./model/app_module1.py`. To complete the import from the interface project, uncomment line #6:

``` python
from .imported_models import *
```

The configuration of the application module is done completely in this Configuration as Code file.
According to the illustration above, `app_module1` is supposed to connected to the `HvacStatus` r-port
and therefore needs a corresponding consumer interface. For communication with `app_module2` it acts
as provider of the earlier defined `DataExchangeInterface`. The `app_module2` acts as consumer
counterpart for the `DataExchangeInterface` and towards the platform-side, needs a provider
interface for `HvacControl`.

The configuration for `app_module1`:
Extend the existing app_module1 with consumed and provided interfaces.

``` python
app_module1 = vafpy.ApplicationModule(name="AppModule1", namespace="demo")

app_module1.add_consumed_interface(instance_name="HvacStatusConsumer", interface=interfaces.Nsprototype.Nsserviceinterface.Nshvacstatus.hvac_status)
app_module1.add_provided_interface(instance_name="DataExchangeProvider", interface=interfaces.Demo.data_exchange_interface)

periodic_task = vafpy.Task(name="PeriodicTask", period=timedelta(milliseconds=200))
app_module1.add_task(task=periodic_task)
```

Once complete, the next step is to export the configuration to JSON and code generation.
The resulting file from the export (`./model/model.json`) contains all information that is needed for the code generation step.

``` bash
vaf project generate
```

The generated code can be divided into read-write and read-only parts. Former gets generated to the
`./implementation` folder. This is user space, where the framework only provides implementation
stubs for the developer to start. In case of re-generation, a 3-way merge strategy based on git-merge
is applied to the files in this location. The read-only parts get generated to `./src-gen` and
`./test-gen`. Those folders are under control of the framework. Any user modification will be
overwritten in case of re-generation.

The entry file for the user to add own code is located in `./implementation/src/app_module1.cpp`.
Some sample code for reference is shipped as part of the container and located in: `/opt/vaf/Demo/HvacDemo/MSRA/platform-centric/app`.

The application module project is also ready to be built as library. To do so and in order to check
if added code passes the compiler checks, execute the following two steps. First, start preset to
prepare the build step, which includes CMake preset and Conan cache setup. Second, trigger the
CMake-based build process. On the command line:

``` bash
vaf make preset
vaf make build
```

Once done with `AppModule1`, the same procedure can be repeated for `AppModule2`. Starting with the
creation of a new app-module project.

The configuration for `app_module2`:
Extend the existing app_module2 with consumed and provided interfaces.

``` python
app_module2 = vafpy.ApplicationModule(name="AppModule2", namespace="demo")

app_module2.add_consumed_interface(instance_name="DataExchangeConsumer", interface=interfaces.Demo.data_exchange_interface)
app_module2.add_provided_interface(instance_name="HvacControlProvider", interface=interfaces.Nsprototype.Nsserviceinterface.Nshvaccontrol.hvac_control)

periodic_task = vafpy.Task(name="PeriodicTask", period=timedelta(milliseconds=200))
app_module2.add_task(task=periodic_task)
```

### Unit testing of app-modules

The Vector Application Framework provides means for unit testing. Test mocks for Googletest are
generated to `./test-gen` accordingly and allow independent first-level testing of application
modules. Custom test code can be added in the corresponding `tests.cpp` file in the
`./implementation/test/unittest` folder.

Some sample test code for the app-modules is provided for reference in:
`/opt/vaf/Demo/HvacDemo/MSRA/platform-centric/app/src`.

Please note. The build step of these unit tests is enabled by default. To deactivate it, use the
following commands:

``` bash
vaf make preset -d -DVAF_BUILD_TESTS=OFF
vaf make build
```

The resulting test binaries get stored in `build/bin` for execution.

### Executable integration

Final integration of all application modules is done using a **VAF integration project**. This is
where the whole application, which potentially consists of multiple executables, gets integrated. In
practice, app-modules and platform modules, as provided by the framework, get instantiated and
wired. The complete picture of the HvacExecutable is illustrated below.

![hvac_detailed](../../figures/hvac-detailed.svg)

To start a new integration project, execute the following steps:

``` bash
vaf project init integration
Enter your project name: HvacExecutable

cd HvacExecutable
```

To get started, all relevant application-module projects need to be imported in first place:

```bash
vaf project import
Enter the path to the application module project to be imported: ../AppModule1/

vaf project import
Enter the path to the application module project to be imported: ../AppModule2/
```

The import command adds new files to `./model/vaf/application_modules`. This includes relevant path
information but, most and foremost, the importer and CaC-support artifacts, which make the
model elements from the app-module accessible for the configuration in the integration project.

In this example, the executable is already defined as part of the AUTOSAR model. To import this
deployment information to the project and also to execute the code generators of MICROSAR Adaptive
(MSRA), use the following command:

``` bash
vaf platform derive msra
Enter the path to the ARXML model directory: /opt/vaf/Demo/HvacDemo/MSRA/platform-centric/app/model/msra
```

The generated code artifacts for MSRA go into the read-only directory `./msra-gen`.

Next step is the configuration of the integration project in `./model/vaf/hvac_executable.py`.

To import the platform artifacts from MSRA, uncomment line #7 in the template:

``` python
from .msra import *
```

> **ℹ️ Note**  
> Checking the file `msra.py` or one of the import files for the app-modules can be helpful to
> understand the class-tree structure, which is behind the VAF Configuration as Code solution.

The executable is already defined in `msra.py:45`. Uncommenting line #10 is not required.
Means, we can directly continue with the step of instantiating and adding app-modules as follows:

``` python
HvacExecutable.executable.add_application_module(AppModule1, [])
HvacExecutable.executable.add_application_module(AppModule2, [])
```

The second parameter, here set as empty list, allows the definition of an integration-specific task
mapping. This includes the task period and execution order and allows the integrator to overrule the
original settings from the application module project.

The two app-modules instances now can be connected. Among each other, on the one hand, and with
platform modules, on the other hand. The below configuration code snippet details the part
for executable-internal communication in this example project:

``` python
HvacExecutable.executable.connect_interfaces(AppModule1, Instances.AppModule1.ProvidedInterfaces.DataExchangeProvider,
                                             AppModule2, Instances.AppModule2.ConsumedInterfaces.DataExchangeConsumer)
```

The communication with a lower-layer platform is abstracted by platform modules. They deal with the
platform API towards the lower layer, i.e. the middleware stack, and with the VAF API towards the
upper, the application layer. The connection between application and platform modules is configured
as follows:

``` python
HvacExecutable.executable.connect_consumed_interface_to_platform(AppModule1, Instances.AppModule1.ConsumedInterfaces.HvacStatusConsumer,
                                                               HvacExecutable.ConsumerModules.hvac_status)
HvacExecutable.executable.connect_provided_interface_to_platform(AppModule2, Instances.AppModule2.ProvidedInterfaces.HvacControlProvider,
                                                               HvacExecutable.ProviderModules.hvac_control)
```

With this step done, the HVAC demo configuration part is complete. The next step is model and code
generation:

``` bash
vaf project generate
```

Using `--mode prj` or `--mode all` further allows to set the scope of this command to either, the
current integration project only (prj), or to this and all related sub-projects (all).

To complete the integration project, build preset, build, and final installation are missing:

``` bash
vaf make preset
vaf make build
vaf make install
```

The final (MSRA-compatible) executable for execution is stored in `./build/Release/install/opt`.

## HvacPlatform (test-app)

In order to run the HvacExecutable application, a counterpart is required that mimics the platform
side and consumes/provides the necessary services as expected from there. This counterpart can be
built with the same workflow as described above for the HvacExecutable.

>**ℹ️ Note**  
> The HvacPlatform executable in the example contains one app-module only (AppModule3). In this
> case, the interface project can be skipped. Same holds for importing the app-module project. It
> can also be created directly within the integration project. To do so, use `vaf project create
> app-module`. Instead of being linked to the integration project, the app-module will live as local
> copy in `./src/application_modules`.

The input AUTOSAR model, on the one hand, and VAF configuration and source code samples for
reference, on the other hand, are provided in:
`/opt/vaf/Demo/HvacDemo/MSRA/platform-centric/test-app`.

## Running the HVAC application

Open three terminal sessions to get started. One for the HvacExecutable process (app), one for the
counterpart (test-app), and one for the IPC service discovery daemon of MSRA.

>**ℹ️ Note**  
> It is important to run all sessions (processes) on the same machine or within the same container.
> The compiled applications have no dependencies on the devcontainer and can be run on your host
> machine. Running inside a container is discouraged, as the container requires special network
> configuration/permissions to set up the loopback interface.

In any case, configure the environment per terminal session using the `setup.sh` script. It sets the
necessary environment variables.

``` bash
source setup.sh
```

In the first terminal, start the IPC service discovery daemon located in:
`./build/Release/install/opt/amsr_ipcservicediscovery_daemon`. Then launch both executables, located in the
corresponding `./build/Release/install/opt` directories of your `app` and `test-app` projects.
