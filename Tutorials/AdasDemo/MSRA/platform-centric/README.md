# ADAS Demo, platform-centric scenario based on MSRA

The platform-centric use case describes a project setup, where some platform model is already
available and supposed to be used as starting point for the development of the application in
question. In this demo, it is an AUTOSAR Adaptive model that consists of a design and a deployment
part. The **design part** covers the definition of multiple service interfaces as follows:

| Service: BrakeService            | Service: ImageService       |
| -------------------------------- | --------------------------- |
| Event: brake_action              | Event: camera_image         |
| Field: brake_summand_coefficient | Field: image_scaling_factor |
| Method: SumTwoSummands           | Method: GetImageSize        |

| Service: SteeringAngleService  | Service: VelocityService |
| ------------------------------ | ------------------------ |
| Event: steering_angle          | Event: car_velocity      |

The **deployment part** defines one executable with multiple consumer ports (r-ports) for the sensors
and one provider port (p-port) of the Brake interface:

* Executable: Executable
* RPorts: Camera (left), Camera (right), SteeringAngle, Velocity
* PPort: Brake

## AdasApplication (app)

As mentioned above, the executable and its external connections are known. The internal architecture
of the executable, however, is open and within the scope of the **Vector Application Framework
(VAF)**. Within this executable two application modules - sensor fusion and collision detection -
will be used. The collision detection receives the object detection list from the sensor fusion
application module. The sensor fusion tasks consume the left and right camera's ImageService, the
VelocityService and the SteeringAngleService. From this information, the object detection list is
prepared and send to the collision detection application module. The collision detection application
module then commands the the brake service accordingly. A high-level illustration of this setup is
given below:

![adas_demo](../../figures/adas-demo.svg)

### Import and definition of interfaces

At first, internal and external interfaces for the application modules need to be imported to the
VAF. For that, a **VAF interface project** can be used.

To create a new interface project use the VAF command line tool as follows:

``` bash
vaf project init interface
Enter your project name: Interfaces
```

Next, enter the project directory.

``` bash
cd Interfaces
```

The example AUTOSAR model is shipped as part of the container and located in:
`/opt/vaf/Demo/AdasDemo/MSRA/platform-centric/app`. Import the ADAS platform model by typing the
following VAF command to a terminal:

``` bash
vaf platform derive msra
Enter the path to the ARXML model directory: /opt/vaf/Demo/AdasDemo/MSRA/platform-centric/app/model/msra
```

Two new files are added to the project folder by this command. `msra-derived-model.json` is the VAF
model file in JSON format. It contains all relevant information as imported from the original ARXML
files. `msra.py` is the Configuration as Code (CaC) support, which is needed to access the model
artifacts from the Python configuration.

Both, consolidation of external model information and definition of own data types and interfaces is
done in Python. For that, open the template file `interfaces.py`. To import the interfaces from the
previous platform derive step, uncomment line #4:

``` **python**
from .msra import *
```

The definition of own data types and interfaces is also done in the `interfaces.py` file by using
elements from the `vafpy` package. For the ADAS example, one extra interface for the direct data
exchange of the two app-modules is needed. Because the interface uses a complex data type, it must
be created beforehand:

``` python
# Define the required data types
od_struct = vafpy.datatypes.Struct(name="ObjectDetection", namespace="adas::interfaces")
od_struct.add_subelement(name="x", datatype=BaseTypes.UINT64_T)
od_struct.add_subelement(name="y", datatype=BaseTypes.UINT64_T)
od_struct.add_subelement(name="z", datatype=BaseTypes.UINT64_T)

od_list = vafpy.datatypes.Vector(
    name="ObjectDetectionList",
    namespace="adas::interfaces",
    datatype=od_struct
)

# Define the interface
od_interface = vafpy.ModuleInterface(
    name="ObjectDetectionListInterface",
    namespace="nsapplicationunit::nsmoduleinterface::nsobjectdetectionlist",
)
od_interface.add_data_element(name="object_detection_list", datatype=od_list)
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

To start a new application module project, leave the interface project and type:

``` bash
cd ..
vaf project init app-module
Enter the name of the app-module: SensorFusion
Enter the namespace of the app-module: NsApplicationUnit::NsSensorFusion
```

Also here, switch to the just created project directory:

``` bash
cd SensorFusion
```

In first place, the above-created data exchange file from the interface project needs to be imported
to make the model elements from there accessible in the app-module project. Use the following
command:

```bash
vaf project import
Enter the path to the exported VAF model JSON file: ../Interfaces/export/Interfaces.json
```

Next step is the configuration of the application module. For that, open the file `./model/sensor_fusion.py`. To
complete the import from the interface project, uncomment line #6:

``` python
from .imported_models import *
```

The configuration of the application module is done completely in this Configuration as Code file.
According to the illustration above, `sensor_fusion` is supposed to connected to the sensor r-ports
and therefore needs corresponding consumer interfaces. For communication with `collision_detection`
it acts as provider of the earlier defined `ObjectDetectionListInterface`. The `collision_detection`
module acts as consumer counterpart for the `ObjectDetectionListInterface` and towards the
platform-side, needs a provider interface for `BrakeService`.

The configuration for `sensor_fusion`:  
Extend the existing sensor_fusion with consumed and provided interfaces.

``` python
sensor_fusion = vafpy.ApplicationModule(name="SensorFusion", namespace="NsApplicationUnit::NsSensorFusion")

sensor_fusion.add_provided_interface("ObjectDetectionListModule", interfaces.Nsapplicationunit.Nsmoduleinterface.Nsobjectdetectionlist.object_detection_list_interface)
sensor_fusion.add_consumed_interface("ImageServiceConsumer1", interfaces.Af.AdasDemoApp.Services.image_service)
sensor_fusion.add_consumed_interface("ImageServiceConsumer2", interfaces.Af.AdasDemoApp.Services.image_service)
sensor_fusion.add_consumed_interface("SteeringAngleServiceConsumer", interfaces.Af.AdasDemoApp.Services.steering_angle_service)
sensor_fusion.add_consumed_interface("VelocityServiceConsumer", interfaces.Af.AdasDemoApp.Services.velocity_service)

periodic_task = vafpy.Task(name="PeriodicTask", period=timedelta(milliseconds=200))
sensor_fusion.add_task(task=periodic_task)
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

The entry file for the user to add own code is located in `./implementation/src/sensor_fusion.cpp`.
Some sample code for reference is shipped as part of the container and located in:
`/opt/vaf/Demo/AdasDemo/MSRA/platform-centric/app`.

The application module project is also ready to be built as library. To do so and in order to check
if added code passes the compiler checks, trigger the CMake-based build process. 

``` bash
vaf make build
```

Once done with the `SensorFusion` module, the same procedure can be repeated for the
`CollisionDetection` module. Starting with the creation of a new app-module project.

``` bash
cd ..
vaf project init app-module
Enter the name of the app-module: CollisionDetection
Enter the namespace of the app-module: NsApplicationUnit::NsCollisionDetection 

cd CollisionDetection
```

First, the above-created data exchange file from the interface project needs to be imported
to make the model elements from there accessible in the app-module project. Use the following
command:

```bash
vaf project import
Enter the path to the exported VAF model JSON file: ../Interfaces/export/Interfaces.json
```

Next step is the configuration of the application module. For that, open the file
`./model/collision_detection.py`. To complete the import from the interface project, uncomment line
#6:

``` python
from .imported_models import *
```

The configuration for `collision_detection` is as follows:

``` python
collision_detection = vafpy.ApplicationModule(
    name="CollisionDetection", namespace="NsApplicationUnit::NsCollisionDetection"
)

collision_detection.add_provided_interface("BrakeServiceProvider", interfaces.Af.AdasDemoApp.Services.brake_service)
collision_detection.add_consumed_interface("ObjectDetectionListModule", interfaces.Nsapplicationunit.Nsmoduleinterface.Nsobjectdetectionlist.object_detection_list_interface)

periodic_task = vafpy.Task(name="PeriodicTask", period=timedelta(milliseconds=200))
collision_detection.add_task(task=periodic_task)
```

Once complete, the next step is to export the configuration to JSON and code generation. The
resulting file from the export (`./model/model.json`) contains all information that is needed for
the code generation step.

``` bash
vaf project generate
```

The application module project is also ready to be built as library. To do so and in order to check
if added code passes the compiler checks, trigger the CMake-based build process. 

``` bash
vaf make build
```

### Unit testing of app-modules

The Vector Application Framework provides means for unit testing of app-modules. Test mocks for
Googletest are generated to `./test-gen` accordingly and allow independent first-level testing of
application modules. Custom test code can be added in the corresponding `tests.cpp` file in the
`./implementation/test/unittest` folder.

> **ℹ️ Note**  
> The build step of these unit tests is enabled by default. To deactivate it, use the following
> commands:
>
> ``` bash
> vaf project generate --skip-make-preset
> vaf make preset -d -DVAF_BUILD_TESTS=OFF
> vaf make build
> ```

The resulting test binaries get stored in `build/Release/bin` for execution.

### Executable integration

Final integration of all application modules is done using a **VAF integration project**. This is
where the whole application, which potentially consists of multiple executables, gets integrated. In
practice, app-modules and platform modules, as provided by the framework, get instantiated and
wired.

To start a new integration project, execute the following steps:

``` bash
cd ..
vaf project init integration
Enter your project name: AdasApplication

cd AdasApplication
```

To get started, all relevant application-module projects need to be imported. Use the
following command for that:

```bash
vaf project import
Enter the path to the application module project to be imported ../SensorFusion/

vaf project import
Enter the path to the application module project to be imported ../CollisionDetection/
```

The import command adds new files to `./model/vaf/application_modules`. This includes relevant path
information but, most and foremost, the importer and CaC-support artifacts, which make the model
elements from the app-module accessible for the configuration in the integration project.

In this example, the executable is already defined as part of the AUTOSAR model. To import this
deployment information to the project and also to execute the code generators of MICROSAR Adaptive
(MSRA), use the following command:

``` bash
vaf platform derive msra
Enter the path to the ARXML model directory: /opt/vaf/Demo/AdasDemo/MSRA/platform-centric/app/model/msra
```

The generated code artifacts for MSRA go into the read-only directory `./msra-gen`.

Next step is the configuration of the integration project in `./model/vaf/adas_application.py`.

To import the platform artifacts from MSRA, uncomment line #7 in the template:

``` python
from msra import *
```

> **ℹ️ Note**  
> Checking the file `msra.py` or one of the import files for the app-modules can be helpful to
> understand the class-tree structure, which is behind the VAF Configuration as Code solution.

The executable is already defined in `msra.py:49`. Means, we can directly continue with the step of
instantiating and adding app-modules in `./model/vaf/adas_application.py` as follows:

``` python
AdasApplication.executable.add_application_module(SensorFusion, [])
AdasApplication.executable.add_application_module(CollisionDetection, [])
```

The second parameter, here set as empty list, allows the definition of an integration-specific task
mapping. This includes the task period and execution order and allows the integrator to overrule the
original settings from the application module project.

The two app-module instances now can be connected. Among each other, on the one hand, and with
platform modules, on the other hand. The below configuration code snippet details the part for
executable-internal communication in this example project:

``` python
AdasApplication.executable.connect_interfaces(
    SensorFusion, Instances.SensorFusion.ProvidedInterfaces.ObjectDetectionListModule,
    CollisionDetection, Instances.CollisionDetection.ConsumedInterfaces.ObjectDetectionListModule,
)
```

The communication with a lower-layer platform is abstracted by platform modules. They deal with the
platform API towards the lower layer, i.e. the middleware stack, and with the VAF API towards the
upper, the application layer. The connection between application and platform modules is configured
as follows:

``` python
AdasApplication.executable.connect_consumed_interface_to_platform(
    SensorFusion,
    Instances.SensorFusion.ConsumedInterfaces.ImageServiceConsumer1,
    AdasApplication.ConsumerModules.r_port_image_service1,
)
AdasApplication.executable.connect_consumed_interface_to_platform(
    SensorFusion,
    Instances.SensorFusion.ConsumedInterfaces.ImageServiceConsumer2,
    AdasApplication.ConsumerModules.r_port_image_service2,
)
AdasApplication.executable.connect_consumed_interface_to_platform(
    SensorFusion,
    Instances.SensorFusion.ConsumedInterfaces.SteeringAngleServiceConsumer,
    AdasApplication.ConsumerModules.r_port_steering_angle_service,
)
AdasApplication.executable.connect_consumed_interface_to_platform(
    SensorFusion,
    Instances.SensorFusion.ConsumedInterfaces.VelocityServiceConsumer,
    AdasApplication.ConsumerModules.r_port_velocity_service,
)

AdasApplication.executable.connect_provided_interface_to_platform(
    CollisionDetection,
    Instances.CollisionDetection.ProvidedInterfaces.BrakeServiceProvider,
    AdasApplication.ProviderModules.p_port_brake_service,
)
```

With this step done, the ADAS demo configuration part is complete. The next step is model and code generation using:

``` bash
vaf project generate
```

Using `--mode prj` or `--mode all` allows to set the scope of this command to either, 
the current integration project only (prj), or to this and all related sub-projects (all).

To complete the integration project, make build, and final installation are missing:

``` bas--mode prj
vaf make build
vaf make install
```

The final (MSRA-compatible) executable for execution is stored in `./build/Release/install/opt`.

## AdasPlatform (test-app)

In order to run the AdasApplication, a counterpart is required that mimics the platform
side and consumes/provides the necessary services as expected from there. This counterpart can be
built with the same workflow as described above for the AdasApplication.

>**ℹ️ Note**  
> The AdasPlatform executable in the example contains one app-module only (MsraPlatform). In this
> case, the interface project can be skipped as no internal communication is required.
> The same holds for importing the app-module project. It can also be created directly within the
> integration project. To do so, use `vaf project create app-module`.
> Instead of being linked to the integration project, the app-module will live as local
> copy in `./src/application_modules`.

To start a new integration project, execute the following steps:

``` bash
cd ..
vaf project init integration
Enter your project name: AdasPlatform

cd AdasPlatform
```

Add application module to the integration project:

``` bash
vaf project create app-module
Enter the name of the app-module: MsraPlatform
Enter the namespace of the app-module: NsApplicationUnit::NsMsraPlatform
```

Switch to the just created app-module directory:

``` bash
cd src/application_modules/msra_platform/
```

The example AUTOSAR model is shipped as part of the container and located in:
`/opt/vaf/Demo/AdasDemo/MSRA/platform-centric/test-app`. Import the ADAS platform model by typing the
following VAF command to a terminal:

``` bash
vaf platform derive msra
Enter the path to the ARXML model directory: /opt/vaf/Demo/AdasDemo/MSRA/platform-centric/test-app/model/msra
```

Two new files are added to the project folder by this command. `msra-derived-model.json` is the VAF
model file in JSON format. It contains all relevant information as imported from the original ARXML
files. `msra.py` is the Configuration as Code (CaC) support, which is needed to access the model
artifacts from the Python configuration.

Both, consolidation of external model information and definition of own data types and interfaces is
done in Python. For that, open the template file `./model/msra_platform.py`. To import the
interfaces from the previous platform derive step, uncomment the below line:

``` **python**
from .msra import *
```

The definition of own data types and interfaces is also done in the `./model/msra_platform.py` file by using
elements from the `vafpy` package. Next step is the configuration of the application module.
The configuration of the application module is done completely in this Configuration as Code file.
According to the illustration above, `MsraPlatform` acts as the platform-side for AdasApplication.
For communication with AdasApplication it acts as provider of the earlier defined  `ImageService1`, `ImageService2`´,
`VelocityService` and `SteeringAngleService`, and as consumer of `BrakeService` interfaces.

The configuration for `MsraPlatform`:  
Extend the existing msra_platform app-module with consumed and provided interfaces.

``` python
msra_platform = vafpy.ApplicationModule(name="MsraPlatform", namespace="NsApplicationUnit::NsMsraPlatform")

msra_platform.add_consumed_interface("BrakeServiceConsumer", Af.AdasDemoApp.Services.brake_service)
msra_platform.add_provided_interface("ImageServiceProvider1", Af.AdasDemoApp.Services.image_service)
msra_platform.add_provided_interface("ImageServiceProvider2", Af.AdasDemoApp.Services.image_service)
msra_platform.add_provided_interface("SteeringAngleServiceProvider", Af.AdasDemoApp.Services.steering_angle_service)
msra_platform.add_provided_interface("VelocityServiceProvider", Af.AdasDemoApp.Services.velocity_service)

periodic_task = vafpy.Task(name="PeriodicTask", period=timedelta(milliseconds=200))
msra_platform.add_task(task=periodic_task)
```

Once complete, the next step is to export the configuration to JSON and code generation.
The resulting file from the export (`./model/model.json`) contains all information that is needed for the code generation step.

``` bash
vaf project generate
```

The generated code can be divided into read-write and read-only parts. Former gets generated to the
`./implementation` folder. This is user space, where the framework only provides implementation
stubs for the developer to start. In case of re-generation, a 3-way merge strategy based on
git-merge is applied to the files in this location. The read-only parts get generated to `./src-gen`
and `./test-gen`. Those folders are under control of the framework. Any user modification will be
overwritten in case of re-generation.

The entry file for the user to add own code is located in `./implementation/src/msra_platform.cpp`.
Some sample code for reference is shipped as part of the container and located in:
`/opt/vaf/Demo/AdasDemo/MSRA/platform-centric/test-app`.

The application module project is also ready to be built as library. To do so and in order to check
if added code passes the compiler checks, trigger the
CMake-based build process. 

``` bash
vaf make build
```

Final integration of the application module is done in the **VAF integration project**. In this
example, the executable is already defined as part of the AUTOSAR model. To import this deployment
information to the project and also to execute the code generators of MICROSAR Adaptive (MSRA), use
the following command:

``` bash
cd <path-to-integration-project>

vaf platform derive msra
Enter the path to the ARXML model directory: /opt/vaf/Demo/AdasDemo/MSRA/platform-centric/test-app/model/msra
```

The generated code artifacts for MSRA go into the read-only directory `./msra-gen`.

Next step is the configuration of the integration project in `./model/vaf/adas_platform.py`.

To import the platform artifacts from MSRA, uncomment the below line in the template:

``` python
from .msra import *
```

The executable is already defined in `msra.py:49`. Means, we can directly continue with the step of
instantiating and adding app-module as follows:

``` python
AdasPlatform.executable.add_application_module(MsraPlatform, [])
```

The connection between platform module and application modules is configured as follows:

``` python
AdasPlatform.executable.connect_provided_interface_to_platform(
    MsraPlatform,
    Instances.MsraPlatform.ProvidedInterfaces.ImageServiceProvider1,
    AdasPlatform.ProviderModules.p_port_image_service1,
)
AdasPlatform.executable.connect_provided_interface_to_platform(
    MsraPlatform,
    Instances.MsraPlatform.ProvidedInterfaces.ImageServiceProvider2,
    AdasPlatform.ProviderModules.p_port_image_service2,
)
AdasPlatform.executable.connect_provided_interface_to_platform(
    MsraPlatform,
    Instances.MsraPlatform.ProvidedInterfaces.VelocityServiceProvider,
    AdasPlatform.ProviderModules.p_port_velocity_service,
)
AdasPlatform.executable.connect_provided_interface_to_platform(
    MsraPlatform,
    Instances.MsraPlatform.ProvidedInterfaces.SteeringAngleServiceProvider,
    AdasPlatform.ProviderModules.p_port_steering_angle_service,
)

AdasPlatform.executable.connect_consumed_interface_to_platform(
    MsraPlatform,
    Instances.MsraPlatform.ConsumedInterfaces.BrakeServiceConsumer,
    AdasPlatform.ConsumerModules.r_port_brake_service,
)
```

With this step done, the AdasPlatform configuration part is complete. The next step is model and
code generation using:

``` bash
vaf model update
vaf project generate
```

Using `--mode prj` or `--mode all` allows to set the scope of this command to either, 
the current integration project only (prj), or to this and all related sub-projects (all).

To complete the integration project, build and final installation are missing:

``` bash
vaf make build
vaf make install
```

The final (MSRA-compatible) executable is stored in `./build/Release/install/opt`.

## Running the ADAS application

The execution of the ADAS executable is possible after building. 

Configure the environment per terminal session using the `setup.sh` script. It sets the
necessary environment variables.

``` bash
source setup.sh
```

You can launch the executables of the IPC service discovery, the AdasApplication and the
AdasPlatform from `./build/Release/install/opt` directory of your working directory.

``` bash
cd AdasApplication/build/Release/install/opt/amsr_ipcservicediscovery_daemon
./bin/amsr_ipcservicediscovery_daemon

cd AdasApplication/build/Release/install/opt/adas_application
./bin/adas_application

cd AdasPlatform/build/Release/install/opt/adas_platform
./bin/adas_platform
```
