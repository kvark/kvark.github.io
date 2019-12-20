---
layout: post
title:  "Implementing WebGPU in Gecko"
date:   2019-12-10 11:11:11 -0500
categories: web gpu gecko
---

As WebGPU specification is approaching a usable shape, we are working on prototype implementations in both Gecko and Servo. If you ever wondered about what it's like to implement a complex Web API in Gecko, this article is for you. It's focused on the plumbing: what the parts of implementation are, and how they fit together. We are going to start with the user-facing API (JS on the Web) and go down the shaft till we reach the GPU.

## WebIDL API

JavaScript code uses interfaces and calls methods described by the WebIDL spec. Our working group generates it from a [Bikeshed](https://github.com/gpuweb/gpuweb/blob/402b69138fbedf4a3c9c85cd1bf7e1cc27c1b34e/spec/index.bs) file, which we work on by filing Github PRs. You can see the rendered specification [here](https://gpuweb.github.io/gpuweb/).

WebIDL [definition](https://searchfox.org/mozilla-central/rev/ac43d7e69a435ede789827f20ab309da6f04c09b/dom/webidl/WebGPU.webidl#109) of an interface looks like this:
```webidl
interface GPUDevice {
    [NewObject]
    GPUBuffer createBuffer(GPUBufferDescriptor descriptor);
};
```

When Gecko's WebIDL binding generator processes this interace, it generates a lot of JS interop boilerplate in "dom/WebGPUBinding.h":
```cpp
namespace GPUDevice_Binding {
  typedef mozilla::webgpu::Device NativeType;
  extern const NativePropertyHooks sNativePropertyHooks[];

  bool ConstructorEnabled(JSContext* aCx, JS::Handle<JSObject*> aObj);
  const JSClass* GetJSClass();
  bool Wrap(JSContext* aCx, mozilla::webgpu::Device* aObject, nsWrapperCache* aCache, JS::Handle<JSObject*> aGivenProto, JS::MutableHandle<JSObject*> aReflector);
  template <class T>
  inline JSObject* Wrap(JSContext* aCx, T* aObject, JS::Handle<JSObject*> aGivenProto) { ... }
  void CreateInterfaceObjects(JSContext* aCx, JS::Handle<JSObject*> aGlobal, ProtoAndIfaceCache& aProtoAndIfaceCache, bool aDefineOnGlobal);
  inline JS::Handle<JSObject*> GetProtoObjectHandle(JSContext* aCx) { ... }
  inline JS::Handle<JSObject*> GetConstructorObjectHandle(JSContext* aCx, bool aDefineOnGlobal = true) { ... }
  JSObject* GetConstructorObject(JSContext* aCx);
} // namespace GPUDevice_Binding
```

## C++ DOM bindings

In order to implement WebIDL interfaces, we are creating a C++ class with the name corresponding to the interface. The class needs to have methods implementing the WebIDL, following specific mapping between WebIDL and C++:

```cpp
class Device final : public DOMEventTargetHelper {
 public:
  NS_DECL_ISUPPORTS_INHERITED
  NS_DECL_CYCLE_COLLECTION_CLASS_INHERITED(Device, DOMEventTargetHelper)
  GPU_DECL_JS_WRAP(Device)

 private:
  Device() = delete;
  virtual ~Device();

  const RefPtr<WebGPUChild> mBridge;
  const RawId mId;

 public:
  explicit Device(Adapter* const aParent, RawId aId);
  already_AddRefed<Buffer> CreateBuffer(const dom::GPUBufferDescriptor& aDesc);
};
```

One of the trickiest part of this step is lifetime management. Our interfaces may hold JavaScript containers, which may in turn reference other interfaces. In order to cope with life times, Gecko has reference counting and cycle collection for WebIDL implementing classes. It means we are carefully describing what members we have and how cycle collector (CC for short) would need to traverse them and communicate with JavaScript garbage collector.

Getting this annotation right is a bit of an art form. We have a collection of useful but undocumented macros that cover the boilerplate, such as `NS_DEC*` ones in the code above. Yet, the situation could have been much better. Ideally, all I would need to do is:
```rust
#[derive(CycleCollection)]
struct MyImplementation { ... }
impl MyInterface for MyImplementation { ... }
```

## IPDL transfer

Gecko is a multi-process engine, which isolates GPU access to its own process. This means our actual interaction with OS and drivers happens on a different process from the content, where the WebIDL bindings are implemented. In order to communicate between processes, Gecko offers a system called "Inter-x-communication Protocol Definition Language", or IPDL for short. It allows sending messages back and forth, asynchronously, as well as sharing data between processes.

The trick is that IPDL protocols are defined in [separate files](https://searchfox.org/mozilla-central/rev/ac43d7e69a435ede789827f20ab309da6f04c09b/dom/webgpu/ipc/PWebGPU.ipdl) and glued with C++ using a lot of auto-generated code. The relevant messages for `GPUDevice` could look like this:

```cpp
async protocol PWebGPU
{
parent:
  async DeviceCreateBuffer(RawId selfId, GPUBufferDescriptor desc, RawId newId);
  async Shutdown();

child:
  async __delete__();
};
```

At build time, this definition spawns another piece of auto-generated boilerplate, split between the parent and child sides. The "PWebGPUParent.h/cpp" is fairly vebose and doesn't have anything interesting. The child portion of "PWebGPUChild.h/cpp" has our logic bits:
```cpp
class PWebGPUChild :
    public mozilla::ipc::IProtocol,
    public SupportsWeakPtr<PWebGPUChild>
{
public:
    MOZ_DECLARE_WEAKREFERENCE_TYPENAME(PWebGPUChild)
protected:
    typedef dom::GPUBufferDescriptor GPUBufferDescriptor;
    //...

public:
    bool SendDeviceCreateBuffer(const RawId& selfId, const GPUBufferDescriptor& desc, const RawId& newId);
    bool SendShutdown();
};
```

The IPDL generator knows about basic types, including some of the collections, but it doesn't know about our custom types. Thus, once again, we [use a few magic macros](https://searchfox.org/mozilla-central/rev/ac43d7e69a435ede789827f20ab309da6f04c09b/dom/webgpu/ipc/WebGPUSerialize.h#28) to reflect the structure of our types down to components that IPDL already knows:
```cpp
DEFINE_IPC_SERIALIZER_WITH_FIELDS(mozilla::dom::GPUExtensions,
                                  mAnisotropicFiltering);
DEFINE_IPC_SERIALIZER_WITH_FIELDS(mozilla::dom::GPULimits, mMaxBindGroups);
DEFINE_IPC_SERIALIZER_WITH_FIELDS(mozilla::dom::GPUDeviceDescriptor,
                                  mExtensions, mLimits);
```

## Object identities

The WebGPU interface methods for creating objects are not asynchronous: they are supposed to return a handle to the new object right away, even if the actual operation takes time. And there is definitely some time needed for the message to be generated and processed on the GPU process, nevermind the driver work.

In order to avoid blocking on that, we have the object "ID" management logic on the content side. The content can generate and remove IDs, while just notifying the server part (GPU process) about which IDs get used when creating new objects or using existing ones. This is what it looks like in [buffer creation](https://searchfox.org/mozilla-central/rev/ac43d7e69a435ede789827f20ab309da6f04c09b/dom/webgpu/ipc/WebGPUChild.cpp#83):
```cpp
RawId WebGPUChild::DeviceCreateBuffer(RawId aSelfId, const dom::GPUBufferDescriptor& aDesc) {
  RawId id = ffi::wgpu_client_make_device_id(mClient, aSelfId);
  if (!SendDeviceCreateBuffer(aSelfId, aDesc, id)) {
    MOZ_CRASH("IPC failure");
  }
  return id;
}
```

The `ffi::wgpu_client_*` methods are the exposed Rust interface for the WebGPU client, which is our content process.

## Glue

Having the IPDL defined, we are putting the actual message handling logic in the [methods of a class](https://searchfox.org/mozilla-central/rev/ac43d7e69a435ede789827f20ab309da6f04c09b/dom/webgpu/ipc/WebGPUParent.cpp#51) that we derive from `PWebGPUParent` (for messages that come from the content to GPU):
```cpp
class WebGPUParent final : public PWebGPUParent {
  NS_INLINE_DECL_THREADSAFE_REFCOUNTING(WebGPUParent)

 public:
  explicit WebGPUParent();
  ipc::IPCResult RecvDeviceCreateBuffer(RawId aSelfId, const dom::GPUBufferDescriptor& aDesc, RawId aNewId);
  ipc::IPCResult RecvShutdown();

 private:
  virtual ~WebGPUParent();
  const ffi::WGPUGlobal* const mContext;
};

ipc::IPCResult WebGPUParent::RecvDeviceCreateBuffer(RawId aSelfId, const dom::GPUBufferDescriptor& aDesc, RawId aNewId) {
  ffi::WGPUBufferDescriptor desc = {};
  desc.usage = aDesc.mUsage;
  desc.size = aDesc.mSize;
  ffi::wgpu_server_device_create_buffer(mContext, aSelfId, &desc, aNewId);
  return IPC_OK();
}
```

Notice the conversion from `dom::GPUBufferDescriptor` to `ffi::WGPUBufferDescriptor` here, which is then passed to Rust.

## Rust FFI

We are getting close to the core of the implementation, which originated in [Github](https://github.com/gfx-rs/wgpu) and is now mirrored on it. The project is split into 3 parts/crates:
  - "wgpu-core" has the Rust API and exposes the logic of implementing WebGPU.
  - "wgpu-native" has the C API to "wgpu-core" in native environment, which treats the global context as implicitly given. Applications (written in any language) can link to it in order to run on native WebGPU implementation. It's not used in Gecko.
  - "wgpu-remote" has the C API to "wgpu-core" in browser environment, which requires explicit management of contexts. It has separate entry points for the client and server parts.

The handling of [buffer creation](https://searchfox.org/mozilla-central/rev/ac43d7e69a435ede789827f20ab309da6f04c09b/gfx/wgpu/wgpu-remote/src/server.rs#61) is exposed on the server side as follows:
```rust
#[no_mangle]
pub extern "C" fn wgpu_server_device_create_buffer(
    global: &Global,
    self_id: id::DeviceId,
    desc: &core::resource::BufferDescriptor,
    new_id: id::BufferId,
) {
    gfx_select!(self_id => global.device_create_buffer(self_id, desc, new_id));
}
```

This function compiles to a standard "C" entry point, but in order to call it we still need a header definition. For this to be generated, we use the internally developed tool called [cbindgen](https://github.com/eqrion/cbindgen). "wgpu" has special [manifests](https://searchfox.org/mozilla-central/rev/ac43d7e69a435ede789827f20ab309da6f04c09b/gfx/wgpu/wgpu-remote/cbindgen.toml) for configuring the way C headers are generated. The way our function looks in "dom/webgpu/ffi/wgpu_ffi_generated.h" is:
```cpp
WGPU_INLINE
void wgpu_server_device_create_buffer(const WGPUGlobal *aGlobal,
                                      WGPUDeviceId aSelfId,
                                      const WGPUBufferDescriptor *aDesc,
                                      WGPUBufferId aNewId)
WGPU_FUNC;
```

Note: all the auto-generated headers are placed in the object folder (e.g. "obj-x86_64-pc-linux-gnu"), just like the non-generated ones. Except that the non-generated ones are already seen by your text editor, so it's typically configured to *avoid* the object folder. This makes it hard to look at the auto-generated headers when working on them. At least, [SearchFox](https://searchfox.org) indexes them, even though it doesn't tie them to specific revisions.

## Driver calls

Let's go back to `wgpu_server_device_create_buffer` implementation and see what it's doing. Since WebGPU may run on different native APIs, such as Vulkan, D3D12, and Metal, we don't know at compile time what logic needs to be followed for this operation. We encode the backend variant into the ID of the resource and provide a `gfx_select!` macro that dispatches to a proper generic "wgpu-native" variant based on that run-time information:
```rust
impl<F: IdentityFilter<BufferId>> Global<F> {
    pub fn device_create_buffer<B: GfxBackend>(
        &self,
        device_id: DeviceId,
        desc: &resource::BufferDescriptor,
        id_in: F::Input,
    ) -> BufferId {...}
}
```

This function ends up going through the following steps:
  1. Asking [rendy-memory](https://crates.io/crates/rendy-memory) for an appropriate chunk of memory.
  2. Asking [gfx-hal](https://crates.io/crates/gfx-hal) for a new buffer placed in this memory.
  3. Hooking up the new object to "wgpu-core"'s [resource tracker](https://searchfox.org/mozilla-central/rev/ac43d7e69a435ede789827f20ab309da6f04c09b/gfx/wgpu/wgpu-core/src/device.rs#822).

Rendy works on top of "gfx-hal", which very closely resembles Vulkan, with [a few differences](https://github.com/gfx-rs/gfx/wiki/Deviations-from-Vulkan).
So in the end we get "gfx-backend-xxx" translating the calls into the native API. The actual API is selected in WebGPU at the [adapter initialization](https://github.com/gpuweb/gpuweb/blob/402b69138fbedf4a3c9c85cd1bf7e1cc27c1b34e/spec/index.bs#L447) stage, which may consider adapters representing different native APIs, such as D3D12 and Vulkan on Windows.

All the objects come in and out of "wgpu-core" as indices, and they are resolved based on the `Global` context that is associated with the Gecko's GPU process endpoint, one per content process.

## Final words

Note: This walkthrough doesn't include any asynchronous bits of the API, such as requesting a buffer mapping, or handling errors in general. The focus of the post is not explaining how the API is difficult, but rather demonstrating the many circles of hell we go through in order to get it working.

Implementing WebGPU in Gecko involves parts written in 4 different languages, gluing them with various types of auto-generated code, and communicating across the process boundaries. It's complex and unlikely most efficient way to implement a Web API, comparing to Chromium's world where most of the pieces and plumbing are auto-generated from JSON declarations. Gecko's way has it's own benefits, if we can cope with complexity, such as having very specific behavior at any stage of the pipeline (as opposed to standard auto-generated code), and we'll do our best to take advantage of them.
