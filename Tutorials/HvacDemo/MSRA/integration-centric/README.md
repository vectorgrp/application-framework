# HVAC demo, integration-centric scenario based on MSRA

The integration-centric use case describes a project setup in which both platform and application
models are already available, slightly mismatched, and supposed to be integrated. 

In this demo, the platform is AUTOSAR Adaptive, and the following are the design and deployment
parts of the platform model. The **design part** covers the definition of two service interfaces as
follows:

| Service: HvacStatus     | Service: HvacControl      |
| ----------------------- | ------------------------- |
| Event: CompressorStatus | Event: CompressorState    |
| Event: ValveStatus      | Event: ValvePosition      |
| Event: FanRightSpeed    | Event: FanSpeed           |
| Event: FanLeftSpeed     | Method: ChangeTemperature |
| Method: SetDegree       |                           |

The **deployment part** defines one executable with a consumer port (rport) of the HvacStatus interface
and a provider port (pport) of the HvacControl interface:

* Executable: HvacApplication
* RPort: HvacStatus
* PPort: HvacControl

The Application also has some pre-defined interfaces similar to those explained above, but with
different data types. Hence, the interfaces defined by the platform and application do not match and
cannot be directly connected. This results in an initial situation based on the AUTOSAR Adaptive
model as illustrated below:

![hvac_start](../../figures/hvac-integration-start.svg)

## HvacApplication (app)

The plan in this demo is to introduce two application modules. The first application module acts as
the application which has defined interfaces, while the second one acts as the transformer that
helps connect the platform interfaces with the application interfaces. The transformer should
consume an AUTOSAR service via a consumer port, and provide the equivalent service to the
application via a provider port and vice versa. A high-level illustration of this setup is given
below:

![hvac](../../figures/hvac-integration.svg)

As mentioned above, the executable with application interfaces and its external connections are
known. The internal transformer architecture of the executable, however, is open and in the scope of
the **Vector Application Framework (VAF)**.

### Configuration and implementation of app-modules

Application modules are supposed to be self-contained. The corresponding **VAF app-module project**
allows to configure, implement, test, and maintain it stand-alone and thus separate from the later
integration step. This further allows the use of app-modules in different integration projects.

#### ApplicationAppModule
To start a new application module project, type:

``` bash
vaf project init app-module
Enter the name of the app-module: ApplicationAppModule
Enter the namespace of the app-module: demo

cd ApplicationAppModule
```

The next step is the configuration of the interface definition. For that, open the file
`application_app_module.py`.

The configuration for the ApplicationAppModule project:

```python
hvac_control_service = vafpy.ModuleInterface(name="HvacControl", namespace="demo")
hvac_control_service.add_data_element(name="CompressorState", datatype=BaseTypes.STRING)
hvac_control_service.add_data_element(name="ValvePosition", datatype=BaseTypes.STRING)
hvac_control_service.add_data_element(name="FanSpeed", datatype=BaseTypes.STRING)
hvac_control_service.add_operation(
    name="ChangeTemperature",
    in_parameter={"Value": BaseTypes.UINT8_T},
)

hvac_status_service = vafpy.ModuleInterface(name="HvacStatus", namespace="demo")
hvac_status_service.add_data_element(name="CompressorStatus", datatype=BaseTypes.STRING)
hvac_status_service.add_data_element(name="ValveStatus", datatype=BaseTypes.STRING)
hvac_status_service.add_data_element(name="FanRightSpeed", datatype=BaseTypes.STRING)
hvac_status_service.add_data_element(name="FanLeftSpeed", datatype=BaseTypes.STRING)
hvac_status_service.add_operation(
    name="SetDegree",
    in_parameter={"Value": BaseTypes.UINT8_T},
)

application_app_module = vafpy.ApplicationModule(
    name="ApplicationAppModule", namespace="demo"
)

application_app_module.add_consumed_interface(instance_name="AppHvacStatusConsumer", interface=hvac_status_service)
application_app_module.add_provided_interface(instance_name="AppHvacControlProvider", interface=hvac_control_service)

periodic_task = vafpy.Task(name="PeriodicTask", period=timedelta(milliseconds=200))
application_app_module.add_task(task=periodic_task)
```

#### TransformerAppModule
To start a new application module project, type:

``` bash
vaf project init app-module
Enter the name of the app-module: TransformerAppModule
Enter the namespace of the app-module: demo

cd TransformerAppModule
```

The example AUTOSAR model is shipped as part of the container and located in:
`/opt/vaf/Demo/HvacDemo/MSRA/integration-centric/app`. Import the HVAC application model by typing
the following VAF command to a terminal:

``` bash
vaf platform derive msra
Enter the path to the ARXML model directory: /opt/vaf/Demo/HvacDemo/MSRA/integration-centric/app/model/msra
```

Two new files are added to the project folder by this command. `msra-derived-model.json` is the VAF
model file in JSON format. It contains all relevant information as imported from the original ARXML
files. `msra.py` is the Configuration as Code (CaC) support, which is needed to access the model
artifacts from the Python configuration.

Both the consolidation of external model information and the definition of own data types and
interfaces are done in Python. For that, open the template file `transformer_app_module.py`. To
import the interfaces from the previous platform derive step, uncomment line:

``` python
from .msra import *
```

Next step is the add of the application interfaces. For that, open the file
`./model/transformer_app_module.py`. Add the interfaces:

``` python
hvac_control_service = vafpy.ModuleInterface(name="HvacControl", namespace="demo")
hvac_control_service.add_data_element(name="CompressorState", datatype=BaseTypes.STRING)
hvac_control_service.add_data_element(name="ValvePosition", datatype=BaseTypes.STRING)
hvac_control_service.add_data_element(name="FanSpeed", datatype=BaseTypes.STRING)
hvac_control_service.add_operation(
    name="ChangeTemperature",
    in_parameter={"Value": BaseTypes.UINT8_T},
)

hvac_status_service = vafpy.ModuleInterface(name="HvacStatus", namespace="demo")
hvac_status_service.add_data_element(name="CompressorStatus", datatype=BaseTypes.STRING)
hvac_status_service.add_data_element(name="ValveStatus", datatype=BaseTypes.STRING)
hvac_status_service.add_data_element(name="FanRightSpeed", datatype=BaseTypes.STRING)
hvac_status_service.add_data_element(name="FanLeftSpeed", datatype=BaseTypes.STRING)
hvac_status_service.add_operation(
    name="SetDegree",
    in_parameter={"Value": BaseTypes.UINT8_T},
)
```

The configuration of the application module is done completely in this Configuration as Code file.
Extend the existing transformer_app_module with consumed and provided interfaces.

``` python
transformer_app_module = vafpy.ApplicationModule(
    name="TransformerAppModule", namespace="demo"
)

transformer_app_module.add_consumed_interface(instance_name="PlatformHvacStatusConsumer", interface=Nsprototype.Nsserviceinterface.Nshvacstatus.hvac_status)
transformer_app_module.add_provided_interface(instance_name="PlatformHvacControlProvider", interface=Nsprototype.Nsserviceinterface.Nshvaccontrol.hvac_control)


transformer_app_module.add_provided_interface(instance_name="AppHvacStatusProvider", interface=hvac_status_service)
transformer_app_module.add_consumed_interface(instance_name="AppHvacControlConsumer", interface=hvac_control_service)

periodic_task = vafpy.Task(name="PeriodicTask", period=timedelta(milliseconds=200))
transformer_app_module.add_task(task=periodic_task)
```

> **ℹ️ Note**  
> Run the below commands for both application modules after configuration is complete.

Once complete, the next step is to export the configuration to JSON and code generation. The
resulting file from the export (`./model/model.json`) contains all information that is needed for
the code generation step.

``` bash
vaf project generate
```

The generated code can be divided into read-write and read-only parts. Former gets generated to the
`./implementation` folder. This is the user space, where the framework only provides implementation
stubs for the developer to start. In case of re-generation, a 3-way merge strategy based on
git-merge is applied to the files in this location. The read-only parts get generated to `./src-gen`
and `./test-gen`. Those folders are under the control of the framework. Any user modification will
be overwritten in case of regeneration.

The entry file for the user to add their own code is located in
`./implementation/src/<app_module>.cpp`. Some sample code for reference is shipped as part of the
container and located in: `/opt/vaf/Demo/HvacDemo/MSRA/integration-centric/app`.

The application module project is also ready to be built as a library. To do so and in order to
check if the added code passes the compiler checks, trigger the CMake-based build process. On the
command line:

``` bash
vaf make build
```

Once done with `TransformerAppModule`, the same procedure can be repeated for `ApplicationAppModule`.

### Unit testing of app-modules

The Vector Application Framework provides means for unit testing. Test mocks for Googletest are
generated to `./test-gen` accordingly and allow independent first-level testing of application
modules. Custom test code can be added in the corresponding `tests.cpp` file in the
`./implementation/test/unittest` folder.

Some sample test code for the app-modules is provided for reference in:
`/opt/vaf/Demo/HvacDemo/MSRA/integration-centric/app/src`.

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
wired. The complete picture of the HvacApplication is illustrated below.

![hvac_detailed](../../figures/hvac-integration-detailed.svg)

To start a new integration project, execute the following steps:

``` bash
vaf project init integration
Enter your project name: HvacApplication

cd HvacApplication
```

To get started, all relevant application-module projects need to be imported in first place:

```bash
vaf project import
Enter the path to the application module project to be imported: ../TransformerAppModule/

vaf project import
Enter the path to the application module project to be imported: ../ApplicationAppModule/
```

The import command adds new files to `./model/vaf/application_modules`. This includes the relevant
path information, but most and foremost, the importer and CaC-support artifacts, which make the
Model elements from the app-module are accessible for configuration in the integration project.

In this example, the executable is already defined as part of the AUTOSAR model. To import this
deployment information to the project and also to execute the code generators of MICROSAR Adaptive
(MSRA), Use the following command:

``` bash
vaf platform derive msra
Enter the path to the ARXML model directory: /opt/vaf/Demo/HvacDemo/MSRA/integration-centric/app/model/msra
```

The generated code artifacts for MSRA go into the read-only directory `./msra-gen`.

Next step is the configuration of the integration project in `./model/vaf/hvac_application.py`.

To import the platform artifacts from MSRA, uncomment line #7 in the template:

``` python
from .msra import *
```

> **ℹ️ Note**  
> Checking the file `msra.py` or one of the import files for the app-modules can be helpful to
> understand the class-tree structure, which is behind the VAF Configuration as Code solution.

The executable is already defined in `msra.py:45`. Uncommenting line #10 is not required. Means, we
can directly continue with the step of instantiating and adding app-modules as follows:

``` python
HvacApplication.executable.add_application_module(TransformerAppModule, [])
HvacApplication.executable.add_application_module(ApplicationAppModule, [])
```

The second parameter, here set as an empty list, allows the definition of an integration-specific
task mapping. This includes the task period and execution order and allows the integrator to
overrule the original settings from the application module project.

The two app-modules instances can now be connected. Among each other, on the one hand, and with
platform modules, on the other hand. The configuration code snippet below details the part for
executable-internal communication in this example project:

``` python
HvacApplication.executable.connect_interfaces(
    TransformerAppModule,
    Instances.TransformerAppModule.ProvidedInterfaces.AppHvacStatusProvider,
    ApplicationAppModule,
    Instances.ApplicationAppModule.ConsumedInterfaces.AppHvacStatusConsumer,
)

HvacApplication.executable.connect_interfaces(
    ApplicationAppModule,
    Instances.ApplicationAppModule.ProvidedInterfaces.AppHvacControlProvider,
    TransformerAppModule,
    Instances.TransformerAppModule.ConsumedInterfaces.AppHvacControlConsumer,
)
```

The communication with a lower-layer platform is abstracted by platform modules. They deal with the
platform API towards the lower layer, i.e. the middleware stack, and with the VAF API towards the
upper, the application layer. The connection between application and platform modules is configured
as follows:

``` python
HvacApplication.executable.connect_consumed_interface_to_platform(
    TransformerAppModule,
    Instances.TransformerAppModule.ConsumedInterfaces.PlatformHvacStatusConsumer,
    HvacApplication.ConsumerModules.hvac_status,
)
HvacApplication.executable.connect_provided_interface_to_platform(
    TransformerAppModule,
    Instances.TransformerAppModule.ProvidedInterfaces.PlatformHvacControlProvider,
    HvacApplication.ProviderModules.hvac_control,
)
```

With this step done, the HVAC application configuration part is complete. The next step is model and
code generation:

``` bash
vaf project generate
```

Using `--mode prj` or `--mode all` further allows to set the scope of this command to either, 
the current integration project only (prj), or to this and all related sub-projects (all).
--mode prj
To complete the integration project, build and final installation are missing:

``` bash
vaf make build
vaf make install
```

The final (MSRA-compatible) executable for execution is stored in `./build/Release/install/opt`.

## HvacPlatform (test-app)

In order to run the HvacApplication application, a counterpart is required that mimics the platform
side and consumes/provides the necessary services as expected from there. This counterpart can be
built with the same workflow as described above for the HvacApplication.

>**ℹ️ Note**  
> The HvacPlatform executable in the example contains one app-module only (PlatformAppModule). In
> this case, importing the app-module project can be skipped. It can also be created directly within
> the integration project. To do so, use `vaf project create app-module`. Instead of being linked to
> the integration project, the app-module will live as a local copy in `./src/application_modules`.

The input AUTOSAR model, on the one hand, and the VAF configuration and source code samples for references, 
on the other hand, are provided in: `/opt/vaf/Demo/HvacDemo/MSRA/integration-centric/test-app`.

## Running the HVAC application

Open three terminal sessions to get started. One for the HvacApplication process (app), one for the
counterpart (test-app), and one for the IPC service discovery daemon of MSRA.

>**ℹ️ Note**  
> It is important to run all sessions (processes) on the same machine or within the same container.
> The compiled applications have no dependencies on the devcontainer and can be run on your host
> machine. Running inside a container is discouraged, as the container requires special network
> configuration/permissions to set up the loopback interface.

In any case, configure the environment per terminal session using the `setup.sh` script. It sets the
necessary environment variables and configure the loopback device.

``` bash
source setup.sh
```

In the first terminal, start the IPC service discovery daemon located in:
`./build/Release/install/opt/amsr_ipcservicediscovery_daemon`. Then launch both executables, located in the
corresponding `./build/Release/install/opt` directories of your `app` and `test-app` projects.
