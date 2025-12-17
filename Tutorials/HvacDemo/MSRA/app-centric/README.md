# HVAC demo, app-centric scenario based on MSRA

The app-centric use case describes a project setup, where no platform model is available. Here, the
aimed target platform is AUTOSAR Adaptive. A suitable AUTOSAR model must therefore be created by the
framework during development.

## HvacExecutable (app)

Plan in this demo is to introduce two application modules. The first application module should
consume an AUTOSAR service via a consumer port, while the second should provide such a service via a
provider port. Besides that, an internal communication channel between the modules is wanted. A
high-level illustration of this setup is given below:

![hvac](../../figures/hvac.svg)

### Creating the interface definitions

As no interface definition is available for the app-centric use case, the first step is to create
one. The associated **VAF interface project** allows an interface definition to be configured and
maintained independently and therefore separately from a specific application module.

To start a new interface project, type:

``` bash
vaf project init interface
Enter the name of the project: Interfaces
Enter the directory to store your project in: .
```

Next, switch to the just created project directory.

``` bash
cd Interfaces
```

Next step is the configuration of the interface definition. For that, open the file
`interfaces.py`.

The configuration for the interface project:

```python
hvac_control_service = vafpy.ModuleInterface(name="HvacControl", namespace="nsprototype::nsserviceinterface::nshvaccontrol")
hvac_control_service.add_data_element(name="CompressorState", datatype=BaseTypes.UINT8_T)
hvac_control_service.add_data_element(name="ValvePosition", datatype=BaseTypes.UINT8_T)
hvac_control_service.add_data_element(name="FanSpeed", datatype=BaseTypes.UINT8_T)
hvac_control_service.add_operation(
    name="ChangeTemperature",
    in_parameter={"Value": BaseTypes.UINT8_T},
)

hvac_status_service = vafpy.ModuleInterface(name="HvacStatus", namespace="nsprototype::nsserviceinterface::nshvacstatus")
hvac_status_service.add_data_element(name="CompressorStatus", datatype=BaseTypes.UINT32_T)
hvac_status_service.add_data_element(name="ValveStatus", datatype=BaseTypes.UINT32_T)
hvac_status_service.add_data_element(name="FanRightSpeed", datatype=BaseTypes.UINT32_T)
hvac_status_service.add_data_element(name="FanLeftSpeed", datatype=BaseTypes.UINT32_T)
hvac_status_service.add_operation(
    name="SetDegree",
    in_parameter={"Value": BaseTypes.UINT8_T},
)

my_interface = vafpy.ModuleInterface(name="DataExchangeInterface", namespace="demo")
my_interface.add_data_element(name="MyValue", datatype=BaseTypes.UINT32_T)
my_interface.add_operation(name="MyFunction", in_parameter={"MyValueIn": BaseTypes.UINT32_T}, out_parameter={"MyValueOut": BaseTypes.UINT32_T})
```

Once complete, the configuration needs to be exported to JSON by using the following command:

``` bash
vaf model generate
```

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
to make the model elements from there accessible in the app-module project:

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
According to the illustration above, `app_module1` is supposed to connected to the `HvacStatus`
r-port and therefore needs a corresponding consumer interface. For communication with `app_module2`
it acts as provider of the earlier defined `DataExchangeInterface`. The `app_module2` acts as
consumer counterpart for the `DataExchangeInterface` and towards the platform-side, needs a provider
interface for `HvacControl`.

Add the configurations given below to the respective ./model/<app-module>.py files.
The configuration for `app_module1`:

``` python
app_module1 = vafpy.ApplicationModule(name="AppModule1", namespace="demo")

app_module1.add_consumed_interface(instance_name="HvacStatusConsumer", interface=interfaces.Nsprototype.Nsserviceinterface.Nshvacstatus.hvac_status)
app_module1.add_provided_interface(instance_name="DataExchangeProvider", interface=interfaces.Demo.data_exchange_interface)

periodic_task = vafpy.Task(name="PeriodicTask", period=timedelta(milliseconds=200))
app_module1.add_task(task=periodic_task)
```

The configuration for `app_module2`:

``` python
app_module2 = vafpy.ApplicationModule(name="AppModule2", namespace="demo")

app_module2.add_consumed_interface(instance_name="DataExchangeConsumer", interface=interfaces.Demo.data_exchange_interface)
app_module2.add_provided_interface(instance_name="HvacControlProvider", interface=interfaces.Nsprototype.Nsserviceinterface.Nshvaccontrol.hvac_control)

periodic_task = vafpy.Task(name="PeriodicTask", period=timedelta(milliseconds=200))
app_module2.add_task(task=periodic_task)
```

In order to run the HvacDemo at the end, a counterpart is needed to provide the platform interfaces.
This is implemented with another app module, which is then compiled into a second executable:

``` python
app_module3 = vafpy.ApplicationModule(name="AppModule3", namespace="demo")

app_module3.add_consumed_interface(instance_name="HvacControlConsumer", interface=interfaces.Nsprototype.Nsserviceinterface.Nshvaccontrol.hvac_control)
app_module3.add_provided_interface(instance_name="HvacStatusProvider", interface=interfaces.Nsprototype.Nsserviceinterface.Nshvacstatus.hvac_status)

periodic_task = vafpy.Task(name="PeriodicTask", period=timedelta(milliseconds=200))
app_module3.add_task(task=periodic_task)
```

Once complete, the next step is to export the configuration to JSON and code generation. The
resulting file from the export (`./model/model.json`) contains all information that is needed for
the code generation step.

``` bash
vaf project generate
```

The generated code can be divided into read-write and read-only parts. Former gets generated to the
`./implementation` folder. This is user space, where the framework only provides implementation
stubs for the developer to start. In case of re-generation, a 3-way merge strategy based on
git-merge is applied to the files in this location. The read-only parts get generated to `./src-gen`
and `./test-gen`. Those folders are under control of the framework. Any user modification will be
overwritten in case of re-generation.

The entry file for the user to add own code is located in `./implementation/src/app_module1.cpp`.
Some sample code for reference is shipped as part of the container and located in:
`/opt/vaf/Demo/HvacDemo/MSRA/app-centric/app/src/app_module1`.

The application module project is also ready to be built as library. To do so and in order to check
if added code passes the compiler checks, execute the following two steps. First, start preset to
prepare the build step, which includes CMake preset and Conan cache setup. Second, trigger the
CMake-based build process. On the command line:

``` bash
vaf make build
```

Once done with `AppModule1`, the same procedure can be repeated for `AppModule2` and `AppModule3`.

### Unit testing of app-modules

The Vector Application Framework provides means for unit testing. Test mocks for Googletest are
generated to `./test-gen` accordingly and allow independent first-level testing of application
modules. Custom test code can be added in the corresponding `tests.cpp` file in the
`./implementation/test/unittest` folder.

Some sample test code for the app-modules is provided for reference in:
`/opt/vaf/Demo/HvacDemo/MSRA/app-centric/app/src`.

Please note. The build step of these unit tests is enabled by default. To deactivate it, use:

``` bash
vaf project generate --skip-make-preset
vaf make preset -d -DVAF_BUILD_TESTS=OFF
vaf make build
```

The resulting test binaries get stored in `build/Release/bin` for execution.

### Executable integration

Final integration of all application modules is done using a **VAF integration project**. This is
where the whole application, which potentially consists of multiple executables (HvacExecutable and
TestHvacExecutable), gets integrated. In practice, app-modules and platform modules, as provided by
the framework, get instantiated and wired. The complete picture of the HvacExecutable is illustrated
below.

![hvac_detailed](../../figures/hvac-detailed.svg)

To start a new integration project, execute the following steps:

``` bash
vaf project init integration
Enter your project name: HvacDemo

cd HvacDemo
```

To get started, all relevant application-module projects need to be imported in first place:

```bash
vaf project import
Enter the path to the application module project to be imported: ../AppModule1/

vaf project import
Enter the path to the application module project to be imported: ../AppModule2/

vaf project import
Enter the path to the application module project to be imported: ../AppModule3/
```

The import command adds new files to `./vaf/model/application_modules`. This includes relevant path
information but, most and foremost, the importer and CaC-support artifacts, which make the model
elements from the app-module accessible for the configuration in the integration project.

Next step is the configuration of the integration project in `./model/vaf/hvac_demo.py`.

The configuration for the integration project:

```python
# Create HvacExecutable
HvacExecutable = Executable("hvac_app", timedelta(milliseconds=20))
HvacExecutable.add_application_module(AppModule1, [("PeriodicTask", timedelta(milliseconds=1), 0)])
HvacExecutable.add_application_module(AppModule2, [("PeriodicTask", timedelta(milliseconds=1), 1)])

HvacExecutable.connect_interfaces(
    AppModule1, 
    Instances.AppModule1.ProvidedInterfaces.DataExchangeProvider, 
    AppModule2, 
    Instances.AppModule2.ConsumedInterfaces.DataExchangeConsumer
)

HvacExecutable.connect_consumed_interface_to_msra(
    AppModule1,
    Instances.AppModule1.ConsumedInterfaces.HvacStatusConsumer
)
HvacExecutable.connect_provided_interface_to_msra(
    AppModule2,
    Instances.AppModule2.ProvidedInterfaces.HvacControlProvider
)

# Create TestHvacExecutable

TestHvacExecutable = Executable("test_hvac_app", timedelta(milliseconds=20))

TestHvacExecutable.add_application_module(AppModule3, [])

TestHvacExecutable.connect_consumed_interface_to_msra(
    AppModule3, 
    Instances.AppModule3.ConsumedInterfaces.HvacControlConsumer
)
TestHvacExecutable.connect_provided_interface_to_msra(
    AppModule3, 
    Instances.AppModule3.ProvidedInterfaces.HvacStatusProvider
)
```

With this step done, the HVAC demo configuration part is complete. The next step is model and code
generation:

``` bash
vaf project generate
```

Using `--mode prj` or `--mode all` further allows to set the scope of this command to either, 
the current integration project only (prj), or to this and all related sub-projects (all).

Although the VAF-related code has been generated, the MSRA-related code has not yet been generated
and no ARXML model is yet available. In order to be able to compile the project, a suitable ARXML is
first required, which is necessary for generating the MSRA-related code. Both, the generation of the
ARXML and the code is done using:

``` bash
vaf platform export msra -o ./model/msra
```

To complete the integration project, build preset, build, and final installation are missing:

``` bash
vaf make preset
vaf make build
vaf make install
```

The final (MSRA-compatible) executable for execution is stored in `./build/Release/install/opt`.

## Running the HVAC application

Open three terminal sessions to get started. One for the HvacExecutable process (hvac_app), one for
the counterpart (test_hvac_app), and one for the IPC service discovery daemon of MSRA.

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
`./build/Release/install/opt/amsr_ipcservicediscovery_daemon`. Then launch both executables, located
in the corresponding `./build/Release/install/opt` directories of your `app` and `test-app`
projects.

>**ℹ️ Hint**  
> Since the adaptive applications require some configuration files, it is important to change the
> directory to the base of the installed application (e.g. `./build/Release/install/opt/hvac_app`)
> before running the executable.
