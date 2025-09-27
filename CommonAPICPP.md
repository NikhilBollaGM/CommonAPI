# CommonAPI C++ Specification

**Document Title:** CommonAPI C++ Specification  
**Website:** http://projects.genivi.org/commonapi/  
**Version:** [Version]  
**Revision Date:** [Revision Date]  

This is the specification for *Common API [Version]* released at [Revision Date].

## Copyright and License

Copyright (C) 2015, BMW AG  
Copyright (C) 2015, GENIVI Alliance, Inc.

This file is part of the GENIVI IPC Common API C++ project.

Contributions are licensed to the GENIVI Alliance under one or more Contribution License Agreements or MPL 2.0.

## Introduction

IPC Common API is a C++ abstraction framework for *Interprocess Communication* (IPC). It is designed to be neutral to IPC implementations and can be used with any IPC mechanism if a middleware-specific *CommonAPI binding* is provided.

Applications (clients and servers) developed against CommonAPI can be linked with different CommonAPI *backends* without changes to the application code. This allows components developed for one IPC system to be deployed on another system by simply exchanging the backend without recompilation.

Interface definitions are created using [Franca IDL](http://code.google.com/a/eclipselabs.org/p/franca/), the preferred IDL solution by [GENIVI](http://www.genivi.org/).

CommonAPI is open source, split into runtime code for the target system and code generation tooling for development systems.

## General Design

### Basic Assumptions

- Applications use the client-server communication paradigm.
- The C++ API is generated from Franca IDL specifications.
- CommonAPI specifies an API, not a concrete IPC mechanism; bindings must be developed for specific middleware.
- CommonAPI aims to be platform-independent, targeting GNU C++ compiler versions ≤ 4.4.2. Supported compilers are listed in the NEWS file.

### Deployment

Deployment parameters (e.g., QoS, array lengths, endianness) are specified in middleware-specific deployment models (`.depl` files). These parameters must not affect the generated API to ensure application portability across backends.

### Basic Parts of CommonAPI

1. **Franca-based Part:** Generated from Franca IDL (data types, interfaces, methods, broadcasts, attributes).
2. **Runtime Features:** CommonAPI library functions (service discovery, connection management, address handling) and base classes.

## Franca-Based Part

### Namespaces

The base namespace is `CommonAPI`. Application namespaces are derived from the Franca package name and interface version.

**Example:**
```franca
package example.user
interface Test { version { major 1 minor 0 } }
```

```cpp
namespace v1_0 {
namespace example {
namespace user { }
} }
```

### Data Types

#### Primitive Types

| Franca Type | C++ Type        | Notes                                       |
|-------------|-----------------|---------------------------------------------|
| UInt8       | uint8_t         | Unsigned 8-bit integer (0–255)              |
| Int8        | int8_t          | Signed 8-bit integer (-128–127)             |
| UInt16      | uint16_t        | Unsigned 16-bit integer (0–65535)           |
| Int16       | int16_t         | Signed 16-bit integer (-32768–32767)        |
| UInt32      | uint32_t        | Unsigned 32-bit integer (0–4294967295)      |
| Int32       | int32_t         | Signed 32-bit integer (-2147483648–2147483647) |
| UInt64      | uint64_t        | Unsigned 64-bit integer                     |
| Int64       | int64_t         | Signed 64-bit integer                       |
| Boolean     | bool            | Boolean value (false/true)                  |
| Float       | float           | Floating point number (4 bytes)             |
| Double      | double          | Double precision floating point (8 bytes)   |
| String      | std::string     | Character string (UTF-8)                    |
| ByteBuffer  | std::vector<uint8_t> | Buffer of bytes (BLOB)                 |

#### Arrays

Franca array types map to `std::vector<T>`. Explicit arrays become typedefs, while implicit arrays are generated as `std::vector<T>` where needed.

#### Structures

Franca structs map to C++ structs inheriting from `CommonAPI::Struct`.

**Example:**
```franca
struct TestStruct {
    UInt16 uintValue
    String stringValue
}
```

```cpp
struct TestStruct : CommonAPI::Struct<uint16_t, std::string> {
    TestStruct() {}
    TestStruct(const uint16_t &_uintValue, const std::string &_stringValue) {
        std::get<0>(values_) = _uintValue;
        std::get<1>(values_) = _stringValue;
    }
    
    inline const uint16_t &getUintValue() const { return std::get<0>(values_); }
    inline void setUintValue(const uint16_t &_value) { std::get<0>(values_) = _value; }
    inline const std::string &getStringValue() const { return std::get<1>(values_); }
    inline void setStringValue(const std::string &_value) { std::get<1>(values_) = _value; }
    bool operator==(const TestStruct &_other) const;
    inline bool operator!=(const TestStruct &_other) const { return !((*this) == _other); }
};
```

The base `Struct` class:
```cpp
template<typename... _Types>
struct Struct {
    std::tuple<_Types...> values_;
};
```

Polymorphic structs derive from `PolymorphicStruct`:
```cpp
struct PolymorphicStruct {
    virtual const Serial getSerial() const = 0;
};
```

#### Enumerations

Franca enumerations map to C++ structs inheriting from `CommonAPI::Enumeration`. The default backing type is `uint8_t`.

**Example:**
```franca
enumeration MyEnum {
    E_UNKNOWN = "0x00"
}

enumeration MyEnumExtended extends MyEnum {
    E_NEW = "0x01"
}
```

```cpp
struct MyEnum : CommonAPI::Enumeration<int8_t> {
    MyEnum() = default;
    MyEnum(const int8_t &_value) : CommonAPI::Enumeration<int8_t>(_value) {}
    static const int8_t E_UNKNOWN = 0;
};

struct MyEnumExtended : MyEnum {
    MyEnumExtended() = default;
    MyEnumExtended(const int8_t &_value) : MyEnum(_value) {}
    static const int8_t E_NEW = 1;
};
```

Enumeration base class:
```cpp
template <typename _Base>
struct Enumeration {
    Enumeration() = default;
    Enumeration(const _Base &_value) : value_(_value) {}
    
    inline Enumeration &operator=(const _Base &_value) {
        value_ = _value;
        return (*this);
    }
    inline operator const _Base() const { return value_; }
    inline bool operator==(const Enumeration<_Base> &_other) const {
        return value_ == _other.value_;
    }
    inline bool operator!=(const Enumeration<_Base> &_other) const {
        return value_ != _other.value_;
    }
    
    _Base value_;
};
```

#### Maps

Franca maps use `std::unordered_map<K,V>` for efficiency.

**Example:**
```franca
map MyMap {
    UInt32 to String
}
```

```cpp
typedef std::unordered_map<uint32_t, std::string> MyMap;
```

#### Unions

Franca unions use `CommonAPI::Variant`.

**Example:**
```franca
union MyUnion {
    UInt32 MyUInt
    String MyString
}
```

```cpp
typedef Variant<uint32_t, std::string> MyUnion;
```

Usage:
```cpp
MyUnion union = 5;
MyUnion stringUnion("my String");

int a = union.get<uint32_t>(); // Works
std::string b = union.get<std::string>(); // Throws exception

bool contained = union.isType<uint32_t>(); // True
```

#### Type Aliases

Franca typedefs map directly to C++ `typedef`.

#### Type Collections

Franca type collections generate a C++ struct. Anonymous collections use `__Anonymous__`.

**Example:**
```franca
package example.user
typeCollection {
    typedef a is Int16
}
```

```cpp
namespace example {
namespace user {

struct __Anonymous__ {
    typedef int16_t a;
    
    static inline const char* getTypeCollectionName() {
        static const char* typeCollectionName = "example.user.__Anonymous__";
        return typeCollectionName;
    }
};

} // namespace user
} // namespace example
```

### Interfaces

#### Basics

For each Franca interface, a class is generated with `getInterfaceName` and `getInterfaceVersion` methods.

**Example:**
```franca
package commonapi.examples
interface ExampleInterface {
    version { major 1 minor 0 }
}
```

```cpp
namespace v1_0 {
namespace commonapi {
namespace examples {

class ExampleInterface {
public:
    virtual ~ExampleInterface() { }
    static inline const char* getInterface();
    static inline CommonAPI::Version getInterfaceVersion();
};

const char* ExampleInterface::getInterface() {
    return ("commonapi.examples.ExampleInterface");
}

CommonAPI::Version ExampleInterface::getInterfaceVersion() {
    return CommonAPI::Version(1, 0);
}

} // namespace examples
} // namespace commonapi
} // namespace v1_0
```

Version structure:
```cpp
namespace CommonAPI {
struct Version {
    Version() = default;
    Version(const uint32_t &majorValue, const uint32_t &minorValue)
        : Major(majorValue), Minor(minorValue) { }
    uint32_t Major;
    uint32_t Minor;
};
} // namespace CommonAPI
```

Generated files for each interface:
- `ExampleInterface.hpp` - Common header
- `ExampleInterfaceProxy.hpp` - Proxy class
- `ExampleInterfaceProxyBase.hpp` - Proxy base class
- `ExampleInterfaceStub.hpp` - Stub class

#### Methods

Methods can have in/out parameters and error enumerations. `fireAndForget` methods have no out parameters. Broadcasts have only out parameters and can be `selective`.

**CallStatus Enumeration:**
```cpp
enum class CallStatus {
    SUCCESS,
    OUT_OF_MEMORY,
    NOT_AVAILABLE,
    CONNECTION_FAILED,
    REMOTE_ERROR,
    UNKNOWN,
    INVALID_VALUE,
    SUBSCRIPTION_REFUSED,
    SERIALIZATION_ERROR
};
```

**CallInfo Structure:**
```cpp
struct CallInfo {
    CallInfo() : timeout_(DEFAULT_SEND_TIMEOUT_MS), sender_(0) { }
    CallInfo(Timeout_t _timeout) : timeout_(_timeout), sender_(0) { }
    
    Timeout_t timeout_;
    Sender_t sender_;
};
```

**Example Interface:**
```franca
package commonapi.examples
interface ExampleInterface {
    version { major 1 minor 0 }
    
    method getProperty {
        in { UInt32 ID }
        out { String Property }
        error { OK NOT_OK }
    }
    
    method newMessage fireAndForget {
        in { String MessageName }
    }
    
    broadcast signalChanged {
        out { UInt32 NewValue }
    }
    
    broadcast signalSpecial selective {
        out { UInt32 MyValue }
    }
}
```

**Proxy Side:**
```cpp
// Synchronous call
virtual void getProperty(
    const uint32_t &_ID,
    CommonAPI::CallStatus &_status,
    ExampleInterface::getPropertyError &_error,
    std::string &_Property,
    const CommonAPI::CallInfo *_info = nullptr);

// Asynchronous call
virtual std::future<CommonAPI::CallStatus> getPropertyAsync(
    const uint32_t &_ID,
    GetPropertyAsyncCallback _callback,
    const CommonAPI::CallInfo *_info = nullptr);

// Fire & Forget
virtual void newMessage(const std::string &_MessageName, CommonAPI::CallStatus &_status);

// Broadcast events
virtual SignalChangedEvent& getSignalChangedEvent() {
    return delegate_->getSignalChangedEvent();
}

virtual SignalSpecialSelectiveEvent& getSignalSpecialSelectiveEvent() {
    return delegate_->getSignalSpecialSelectiveEvent();
}
```

**Stub Side:**
```cpp
// Method implementations
virtual void getProperty(
    const std::shared_ptr<CommonAPI::ClientId> _client,
    uint32_t _ID,
    getPropertyReply_t _reply) = 0;

virtual void newMessage(
    const std::shared_ptr<CommonAPI::ClientId> _client,
    std::string _MessageName) = 0;

// Reply callback type
typedef std::function<void (ExampleInterface::getPropertyError _error, 
                           std::string _Property)> getPropertyReply_t;

// Broadcast methods
virtual void fireSignalChangedEvent(const uint32_t &_NewValue) = 0;
virtual void fireSignalSpecialSelective(
    const uint32_t &_MyValue,
    const std::shared_ptr<CommonAPI::ClientIdList> _receivers = nullptr) = 0;

// Selective broadcast hooks
virtual std::shared_ptr<CommonAPI::ClientIdList> const 
    getSubscribersForSignalSpecialSelective() = 0;
    
virtual void onSignalSpecialSelectiveSubscriptionChanged(
    const std::shared_ptr<CommonAPI::ClientId> _client,
    const CommonAPI::SelectiveBroadcastSubscriptionEvent _event) = 0;
    
virtual bool onSignalSpecialSelectiveSubscriptionRequested(
    const std::shared_ptr<CommonAPI::ClientId> _client) = 0;
```

**ClientId Class:**
```cpp
class ClientId {
public:
    virtual ~ClientId() { }
    virtual bool operator==(ClientId& clientIdToCompare) = 0;
    virtual std::size_t hashCode() = 0;
};
```

#### Attributes

Attributes can have `noSubscriptions` and `readonly` flags. Observable attributes provide `ChangedEvent` for updates.

**Stub Side Attribute Handling:**
```cpp
class ExampleInterfaceStub
    : public virtual CommonAPI::Stub<ExampleInterfaceStubAdapter, 
                                    ExampleInterfaceStubRemoteEvent> {
public:
    virtual const uint32_t &getAAttribute(const std::shared_ptr<CommonAPI::ClientId> _client) = 0;
};

class ExampleInterfaceStubRemoteEvent {
public:
    virtual ~ExampleInterfaceStubRemoteEvent() { }
    virtual bool onRemoteSetAAttribute(const std::shared_ptr<CommonAPI::ClientId> _client, 
                                      uint32_t a) = 0;
    virtual void onRemoteAAttributeChanged() = 0;
};

class ExampleInterfaceStubAdapter
    : virtual public CommonAPI::StubAdapter, 
      public ExampleInterface {
public:
    virtual void fireAAttributeChanged(const uint32_t& a) = 0;
};
```

**Attribute Extensions:** Custom extensions can be implemented by deriving from `AttributeExtension` base class.

#### Events

Events provide asynchronous interfaces for broadcasts and attribute changes.

**Event Class:**
```cpp
template<typename... _Arguments>
class Event {
public:
    typedef uint32_t Subscription;
    typedef std::function<void(const _Arguments&...)> Listener;
    
    Subscription subscribe(Listener listener);
    void unsubscribe(Subscription subscription);
};
```

## Runtime

### Runtime Interface

The Runtime class loads middleware libraries based on configuration.

**Key Methods:**

```cpp
class Runtime {
public:
    // Access runtime instance
    static std::shared_ptr<Runtime> get();
    
    // Configuration properties
    static std::string getProperty(const std::string &_name);
    static void setProperty(const std::string &_name, const std::string &_value);
    
    // Build proxy without mainloop integration
    template< template<typename ...> class _ProxyClass, typename ... _AttributeExtensions >
    std::shared_ptr< _ProxyClass<_AttributeExtensions...> > buildProxy(
        const std::string &_domain,
        const std::string &_instance,
        const ConnectionId_t &_connectionId = DEFAULT_CONNECTION_ID);
    
    // Build proxy with mainloop integration
    template<template<typename ...> class _ProxyClass, typename ... _AttributeExtensions >
    std::shared_ptr< _ProxyClass<_AttributeExtensions...> > buildProxy(
        const std::string &_domain,
        const std::string &_instance,
        std::shared_ptr<MainLoopContext> _context);
    
    // Build proxy with default attribute extension
    template <template<typename ...> class _ProxyClass, 
              template<typename> class _AttributeExtension>
    std::shared_ptr<typename DefaultAttributeProxyHelper<_ProxyClass, 
                        _AttributeExtension>::class_t>
    buildProxyWithDefaultAttributeExtension(
        const std::string &_domain,
        const std::string &_instance,
        const ConnectionId_t &_connectionId = DEFAULT_CONNECTION_ID);
    
    // Register service
    template<typename _Stub>
    bool registerService(
        const std::string &_domain,
        const std::string &_instance,
        std::shared_ptr<_Stub> _service,
        const ConnectionId_t &_connectionId = DEFAULT_CONNECTION_ID);
    
    // Unregister service
    bool unregisterService(
        const std::string &_domain,
        const std::string &_interface,
        const std::string &_instance);
};
```

### Proxy Interface

```cpp
class Proxy {
public:
    virtual ~Proxy();
    const Address &getAddress() const;
    std::future<void> getCompletionFuture();
    virtual bool isAvailable() const = 0;
    virtual bool isAvailableBlocking() const = 0;
    virtual ProxyStatusEvent& getProxyStatusEvent() = 0;

protected:
    Address address_;
    std::promise<void> completed_;
};
```

**Address Class:**
```cpp
class Address {
public:
    bool operator<(const Address &_other) const;
    std::string getAddress() const;
    void setAddress(const std::string &_address);
    const std::string &getDomain() const;
    void setDomain(const std::string &_domain);
    const std::string &getInterface() const;
    void setInterface(const std::string &_interface);
    const std::string &getInstance() const;
    void setInstance(const std::string &_instance);

private:
    std::string domain_;
    std::string interface_;
    std::string instance_;
};
```

CommonAPI addresses consist of:
- **Interface Name:** Fully qualified Franca interface name
- **Instance Name:** Arbitrary instance identifier
- **Domain:** Arbitrary domain name (default: "local")

### Configuring CommonAPI

Configuration file: `commonapi.ini` (searched in: executable directory, `COMMONAPI_CONFIG` environment variable, `/etc`)

**Configuration Sections:**
- `logging`: Internal logging settings
- `default`: Default binding and call timeout
- `proxy`: Mapping of addresses to libraries for proxies
- `stub`: Mapping of addresses to libraries for stubs

### Mainloop Integration

CommonAPI supports both multithreaded execution and single-threaded mainloop integration.

**MainLoopContext Components:**
- `DispatchSource`: Work ready to be dispatched (not file descriptor related)
- `Watch`: File descriptor-related work
- `Timeout`: Timeout-related work

Binding developers must provide implementations for these classes for mainloop integration.

## Glossary

**BLOB:** Binary Large Object  
**IDL:** Interface Description Language  
**IPC:** Interprocess Communication  
**GENIVI:** Non-profit industry alliance for In-Vehicle Infotainment open-source platform

---

*End of CommonAPI C++ Specification*
```
