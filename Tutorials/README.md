This folder provides a set of examples to ease your start with the **Vector Application Framework**. Each
demo comes with a comprehensive README that guides you through the steps. Please find a short
description of the provided examples below:

## Hello VAF examples
The following two "Hello, world!" style examples can help to get a first overview when getting
started with the Application Framework. The first demo illustrates modular concept and workflow, the
extended version gives an overview of the binding variants that are supported when using MICROSAR
Adaptive as communication middleware.
* [Hello world basic](https://github.com/vectorgrp/application-framework/tree/main/Tutorials/HelloVaf/HelloWorld)
* [Hello world extended](https://github.com/vectorgrp/application-framework/tree/main/Tutorials/HelloVaf/HelloExtended)

## Application-centric examples
Application-centric means to start from the perspective of an application developer. That means, the
lower-layer middleware of the application is not known. Still, design artifacts like interface or
datatype descriptions might be available. They can be imported and used as a starting point.
Modifications and own definitions on top of the imported artifacts are possible with the
Configuration as Code solution of the framework. The application-centric workflow matches very well
with a prototyping situation.

* [COVESA Vehicle Signal Specification (VSS) support](https://github.com/vectorgrp/application-framework/tree/main/Tutorials/VssDemo)
* [HVAC demo, prototyping with MICROSAR Adaptive](https://github.com/vectorgrp/application-framework/tree/main/Tutorials/HvacDemo/MSRA/app-centric)
* [ADAS demo, prototyping with SIL Kit](https://github.com/vectorgrp/application-framework/tree/main/Tutorials/AdasDemo/SilKit/app-centric)
* [ADAS demo, prototyping with MICROSAR Adaptive](https://github.com/vectorgrp/application-framework/tree/main/Tutorials/AdasDemo/MSRA/app-centric)

## Platform-centric examples
Platform-centric means to start from an existing platform model. Especially, AUTOSAR Adaptive models
are of interest here, as they not only cover design information (datatype and interface definitions)
but also deployment details (ports, executables, and more). From there, the outer surface for the
application is known from the start. In consequence, a tight coupling between application and
lower-layer middleware is possible and wanted.

* [HVAC demo, based on a given AUTOSAR Adaptive model](https://github.com/vectorgrp/application-framework/tree/main/Tutorials/HvacDemo/MSRA/platform-centric)
* [ADAS demo, based on a given AUTOSAR Adaptive model](https://github.com/vectorgrp/application-framework/tree/main/Tutorials/AdasDemo/MSRA/platform-centric)

## Integration-centric examples
The integration-centric example, in essence, is a combination of the two above-mentioned cases. In
other words, it is about combining application-centric app-modules with platform-centric ones that
serve as adapter or gateway respectively and by that facilitate the integration.

* [HVAC demo, integration-centric](https://github.com/vectorgrp/application-framework/tree/main/Tutorials/HvacDemo/MSRA/integration-centric)
