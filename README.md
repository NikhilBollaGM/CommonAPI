

## 🚀 Introduction

This document serves as a supplement to the main **CommonAPI tutorial**, focusing specifically on information related to the **SOME/IP binding**. It is assumed that you have read the base CommonAPI tutorial first.

-----

## 🛠️ Integration Guide for CommonAPI Users

The instructions below are primarily for **Linux platforms** but also include a dedicated section for **Windows**.

### Requirements

CommonAPI utilizes many **C++11 features** (like variadic templates, `std::bind`, `std::function`).

  * **Compiler:** Ensure your target platform's compiler supports C++11 (e.g., **gcc 4.8** or newer).
  * **Build System:** **CMake** is required for building CommonAPI.
  * **IDE:** Do not use versions of Eclipse earlier than **Luna**.
  * **Code Generation:** **Maven 3** or higher is necessary for the build toolchain of the code generators.
  * **SOME/IP:** The CommonAPI SOME/IP binding requires the **vsomeip** implementation.

### Dependencies

The CommonAPI-SomeIP Runtime library depends on the **vsomeip library**.

### Building the CommonAPI SOME/IP Runtime Library

#### Command-line (Linux)

You must have the **vsomeip library** available before building.

1.  **Navigate to the source directory:**
    ```
    $ cd common-api-someip-runtime
    $ mkdir build
    ```
2.  **Run CMake:**
    ```
    $ cmake -D USE_INSTALLED_COMMONAPI=ON -D CMAKE_INSTALL_PREFIX=/usr/local ..
    ```
      * *Note:* Use `-D USE_INSTALLED_COMMONAPI=OFF` if you do not plan to install it.
3.  **Build and Install (Optional):**
    ```
    $ make
    $ make install
    ```

| CMake Option | Description |
| :--- | :--- |
| `-DBUILD_SHARED_LIBS=OFF` | Generates a makefile for building a **static** CommonAPI library (default is shared). |
| `-DCMAKE_BUILD_TYPE=Release` | Generates a makefile for building the **release** version (default is debug). |
| `-DCMAKE_INSTALL_PREFIX=/path` | Changes the installation directory (default is `/usr/local`). |

| Make Target | Description |
| :--- | :--- |
| `make all` | Compiles and links CommonAPI. |
| `make clean` | Deletes binaries (not CMake generated files). |
| `make install` | Copies libraries and header files to the installation prefix. |
| `make DESTDIR=<dir> install` | Influences the destination directory for installation. |

#### Eclipse

Follow the instructions provided in the general **CommonAPI User Guide**.

-----

## ⚙️ Compile Tools and Glue Code

### Compile Tools (SOME/IP Generator)

The SOME/IP code generator is built using **Maven** from the command-line:

```
$ cd CommonAPI-Tools/org.genivi.commonapi.someip.releng
$ mvn clean verify -DCOREPATH=< path to your CommonAPI-Tools dir> -Dtarget.id=org.genivi.commonapi.someip.target
```

The resulting command-line generators are archived in `org.genivi.commonapi.someip.cli.product/target/products/commonapi_someip_generator.zip`.

### Build SomeIP Glue Code

The glue code library contains the generated binding-specific code.

1.  Generate CommonAPI code from `.fidl` files and CommonAPI-SomeIP code from **`.fdepl`** files.
2.  Create an out-of-source build directory.
3.  Call `cmake` with the required parameters, including the paths to the generated tools and dependencies.

**Key CMake Parameters for Glue Code:**

| Parameter | Description |
| :--- | :--- |
| `+COMMONAPI_SOMEIP_TOOL_GENERATOR+` | Full path to the SomeIP Code generator executable. |
| `+COMMONAPI_TOOL_GENERATOR+` | Full path to the Core Code generator executable. |
| `+COMMONAPI_DIR+` | Path to the CommonAPI build directory. |
| `+CommonAPI-SomeIP_DIR+` | Path to the CommonAPI-SomeIP build directory. |
| `+vsomeip_DIR+` | Path to the vSomeIP build directory. |

-----

## 📝 Project Setup and Configuration

### Configuration File (`commonapi-someip.ini`)

CommonAPI-SomeIP is configured via an **ini-file**, named `commonapi-someip.ini` by default.

**Lookup Order (Highest Priority First):**

1.  Directory of the **current executable**.
2.  Directory specified by the environment variable **`COMMONAPI_SOMEIP_CONFIG`**.
3.  Global default directory **`/etc`**.

#### Address Translation Sections

These sections map CommonAPI addresses to SOME/IP addresses (Service ID and Instance ID).

| Parameter | SOME/IP Mapping |
| :--- | :--- |
| `service` | Service Identifier (e.g., `0x1234`) |
| `instance` | Instance Identifier (e.g., `0x5678`) |
| `major` | Major Version |
| `minor` | Minor Version |

**Example:**

```
[local:de.ABC:v1_1:de.app1]
service=0x1234
instance=0x5678
major=1
minor=2
```

### Deployment (`.fdepl` file)

Since there is no predefined translation, you must specify the mapping from CommonAPI to SOME/IP using a deployment file (`.fdepl`).

**Key Deployment Rules:**

  * **Service ID:** Specified via `SomeIpServiceID`.
  * **Method IDs:** (`SomeIpMethodID`, `SomeIpGetterID`, `SomeIpSetterID`) must be unique within an interface and its extensions.
      * **Range:** 1 to 32767.
  * **Event IDs:** (`SomeIpNotifierID`, `SomeIpEventID`) must be unique within an interface.
      * **Range:** 32769 to 65534.
  * **Event Groups:** (`SomeIpEventGroups`) must be at least 1. Every broadcast or attribute notifier must belong to at least one event group.

**Important Unchecked Rules (User Responsibility):**

  * The combination of **`SomeIpServiceID`** and **`SomeIpInstanceID`** must be **unique** in the SOME/IP network.
  * Do not deploy two instances of the same interface on the **same port**.

-----

## 💻 Windows Support

### Building on Windows (Visual Studio 2013)

1.  **Build vsomeip:**
      * Requires **boost 1.54 or higher** (log, system, and thread libraries).
      * Use CMake to create a Visual Studio 2013 solution in a `build` directory within the vsomeip source.
      * Open the solution, adapt boost paths, and build.
2.  **Build CommonAPI:**
      * Use CMake in a `build` directory within the CommonAPI source to create a Visual Studio 2013 solution.
      * Open the solution and build.
3.  **Build CommonAPI-SomeIP:**
      * Use CMake in a `build` directory within the CommonAPI-SomeIP source to create a Visual Studio 2013 solution.
      * Open the solution and build.

