---
layout: post
title:  "Abstracting a graphics API"
date:   2025-02-02 16:00:00 +0100
categories: rendering engine
hidden: true
---


### The wonderful world of graphics APIs
As a toy project, i'm working on a game engine (which will hopefully be used to produce some actual games, sooner or later), and until a month ago i was using Vulkan directly: 
my engine's subsystems were directly dealing with creating and destroying resources, managing command buffers, and in general the ownership of all Vulkan related things wasn't clear. Due to all these issues, i decided to rewrite all the device-related code.

Usually, when you're working on some kind of 3D application, the first step is understanding __what__ you're going to render: let's say that our application's job is to render a 3D **scene** onto a **viewport**: the scene is composed of surfaces that are represented by one or more 3D **meshes**, defining the surface's geometry as a list of vertices, with each mesh associated to a **material instance**, created from a **master material**: the master defines the shading technique, while the instance defines the parameters of the technique.

In 3D, meshes are composed by one or more **buffers**, which contain the mesh's actual geometry: the materials are defined by one or more **shader** programs and a set of resources, some **Textures**, some **Buffers** and the pipeline's state (for example, the *Cull mode*, the geometry's *Winding order* etc...), and this is where i decided to place my abstraction layer, so that:

* The higher-level rendering code (creating Meshes, creating Materials etc...) would be written only once;
* All low-level resources (buffers, textures, shader modules etc...) would be managed by a single entity;
* The underlying graphics API would be hidden from the user (i plan on supporting DX12, sooner or later);
* The user could place barriers explicitly;
* There would still be some amount of validation performed, to ensure correct API usage.

In this post, i'm going to describe how i designed my engine's graphics API abstraction.

### An higher level overview
My rendering API exposes three main objects:
* The `Device` is where all begins: it is created explicitly through a construction function, and it's tasked with creating all resources. All other objects are created by a `Device` instance;
* The `Swapchain` is tasked with presenting and acquiring the window's surface textures;
* The `CommandList`s is where command recording happens;

Additionally, the `Device` can create the following resources:
* `Buffer`s are linear memory region that can be used both as vertex attribute buffers and shader resources, depending on their `BufferUsageFlags`;
* `Texture`s are rectangular memory region that can be used both as vertex attribute buffers and shader resources, depending on their `BufferUsageFlags`; at the moment, they consist of a `VkImage` + `VkImageView` pair (the view is a view on all the mips/layers of the image), in the future i might implement a `TextureSlice` to act like a `VkImageView`;
* `Sampler`s are used to sample a `Texture` in shaders;
* `ShaderModule`s are compiled shaders (for now i only deal with SPIR-V);
* `BindingSetLayout`s define what resource types a shader uses: they are used both to create `BindingSet`s (defining the actual resouces) and `*Pipeline` objects;
* `GraphicsPipeline` and `ComputePipeline` contain all the pipeline state for executing draw calls/dispatch calls;

Additionally, my renderer heavily uses bindless textures, so i decided to introduce the following other objects:
* `TextureTable`s define groups of `Texture`s that can be bound to `BindingSet`s: textures can be added and removed at any moment, and the existing `BindingSet`s using the table will be updated accordingly;
* `SamplerTable`s behave like `TextureTable`s, but for `Sampler`s.

>> ❗`SamplerTable` mainly exists because i used to use Vulkan Combined Image Samplers, which aren't supported by DX12: i then took the radical decision to change my rendering code to use separate samplers and images

As an example, this is how i create a device, a texture, create a texture table, bind the texture to the table and then create a binding set that binds the table:
```cpp
// create_device() returns a Result<gpu::Device, gpu::Device::Error> type
// expect() unwraps the Result panicking if it's an error.
// Additionally, (almost) all ram memory allocations happen through an Allocator object.
gpu::Device device = gpu::Device::create_device(global_heap_allocator().get(), gpu::DeviceCreateInfo {
                                                 .options = gpu::DeviceCreateOptions {
                                                     .enable_debug_features = true,
                                                 }
                                             }).expect("Failed to create gpu device"); 
gpu::Texture texture = device.create_texture(gpu::TextureInfo{
    .debug_name = "Texture",
    .memory_location = MemoryLocation::Device,
    .usage_flags = gpu::TextureUsageFlags {
        .sampled = true,
    },
    .texture_type = gpu::TextureType::Texture2D,
    .texture_format = gpu::TextureFormat::RGBA8,
    .extents = gpu::Extents3D {.width = 128, .height = 128, .depth = 1},
    .num_mips = 1,
    .num_layers = 1,
});

// Write the texture data somehow

gpu::TextureTable texture_table = device.create_texture_table(gpu::TextureTableInfo{});
gpu::BindingSetLayout bsl = device.create_binding_set_layout(gpu::BindingSetLayoutInfo {
    .bindings = std::array {
        gpu::BindingSlotLayoutInfo {
            gpu::BindingSlotType::TextureTable,
        }
    },
    .shader_Stage_flags = gpu::ALL_SHADER_STAGE_FLAGS,
});

gpu::BindingSet bs = device.create_binding_set(gpu::BindingSetInfo {
    .layout = bsl,
    .bindings = std::array {
        BindingSlot::texture_table(texture_table),
    }
});

device.bind_texture_in_table(texture_table, texture, 0);
// The texture will be bound in the table at slot 0:
// all BindingSets using the table will be automatically
// updated to reflect this change

```

### Under the hood: `DeviceImpl`
`gpu::Device` doesn't actually create/destroy the resources directly, rather it passes these events to a class deriving from `DeviceImpl`: the `DeviceImpl` is meant to do very minimal validity checks, and returns, for each resource, a type deriving from `*Impl` (e.g `Buffer` has a `BufferImpl`), which is then wrapped into a `GpuResourceHandle<T>` by the `Device`: the `GpuResourceHandle<T>` is the type actually used and managed by the user:
```cpp
    struct BufferImpl {
        ResourceLabel label;
        u64 buffer_size{};
        BufferUsageFlags flags;
    };
```


Let's examine how `create_buffer()` is implemented (expanding all the macros and condensing the code for readability):
```cpp
void zgear::gpu::Device::destroy_buffer(const Buffer &buffer) { 
        check(impl_);
        impl_->destroy_buffer(buffer.handle);
        if (resource_table_) { std::erase(resource_table_->buffers, buffer); };
}

zgear::gpu::Buffer zgear::gpu::Device::create_buffer(const BufferInfo &buffer_info) {
    validate_buffer_info(buffer_info);
    check(impl_);
    auto res = impl_->create_buffer(buffer_info);
    check(res);
    if (resource_table_) { resource_table_->buffers.push_back({res}); };
    return {res};

}
```

When you create a buffer:
1. The buffer creation parameters (`buffer_info`) are validated (e.g to check that you're not creating a zero-sized buffer);
2. The buffer is created by the `DeviceImpl`, which returns a type deriving from `BufferImpl`;
3. The resulting handle is stored in a list of `Buffer`s managed by this device (`ResourceTable::buffer_info`), to ensure that upon destruction all resources have been destroyed;
    * The `ResourceTable` is only created when device debugging is enabled
4. The resulting handle is wrapped in a `GpuResourceHandle<T>` and returned.

Likewise, `destroy_buffer()` does something similiar:
```cpp
void zgear::gpu::Device::destroy_buffer(const Buffer &buffer) { 
        if (resource_table_) { std::erase(resource_table_->buffers, buffer); };
        check(impl_);
        impl_->destroy_buffer(buffer.handle);
}

```

1. If the `Device` was created with debugging enabled, we check that the `Buffer` was actually created by this device, then it is removed from the `ResourceTable`;
2. The `DeviceImpl` object then destroys the buffer.

### Under the hood: `DeviceImplVulkan`
Let's examine how a Buffer is created by `DeviceImplVulkan`:
```cpp
VmaAllocationCreateInfo get_allocation_info(MemoryLocation memory_location) {
    VmaAllocationCreateInfo allocation_info{};
    allocation_info.usage = VMA_MEMORY_USAGE_AUTO;
    if (memory_location == MemoryLocation::Host) {
        allocation_info.flags = VMA_ALLOCATION_CREATE_MAPPED_BIT |
                                VMA_ALLOCATION_CREATE_HOST_ACCESS_SEQUENTIAL_WRITE_BIT |
                                VMA_ALLOCATION_CREATE_HOST_ACCESS_ALLOW_TRANSFER_INSTEAD_BIT;
        allocation_info.priority = 1.0f;
    } else {
        allocation_info.flags = VMA_ALLOCATION_CREATE_DEDICATED_MEMORY_BIT;
        allocation_info.priority = 1.0f;
    }
    return allocation_info;
}

struct VulkanBuffer : BufferImpl {
    VkBuffer buffer{};
    VmaAllocation allocation{};
    VmaAllocationInfo allocation_info{};
    VkDeviceAddress buffer_address{};
};

BufferImpl *DeviceImplVulkan::create_buffer(const BufferInfo &buffer_info) {
    // 1.
    auto buffer = allocator_->create<VulkanBuffer>();

    // 2.
    std::array<u32, max_queues> queue_family_indices{};
    u32 queue_count = 1;
    queue_family_indices[0] = logical_device_info.graphics_queue.qfi;
    VkSharingMode sharing_mode = VK_SHARING_MODE_EXCLUSIVE;
    if (logical_device_info.compute_queue.handle != logical_device_info.graphics_queue.handle) {
        sharing_mode = VK_SHARING_MODE_CONCURRENT;
        queue_family_indices[1] = logical_device_info.compute_queue.qfi;
        queue_count += 1;
    }
    VkBufferUsageFlags usage_flags = get_buffer_usage_flags(buffer_info);
    VkBufferCreateInfo buffer_create_info = {
        .sType = VK_STRUCTURE_TYPE_BUFFER_CREATE_INFO,
        .pNext = nullptr,
        .flags = {},
        .size = buffer_info.buffer_size,
        .usage = usage_flags,
        .sharingMode = sharing_mode,
        .queueFamilyIndexCount = queue_count,
        .pQueueFamilyIndices = queue_family_indices.data()
    };
    VmaAllocationCreateInfo alloc_info = get_allocation_info(buffer_info.memory_location);
    vk_check(
        vmaCreateBuffer(vma_allocator, &buffer_create_info, &alloc_info, &buffer->buffer, &buffer->allocation,
                        &buffer->allocation_info),
        "Failed to allocate buffer");
    VkBufferDeviceAddressInfo buffer_device_address_info{
        .sType = VK_STRUCTURE_TYPE_BUFFER_DEVICE_ADDRESS_INFO,
        .pNext = nullptr,
        .buffer = buffer->buffer
    };
    buffer->buffer_address = vkGetBufferDeviceAddress(logical_device_info.handle, &buffer_device_address_info);

    // 3.
    set_object_name(logical_device_info.handle, buffer->buffer, VkObjectType::VK_OBJECT_TYPE_BUFFER,
                    buffer_info.debug_name);
    
    // 4.
    buffer->buffer_size = buffer_info.buffer_size;
    buffer->flags = buffer_info.usage_flags;
    buffer->label = ResourceLabel::create(buffer_info.debug_name, allocator_.get());

    return buffer;
}
```

As you can see, this is mostly standard Vulkan code:
1. I allocate an object of type `VulkanBuffer` deriving from `BufferImpl`, and containing the actual Vulkan objects;
2. The actual `VkBuffer` object is created through `Vma`
    * Based on the `Buffer`'s `MemoryLocation`, different Vma flags are used.
3. The object's debug label is set through `vkSetDebugUtilsObjectName` (used by e.g RenderDoc);
4. The buffer's creation parametere are stored, and the `BufferImpl` object is returned.