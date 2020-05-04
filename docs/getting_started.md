# Getting Started

`vk-bootstrap` reduces the complexity of setting up a vulkan application into 3 main steps; instance creation, Physical device selection, and device creation. 

## Instance Creation

Creating an instance with `vk-bootstrap` uses the `vkb::InstanceBuilder` class.

Simply create a builder variable and call the `build()` member function.
```cpp
vkb::InstanceBuilder instance_builder;
auto instance_builder_return = instance_builder.build();
```
Because creating an instance may fail, the builder returns an 'Expected' type. This contains either a valid `vkb::Instance` struct, which includes a `VkInstance` handle, or contains an `vkb::InstanceError`.
```cpp
if (!instance_builder_return) {
    printf("Failed to create Vulkan instance. Cause %s\n", 
        vkb::to_string(instance_builder_return.error().type));
    return -1;
} 
```
Once any possible errors have been dealt with, we can pull the `vkb::Instance` struct out of the `Expected`. 
```cpp
vkb::Instance vkb_instance = instance_builder_return.value();
```
This is enough to create a usable `VkInstance` handle but many will want to customize it a bit. To configure instance creation, simply call the member functions on the `vkb::InstanceBuilder` object before `build()` is called.

The most common customization to instance creation is enabling the "Validation Layers", an invaluable tool for any vulkan application developer. 
```cpp
instance_builder.request_validation_layers ();
```
The other common customization point is setting up the `Debug Messenger Callback`, the mechanism in which an application can control what and where the "Validation Layers" log its output.
```cpp
instance_builder.set_debug_callback (
    [] (VkDebugUtilsMessageSeverityFlagBitsEXT messageSeverity,
	    VkDebugUtilsMessageTypeFlagsEXT messageType,
	    const VkDebugUtilsMessengerCallbackDataEXT* pCallbackData,
	    void *pUserData) 
        -> VkBool32 {
			auto severity = vkb::to_string_message_severity (messageSeverity);
			auto type = vkb::to_string_message_type (messageType);
			printf ("[%s: %s] %s\n", severity, type, pCallbackData->pMessage);
			return VK_FALSE;
		}
    );
```
Alternatively, `vk-bootstrap` provides a 'default debug messenger' that prints to standard output.
```cpp
instance_builder.use_default_debug_messenger();
```
Configuration can be chained together and done inline with building, like so.
```cpp
auto inst_builder_ret = instance_builder 
        .set_app_name ("Awesome Vulkan Application")
        .set_engine_name("Excellent Game Engine")
        .require_api_version(1,0,0)
        .build();
```

The `vkb::Instance` struct is meant to hold all the necessary instance level data to enable proper Physical Device selection. It also is meant for easy destructuring into custom classes if so desired.
```cpp
struct CustomVulkanWrapper {
    VkInstance instance;
    //...
};
CustomVulkanWrapper custom_vk_class;
custom_vk_class.instance = vkb_instance.instance;
```

When the application is finished with the vulkan, call `vkb::destroy_instance()` to dispose of the instance and associated data. 
```cpp
// cleanup 
vkb::destroy_instance(vkb_instance);
```
### Instance Creation Summary
```cpp
vkb::InstanceBuilder instance_builder;
auto instance_builder_return = instance_builder
        // Instance creation configuration
        .request_validation_layers()
        .use_default_debug_messenger()
        .build ();
if (!instance_builder_return) {
    // Handle error
} 
vkb::Instance vkb_instance = instance_builder_return.value ();

// at program end
vkb::destroy_instance(vkb_instance);
```
## Surface Creation

Presenting images to the screen Vulkan requires creating a surface, encapsulated in a `VkSurfaceKHR` handle. Creating a surface is the responsibility of the windowing system, thus is out of scope for `vk-bootstrap`. However, `vk-bootstrap` does try to make the process as painless as possible by automatically enabling the correct windowing extensions in `VkInstance` creation. 

Windowing libraries which support Vulkan usually provide a way of getting the `VkSurfaceKHR` handle for the window. These methods require a valid Vulkan instance, thus must be done after instance creation.

Examples for GLFW and SDL2 are listed below. 
```cpp
vkb::Instance vkb_instance; //valid vkb::Instance
VkSurfaceKHR surface = nullptr;
// window is a valid library specific Window handle

// GLFW
VkResult err = glfwCreateWindowSurface (vkb_instance.instance, window, NULL, &surface);
if (err != VK_SUCCESS) { /* handle error */ }

// SDL2
SDL_bool err = SDL_Vulkan_CreateSurface(window, vkb_instance.instance, &surface);
if (!err){ /* handle error */ }

```

## Physical Device Selection

Once a Vulkan instance has been created, the next step is to find a suitable GPU for the application to use. `vk-bootstrap` provide the `vkb::PhysicalDeviceSelector` class to streamline this process.

Creating a `vkb::PhysicalDeviceSelector` requires a valid `vkb::Instance` to construct.

It follows the same pattern laid out by `vkb::InstanceBuilder`.
```cpp
vkb::PhysicalDeviceSelector phys_device_selector (vkb_instance); 
auto physical_device_selector_return = phys_device_selector
        .set_surface(surface_handle)
        .select ();
if (!physical_device_selector_return) {
    // Handle error
}
auto phys_device = phys_device_ret.value ();
```

To select a physical device, call `select()` on the `vkb::PhysicalDeviceSelector` object.
By default, this will prefer a discrete GPU.

No cleanup is required for `vkb::PhysicalDevice`.

The `vkb::PhysicalDeviceSelector` will look for the first device in the list that satisfied all the specified criteria, and if none is found, will return the first device that partially satisfies the criteria. 

The various "require" and "desire" pairs of functions indicate to `vk-bootstrap` what features and capabilities are necessary for an application and what are simply preferred. A "require" function will fail any `VkPhysicalDevice` that doesn't satisfy the constraint, while any criteria that doesn't satisfy the "desire" functions will make the `VkPhysicalDevice` only 'partially satisfy'. 

```c
// Application cannot function without this extension
phys_device_selector.add_required_extension("VK_KHR_timeline_semaphore");

// Application can deal with the lack of this extension
phys_device_selector.add_desired_extension("VK_KHR_imageless_framebuffer");
```

Note: 

Because `vk-bootstrap` does not manage creating a `VkSurfaceKHR` handle, it is explicitly passed into the `vkb::PhysicalDeviceSelector` for proper querying of surface support details. Unless the `vkb::InstanceBuilder::set_headless()` function was called, the physical device selector will emit `no_surface_provided` error. If an application does intend to present but cannot create a `VkSurfaceKHR` handle before physical device selection, use `defer_surface_initialization()` to disable the `no_surface_provided` error. 

## Device Creation

Once a `VkPhysicalDevice` has been selected, a `VkDevice` can be created. Facilitating that is the `vkb::DeviceBuilder`. Creation and usage follows the forms laid out by `vkb::InstanceBuilder`.

```cpp
vkb::DeviceBuilder device_builder{ phys_device};
auto dev_ret = device_builder.build ();
if (!dev_ret) {
    // error
}
 ```

The features and extensions used as selection criteria in `vkb::PhysicalDeviceSelector` automatically propagate into `vkb::DeviceBuilder`. Because of this, there is no way to enable features or extensions that were not specified during `vkb::PhysicalDeviceSelector`. This is by design as any feature or extension enabled *must* have support from the `VkPhysicalDevice`.

The common method to extend Vulkan functionality in existing API calls is to use the pNext chain. This is accounted for `VkDevice` creation with the `add_pNext` member function of `vkb::DeviceBuilder`. Note: Any structures added to the pNext chain must remain valid until `build()` is called.

```cpp
VkPhysicalDeviceDescriptorIndexingFeatures descriptor_indexing_features{};

auto dev_ret = device_builder.add_pNext(&descriptor_indexing_features)
                             .build ();
```

### Queue Selection and Querying

By default, `vkb::DeviceBuilder` will enable one queue from each queue family available from the `VkPhysicalDevice` unless otherwise specified. This is due to the variety of possible combinations of queue families on different hardware. 

## Swapchain
// TODO

