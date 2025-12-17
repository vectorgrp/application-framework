# ADAS demo, app-centric scenario based on MSRA

The app-centric use case describes a project setup, where no platform model is available. Here, the
aimed target platform is AUTOSAR Adaptive. A suitable AUTOSAR model must therefore be created by the
framework during development.

## AdasExecutable (app)

Plan in this demo is to introduce two application modules sensor fusion and collision detection.
The collision detection receives the
object detection list from the sensor fusion application module. The sensor fusion tasks consume 
the left and right camera's ImageService, the VelocityService and the SteeringAngleService. From 
this information, the object detection list is prepared and send to the collision detection 
application module. The collision detection application module then commands the the brake 
service accordingly. A high-level illustration of this setup is given below:

![adas_demo](../../figures/adas-demo.svg)

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
# Brake Service
brake_pressure = vafpy.datatypes.Struct(name="BrakePressure", namespace="datatypes")
brake_pressure.add_subelement(name="timestamp", datatype=BaseTypes.UINT64_T)
brake_pressure.add_subelement(name="value", datatype=BaseTypes.UINT8_T)

brake_service = vafpy.ModuleInterface(name="BrakeService", namespace="af::adas_demo_app::services")
brake_service.add_data_element(name="brake_action", datatype=brake_pressure)
brake_service.add_data_element(name="brake_summand_coefficient_FieldNotifier", datatype=BaseTypes.UINT64_T)
brake_service.add_operation(name="brake_summand_coefficient_FieldGetter", out_parameter={"data": BaseTypes.UINT64_T})
brake_service.add_operation(name="brake_summand_coefficient_FieldSetter", in_parameter={"data": BaseTypes.UINT64_T})
brake_service.add_operation(
    name="SumTwoSummands",
    in_parameter={"summand_one": BaseTypes.UINT16_T, "summand_two": BaseTypes.UINT16_T},
    out_parameter={"sum": BaseTypes.UINT16_T},
)


# Object Detection List
od_struct = vafpy.datatypes.Struct(name="ObjectDetection", namespace="adas::interfaces")
od_struct.add_subelement(name="x", datatype=BaseTypes.UINT64_T)
od_struct.add_subelement(name="y", datatype=BaseTypes.UINT64_T)
od_struct.add_subelement(name="z", datatype=BaseTypes.UINT64_T)

od_list = vafpy.datatypes.Vector(
    name="ObjectDetectionList",
    namespace="adas::interfaces",
    datatype=od_struct,
)

od_interface = vafpy.ModuleInterface(
    name="ObjectDetectionListInterface",
    namespace="nsapplicationunit::nsmoduleinterface::nsobjectdetectionlist",
)
od_interface.add_data_element(name="object_detection_list", datatype=od_list)


# ImageService
my_vector = vafpy.datatypes.Vector(
    name="UInt8Vector", namespace="datatypes", datatype=BaseTypes.UINT8_T
)

image = vafpy.datatypes.Struct(name="Image", namespace="datatypes")
image.add_subelement(name="timestamp", datatype=BaseTypes.UINT64_T)
image.add_subelement(name="height", datatype=BaseTypes.UINT16_T)
image.add_subelement(name="width", datatype=BaseTypes.UINT16_T)
image.add_subelement(name="R", datatype=my_vector)
image.add_subelement(name="G", datatype=my_vector)
image.add_subelement(name="B", datatype=my_vector)

image_service = vafpy.ModuleInterface(name="ImageService", namespace="af::adas_demo_app::services")
image_service.add_data_element(name="camera_image", datatype=image)
image_service.add_data_element(name="image_scaling_factor_FieldNotifier", datatype=BaseTypes.UINT64_T)
image_service.add_operation(name="image_scaling_factor_FieldGetter", out_parameter={"data": BaseTypes.UINT64_T})
image_service.add_operation(name="image_scaling_factor_FieldSetter", in_parameter={"data": BaseTypes.UINT64_T})
image_service.add_operation(
    name="GetImageSize", out_parameter={"width": BaseTypes.UINT16_T, "height": BaseTypes.UINT16_T}
)


# VelocityService
velocity = vafpy.datatypes.Struct(name="Velocity", namespace="datatypes")
velocity.add_subelement(name="timestamp", datatype=BaseTypes.UINT64_T)
velocity.add_subelement(name="value", datatype=BaseTypes.UINT16_T)

velocity_service = vafpy.ModuleInterface(name="VelocityService", namespace="af::adas_demo_app::services")
velocity_service.add_data_element(name="car_velocity", datatype=velocity)


# Steering Angle
steering_angle = vafpy.datatypes.Struct(name="SteeringAngle", namespace="datatypes")
steering_angle.add_subelement(name="timestamp", datatype=BaseTypes.UINT64_T)
steering_angle.add_subelement(name="value", datatype=BaseTypes.UINT16_T)

steering_angle_service = vafpy.ModuleInterface(name="SteeringAngleService", namespace="af::adas_demo_app::services")
steering_angle_service.add_data_element(name="steering_angle", datatype=steering_angle)
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
cd ..
vaf project init app-module
Enter the name of the app-module: SensorFusion
Enter the namespace of the app-module: NsApplicationUnit::NsSensorFusion

cd SensorFusion
```

In first place, the above-created data exchange file from the interface project needs to be imported
to make the model elements from there accessible in the app-module project:

```bash
vaf project import
Enter the path to the exported VAF model JSON file: ../Interfaces/export/Interfaces.json
```

Next step is the configuration of the application module. For that, open the file
`./model/sensor_fusion.py`. To complete the import from the interface project, uncomment line #6:

``` python
from .imported_models import *
```

According to the illustration above, `sensor_fusion` is supposed to connected to the sensor r-ports
and therefore needs corresponding consumer interfaces. For communication with `collision_detection`
it acts as provider of the earlier defined `ObjectDetectionListInterface`. The `collision_detection`
module acts as consumer counterpart for the `ObjectDetectionListInterface` and towards the
platform-side, needs a provider interface for `BrakeService`.

Add the configurations given below to the respective ./model/<app-module>.py files.
The configuration for `sensor_fusion`:

``` python
sensor_fusion = vafpy.ApplicationModule(name="SensorFusion", namespace="NsApplicationUnit::NsSensorFusion")

sensor_fusion.add_provided_interface(
    "ObjectDetectionListModule",
    interfaces.Nsapplicationunit.Nsmoduleinterface.Nsobjectdetectionlist.object_detection_list_interface,
)
sensor_fusion.add_consumed_interface("ImageServiceConsumer1", interfaces.Af.AdasDemoApp.Services.image_service)
sensor_fusion.add_consumed_interface("ImageServiceConsumer2", interfaces.Af.AdasDemoApp.Services.image_service)
sensor_fusion.add_consumed_interface("SteeringAngleServiceConsumer", interfaces.Af.AdasDemoApp.Services.steering_angle_service)
sensor_fusion.add_consumed_interface("VelocityServiceConsumer", interfaces.Af.AdasDemoApp.Services.velocity_service)

p_200ms = timedelta(milliseconds=200)
step1 = vafpy.Task(name="Step1", period=p_200ms, preferred_offset=0)
step2 = vafpy.Task(name="Step2", period=p_200ms, preferred_offset=0)
step3 = vafpy.Task(name="Step3", period=p_200ms, preferred_offset=0)

sensor_fusion.add_task(task=step1)
sensor_fusion.add_task_chain(tasks=[step2, step3], run_after=[step1])
sensor_fusion.add_task(vafpy.Task(name="Step4", period=p_200ms, preferred_offset=0, run_after=[step1]))
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

The entry file for the user to add own code is located in `./implementation/src/sensor_fusion.cpp`.
Some sample code for reference is shipped as part of the container and located in:
`/opt/vaf/Demo/AdasDemo/MSRA/app-centric/app`.

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

The configuration for `collision_detection`:

``` python
collision_detection = vafpy.ApplicationModule(
    name="CollisionDetection", namespace="NsApplicationUnit::NsCollisionDetection"
)
collision_detection.add_provided_interface("BrakeServiceProvider", interfaces.Af.AdasDemoApp.Services.brake_service)
collision_detection.add_consumed_interface(
    "ObjectDetectionListModule",
    interfaces.Nsapplicationunit.Nsmoduleinterface.Nsobjectdetectionlist.object_detection_list_interface,
)

periodic_task = vafpy.Task(name="PeriodicTask", period=timedelta(milliseconds=200))
collision_detection.add_task(task=periodic_task)
```

Once complete, the next step is to export the configuration to JSON and code generation.
The resulting file from the export (`./model/model.json`) contains all information that 
is needed for the code generation step.

``` bash
vaf project generate
```

The application module project is also ready to be built as library. To do so and in order to check
if added code passes the compiler checks, trigger the
CMake-based build process. 

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


Next step is the configuration of the integration project in `./model/vaf/adas_application.py`.

``` python
# Create application
adas_demo_app = Executable("adas_demo_app", timedelta(milliseconds=20))

b_10ms = timedelta(milliseconds=10)
adas_demo_app.add_application_module(
    SensorFusion,
    [
        (Instances.SensorFusion.Tasks.Step1, b_10ms, 0),
        (Instances.SensorFusion.Tasks.Step2, b_10ms, 0),
        (Instances.SensorFusion.Tasks.Step3, b_10ms, 0),
        (Instances.SensorFusion.Tasks.Step4, b_10ms, 0),
    ],
)
adas_demo_app.add_application_module(CollisionDetection, [])

# connect intra process interfaces
adas_demo_app.connect_interfaces(
    SensorFusion,
    Instances.SensorFusion.ProvidedInterfaces.ObjectDetectionListModule,
    CollisionDetection,
    Instances.CollisionDetection.ConsumedInterfaces.ObjectDetectionListModule,
)

# connect middleware interfaces
adas_demo_app.connect_consumed_interface_to_msra(
    SensorFusion,
    Instances.SensorFusion.ConsumedInterfaces.ImageServiceConsumer1,
    "ImageService1",
    vafmodel.MSRAConnectionPointDeploymentType.SOME_IP,
)
adas_demo_app.connect_consumed_interface_to_msra(
    SensorFusion,
    Instances.SensorFusion.ConsumedInterfaces.ImageServiceConsumer2,
    "ImageService2",
    vafmodel.MSRAConnectionPointDeploymentType.SOME_IP,
)
adas_demo_app.connect_consumed_interface_to_msra(
    SensorFusion,
    Instances.SensorFusion.ConsumedInterfaces.SteeringAngleServiceConsumer,
    deployment_type=vafmodel.MSRAConnectionPointDeploymentType.SOME_IP,
)
adas_demo_app.connect_consumed_interface_to_msra(
    SensorFusion,
    Instances.SensorFusion.ConsumedInterfaces.VelocityServiceConsumer,
    deployment_type=vafmodel.MSRAConnectionPointDeploymentType.SOME_IP,
)

adas_demo_app.connect_provided_interface_to_msra(
    CollisionDetection,
    Instances.CollisionDetection.ProvidedInterfaces.BrakeServiceProvider,
    deployment_type=vafmodel.MSRAConnectionPointDeploymentType.SOME_IP,
)
```

With this step done, the ADAS demo configuration part is complete. The next step is model and code
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

In first place, the above-created data exchange file from the interface project needs to be imported
to make the model elements from there accessible in the app-module project:

```bash
vaf project import
Enter the path to the exported VAF model JSON file: ../Interfaces/export/Interfaces.json
```

Next step is the configuration of the application module. For that, open the file
`./model/msra_platform.py`. To complete the import from the interface project, uncomment line #6:

``` python
from .imported_models import *
```

The definition of own data types and interfaces is also done in the `./model/msra_platform.py` file
by using elements from the `vafpy` package. Next step is the configuration of the application
module. The configuration of the application module is done completely in this Configuration as Code
file. According to the illustration above, `MsraPlatform` acts as the platform-side for
AdasApplication. For communication with AdasApplication it acts as provider of the earlier defined
`ImageService1`, `ImageService2`´, `VelocityService` and `SteeringAngleService`, and as consumer of
`BrakeService` interfaces.

The configuration for `MsraPlatform`:  
Extend the existing msra_platform app-module with consumed and provided interfaces.

``` python
msra_platform = vafpy.ApplicationModule(name="MsraPlatform", namespace="NsApplicationUnit::NsMsraPlatform")

msra_platform.add_consumed_interface("BrakeServiceConsumer", interfaces.Af.AdasDemoApp.Services.brake_service)
msra_platform.add_provided_interface("ImageServiceProvider1", interfaces.Af.AdasDemoApp.Services.image_service)
msra_platform.add_provided_interface("ImageServiceProvider2", interfaces.Af.AdasDemoApp.Services.image_service)
msra_platform.add_provided_interface("SteeringAngleServiceProvider", interfaces.Af.AdasDemoApp.Services.steering_angle_service)
msra_platform.add_provided_interface("VelocityServiceProvider", interfaces.Af.AdasDemoApp.Services.velocity_service)

periodic_task = vafpy.Task(name="PeriodicTask", period=timedelta(milliseconds=200))
msra_platform.add_task(task=periodic_task)
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

The entry file for the user to add own code is located in `./implementation/src/msra_platform.cpp`.
Some sample code for reference is shipped as part of the container and located in:
`/opt/vaf/Demo/AdasDemo/MSRA/app-centric/test-app`.

The application module project is also ready to be built as library. To do so and in order to check
if added code passes the compiler checks, trigger the
CMake-based build process. 

``` bash
vaf make build
```

Final integration of the application module is done in the **VAF integration project**. The
configuration of the integration project can be done in `./model/vaf/adas_platform.py`.

``` python
adas_platform_app = Executable("adas_platform_app", timedelta(milliseconds=20))

adas_platform_app.add_application_module(MsraPlatform, [])


# connect middleware interfaces
adas_platform_app.connect_provided_interface_to_msra(
    MsraPlatform,
    Instances.MsraPlatform.ProvidedInterfaces.ImageServiceProvider1,
    "ImageService1",
    vafmodel.MSRAConnectionPointDeploymentType.SOME_IP,
    tcp_port = 1025,
    udp_port = 1025
)
adas_platform_app.connect_provided_interface_to_msra(
    MsraPlatform,
    Instances.MsraPlatform.ProvidedInterfaces.ImageServiceProvider2,
    "ImageService2",
    vafmodel.MSRAConnectionPointDeploymentType.SOME_IP,
    tcp_port = 1026,
    udp_port = 1026
)
adas_platform_app.connect_provided_interface_to_msra(
    MsraPlatform,
    Instances.MsraPlatform.ProvidedInterfaces.SteeringAngleServiceProvider,
    deployment_type=vafmodel.MSRAConnectionPointDeploymentType.SOME_IP,
)
adas_platform_app.connect_provided_interface_to_msra(
    MsraPlatform,
    Instances.MsraPlatform.ProvidedInterfaces.VelocityServiceProvider,
    deployment_type=vafmodel.MSRAConnectionPointDeploymentType.SOME_IP,
)

adas_platform_app.connect_consumed_interface_to_msra(
    MsraPlatform,
    Instances.MsraPlatform.ConsumedInterfaces.BrakeServiceConsumer,
    deployment_type=vafmodel.MSRAConnectionPointDeploymentType.SOME_IP,
)
```

With this step done, the AdasPlatform configuration part is complete. The next step is model and
code generation using:

``` bash
vaf model update
vaf project generate
```

Using `--mode prj` or `--mode all` allows to set the scope of this command to either, the current
integration project only (prj), or to this and all related sub-projects (all).

Although the VAF-related code has been generated, the MSRA-related code has not yet been generated
and no ARXML model is yet available. In order to be able to compile the project, a suitable ARXML is
first required, which is necessary for generating the MSRA-related code. Both, the generation of the
ARXML and the code is done using:

``` bash
vaf platform export msra -o ./model/msra
```

To complete the integration project, build and final installation are missing:

``` bash
vaf make preset
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

As you may have already seen, the inter-application communication takes place via SOME/IP.
However, before starting the SomeIp daemon, we still need to adjust the configuration. This 
is because the daemon must be aware of all applications. Therefore we have to adapt the json
configuration in `./build/Release/install/opt/amsr_someipd_daemon/etc/someipd_posix.json`
We need to add the adas platform app to the SomeIp daemon configuration.

``` json
{
  "applications": [
      "../adas_demo_app/etc/someip_config.json",
      "../../../../../../AdasPlatform/build/Release/install/opt/adas_platform_app/etc/someip_config.json"
    ]
}
```

Now you can launch the executables SomeIp daemon, the AdasApplication and the AdasPlatform from 
`./build/Release/install/opt` directory of your working directory.

``` bash
cd AdasApplication/build/Release/install/opt/amsr_someipd_daemon
./bin/amsr_someipd_daemon -c ./etc/someipd_posix.json

cd AdasApplication/build/Release/install/opt/adas_application
./bin/adas_application

cd AdasPlatform/build/Release/install/opt/adas_platform
./bin/adas_platform
```
