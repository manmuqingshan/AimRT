# HelloWorld CPP

This chapter will introduce how to set up a most basic AimRT CPP project through a simple Demo.

This Demo will demonstrate the following basic functionalities:
- Referencing AimRT via source code using CMake FetchContent;
- Writing a basic `Module` based on the AimRT CPP interface;
- Using basic logging functionality;
- Using basic configuration functionality;
- Integrating the `Module` in App mode;
- Compiling the project and running the process to execute the logic in the `Module`.


## STEP1: Ensure Local Environment Meets Requirements

First, ensure your local compilation environment and network environment meet the requirements. For details, refer to the requirements in [Reference and Installation (CPP)](installation_cpp.md).


## STEP2: Create Directory Structure and Add Basic Files

Create files according to the following directory structure:
```
├── CMakeLists.txt
├── cmake
│   └── GetAimRT.cmake
└── src
    ├── CMakeLists.txt
    ├── install
    │   └── cfg
    │       └── helloworld_cfg.yaml
    ├── module
    │   └── helloworld_module
    │       ├── CMakeLists.txt
    │       ├── helloworld_module.cc
    │       └── helloworld_module.h
    └── app
        └── helloworld_app
            ├── CMakeLists.txt
            └── main.cc
```

Please note that this is just a reference directory structure and not mandatory. However, it is recommended to create separate folders for the following areas when setting up your own project:
- install: Stores deployment configurations, startup scripts, etc.;
- module: Stores business logic code;
- app: In App mode, stores the main function where business modules are registered;
- pkg: In Pkg mode, stores the entry methods for pkg dynamic libraries where business modules are registered;

### File 1: /CMakeLists.txt
Root CMake file for building the project.
```cmake
cmake_minimum_required(VERSION 3.24)

project(helloworld LANGUAGES C CXX)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

set(CMAKE_POLICY_DEFAULT_CMP0077 NEW)

if(CMAKE_CXX_COMPILER_ID MATCHES "GNU|Clang")
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fPIC")
  set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fPIC")
endif()

include(cmake/GetAimRT.cmake)

add_subdirectory(src)
```

### File 2: /cmake/GetAimRT.cmake
This file is used to fetch AimRT. Note that you need to change the `GIT_TAG` version to the one you want to reference:
```cmake
include(FetchContent)

FetchContent_Declare(
  aimrt
  GIT_REPOSITORY https://github.com/AimRT/aimrt.git
  GIT_TAG v1.x.x)

FetchContent_GetProperties(aimrt)

if(NOT aimrt_POPULATED)
  FetchContent_MakeAvailable(aimrt)
endif()
```

### File 3: /src/CMakeLists.txt
References various subdirectories under src.
```cmake
add_subdirectory(module/helloworld_module)
add_subdirectory(app/helloworld_app)
```

### File 4: /src/module/helloworld_module/CMakeLists.txt
Creates the `helloworld_module` static library.
```cmake
file(GLOB_RECURSE src ${CMAKE_CURRENT_SOURCE_DIR}/*.cc)

add_library(helloworld_module STATIC)
add_library(helloworld::helloworld_module ALIAS helloworld_module)

target_sources(helloworld_module PRIVATE ${src})

target_include_directories(
  helloworld_module
  PUBLIC ${CMAKE_CURRENT_SOURCE_DIR}/..)

# 引用aimrt_module_cpp_interface
target_link_libraries(
  helloworld_module
  PRIVATE yaml-cpp::yaml-cpp
  PUBLIC aimrt::interface::aimrt_module_cpp_interface)
```

### File 5: /src/app/helloworld_app/CMakeLists.txt
Creates the `helloworld_app` executable.
```cmake
file(GLOB_RECURSE src ${CMAKE_CURRENT_SOURCE_DIR}/*.cc)

add_executable(helloworld_app)

target_sources(helloworld_app PRIVATE ${src})

target_include_directories(
  helloworld_app
  PRIVATE ${CMAKE_CURRENT_SOURCE_DIR})

target_link_libraries(
  helloworld_app
  PRIVATE aimrt::runtime::core
          helloworld::helloworld_module)
```

## STEP3: Write Business Code

Business logic is mainly carried by the Module. Refer to the following code to implement a simple Module that parses the input configuration file and prints some simple logs.

### File 6: /src/module/helloworld_module/helloworld_module.h
```cpp
#pragma once

#include "aimrt_module_cpp_interface/module_base.h"

class HelloWorldModule : public aimrt::ModuleBase {
 public:
  HelloWorldModule() = default;
  ~HelloWorldModule() override = default;

  aimrt::ModuleInfo Info() const override {
    return aimrt::ModuleInfo{.name = "HelloWorldModule"};
  }

  bool Initialize(aimrt::CoreRef core) override;
  bool Start() override;
  void Shutdown() override;

 private:
  aimrt::CoreRef core_;
};
```

### File 7: /src/module/helloworld_module/helloworld_module.cc
```cpp
#include "helloworld_module/helloworld_module.h"

#include "yaml-cpp/yaml.h"

bool HelloWorldModule::Initialize(aimrt::CoreRef core) {
  // Save aimrt framework handle
  core_ = core;

  // Log
  AIMRT_HL_INFO(core_.GetLogger(), "Init.");

  try {
    // Read cfg
    auto file_path = core_.GetConfigurator().GetConfigFilePath();
    if (!file_path.empty()) {
      YAML::Node cfg_node = YAML::LoadFile(file_path.data());
      for (const auto& itr : cfg_node) {
        std::string k = itr.first.as<std::string>();
        std::string v = itr.second.as<std::string>();
        AIMRT_HL_INFO(core_.GetLogger(), "cfg [{} : {}]", k, v);
      }
    }

  } catch (const std::exception& e) {
    AIMRT_HL_ERROR(core_.GetLogger(), "Init failed, {}", e.what());
    return false;
  }

  AIMRT_HL_INFO(core_.GetLogger(), "Init succeeded.");

  return true;
}

bool HelloWorldModule::Start() {
  AIMRT_HL_INFO(core_.GetLogger(), "Start succeeded.");
  return true;
}

void HelloWorldModule::Shutdown() {
  AIMRT_HL_INFO(core_.GetLogger(), "Shutdown succeeded.");
}
```

## STEP4: Determine Deployment Plan and Configuration

We use App mode, manually write the Main function, and register the HelloWorldModule into the AimRT framework via hardcoding. Then, write a configuration to determine some runtime details.

### File 8: /src/app/helloworld_app/main.cc
In the following example main function, we capture the kill signal to achieve graceful exit.
```cpp
#include <csignal>
#include <iostream>

#include "core/aimrt_core.h"
#include "helloworld_module/helloworld_module.h"

using namespace aimrt::runtime::core;

AimRTCore *global_core_ptr_ = nullptr;

void SignalHandler(int sig) {
  if (global_core_ptr_ && (sig == SIGINT || sig == SIGTERM)) {
    global_core_ptr_->Shutdown();
    return;
  }
  raise(sig);
};

int32_t main(int32_t argc, char **argv) {
  signal(SIGINT, SignalHandler);
  signal(SIGTERM, SignalHandler);

  std::cout << "AimRT start." << std::endl;

  try {
    AimRTCore core;
    global_core_ptr_ = &core;

    // register module
    HelloWorldModule helloworld_module;
    core.GetModuleManager().RegisterModule(helloworld_module.NativeHandle());

    AimRTCore::Options options;
    options.cfg_file_path = argv[1];
    core.Initialize(options);

    core.Start();

    core.Shutdown();

    global_core_ptr_ = nullptr;
  } catch (const std::exception &e) {
    std::cout << "AimRT run with exception and exit. " << e.what() << std::endl;
    return -1;
  }

  std::cout << "AimRT exit." << std::endl;
  return 0;
}
```

### File 9: /src/install/cfg/helloworld_cfg.yaml
The following is a simple example configuration file. Other contents in this configuration file will be introduced in subsequent chapters. Here, focus on two parts:
- `aimrt.log` node: Specifies some details about logging.
- `HelloWorldModule` node: Configuration for `HelloWorldModule`, which can be read in the module.

```yaml
aimrt:
  log: # log配置
    core_lvl: INFO # 内核日志等级，可选项：Trace/Debug/Info/Warn/Error/Fatal/Off，不区分大小写
    backends: # 日志后端
      - type: console # 控制台日志

# 模块自定义配置，框架会为每个模块生成临时配置文件，开发者通过Configurator接口获取该配置文件路径
HelloWorldModule:
  key1: val1
  key2: val2
```

## STEP5: Start and Test

After completing the code, execute the following commands on Linux to compile:
```shell
# cd to root path of project
cmake -B build
cd build
make -j
```

After compilation, copy the generated executable `helloworld_app` and configuration file `helloworld_cfg.yaml` to a directory, then execute the following command to run the process and observe the printed logs:
```shell
./helloworld_app ./helloworld_cfg.yaml
```