## Introduction

We're now able to pass arbitrary attributes to the vertex shader for each
vertex, but what about global variables? We're going to move on to 3D graphics
from this chapter on and that requires a model-view-projection matrix. We could
include it as vertex data, but that's a waste of memory and it would require us
to update the vertex buffer whenever the transformation changes. The
transformation could easily change every single frame.

The right way to tackle this in Vulkan is to use *resource descriptors*. A
descriptor is a way for shaders to freely access resources like buffers and
images. We're going to set up a buffer that contains the transformation matrices
and have the vertex shader access them through a descriptor. Usage of
descriptors consists of three parts:

* Specify a descriptor layout during pipeline creation
* Allocate a descriptor set from a descriptor pool
* Bind the descriptor set during rendering

The *descriptor layout* specifies the types of resources that are going to be
accessed by the pipeline, just like a render pass specifies the types of
attachments that will be accessed. A *descriptor set* specifies the actual
buffer or image resources that will be bound to the descriptors, just like a
framebuffer specifies the actual image views to bind to render pass attachments.
The descriptor set is then bound for the drawing commands just like the vertex
buffers and framebuffer.

There are many types of descriptors, but in this chapter we'll work with uniform
buffer objects (UBO). We'll look at other types of descriptors in future
chapters, but the basic process is the same. Let's say we have the data we want
the vertex shader to have in a C struct like this:

```c++
struct UniformBufferObject {
    glm::mat4 model;
    glm::mat4 view;
    glm::mat4 proj;
};
```

Then we can copy the data to a `VkBuffer` and access it through a uniform buffer
object descriptor from the vertex shader like this:

```glsl
layout(binding = 0) uniform UniformBufferObject {
    mat4 model;
    mat4 view;
    mat4 proj;
} ubo;

void main() {
    gl_Position = ubo.proj * ubo.view * ubo.model * vec4(inPosition, 0.0, 1.0);
    fragColor = inColor;
}
```

We're going to update the model, view and projection matrices every frame to
make the rectangle from the previous chapter spin around in 3D.

## Vertex shader

Modify the vertex shader to include the uniform buffer object like it was
specified above. I will assume that you are familiar with MVP transformations.
If you're not, see [the resource](http://opengl.datenwolf.net/gltut/html/index.html)
mentioned in the first chapter.

```glsl
#version 450
#extension GL_ARB_separate_shader_objects : enable

layout(binding = 0) uniform UniformBufferObject {
    mat4 model;
    mat4 view;
    mat4 proj;
} ubo;

layout(location = 0) in vec2 inPosition;
layout(location = 1) in vec3 inColor;

layout(location = 0) out vec3 fragColor;

out gl_PerVertex {
    vec4 gl_Position;
};

void main() {
    gl_Position = ubo.proj * ubo.view * ubo.model * vec4(inPosition, 0.0, 1.0);
    fragColor = inColor;
}
```

Note that the order of the `uniform`, `in` and `out` declarations doesn't
matter. The `binding` directive is similar to the `layout` directive for
attributes. We're going to reference this binding in the descriptor layout. The
line with `gl_Position` is changed to use the transformations to compute the
final position in clip coordinates.

## Descriptor set layout

The next step is to define the UBO on the C++ side and to tell Vulkan about this
descriptor in the vertex shader.

```c++
struct UniformBufferObject {
    glm::mat4 model;
    glm::mat4 view;
    glm::mat4 proj;
};
```

We can exactly match the definition in the shader using data types in GLM. The
data in the matrices is binary compatible with the way the shader expects it, so
we can later just `memcpy` a `UniformBufferObject` to a `VkBuffer`.

We need to provide details about every descriptor binding used in the shaders
for pipeline creation, just like we had to do for every vertex attribute and its
`location` index. We'll set up a new function to define all of this information
called `createDescriptorSetLayout`. It should be called right before pipeline
creation, because we're going to need it there.

```c++
void initVulkan() {
    ...
    createDescriptorSetLayout();
    createGraphicsPipeline();
    ...
}

...

void createDescriptorSetLayout() {

}
```

Every binding needs to be described through a `VkDescriptorSetLayoutBinding`
struct.

```c++
void createDescriptorSetLayout() {
    VkDescriptorSetLayoutBinding uboLayoutBinding = {};
    uboLayoutBinding.binding = 0;
    uboLayoutBinding.descriptorType = VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER;
    uboLayoutBinding.descriptorCount = 1;
}
```

The first two fields specify the `binding` used in the shader and the type of
descriptor, which is a uniform buffer. It is possible for a uniform buffer
descriptor to be an array of data, and `descriptorCount` specifies the number of
values in the array. This could be used to specify a transformation for each of
the bones in a skeleton for skeletal animation, for example. Our MVP
transformation is a single object, so we're using a `descriptorCount` of `1`.

```c++
uboLayoutBinding.stageFlags = VK_SHADER_STAGE_VERTEX_BIT;
```

We also need to specify in which shader stages the descriptor is going to be
referenced. The `stageFlags` field can be a combination of `VkShaderStage` flags
or the value `VK_SHADER_STAGE_ALL_GRAPHICS`. In our case, we're only referencing
the descriptor from the vertex shader.

```c++
uboLayoutBinding.pImmutableSamplers = nullptr; // Optional
```

The `pImmutableSamplers` field is only relevant for image sampling related
descriptors, which we'll look at later. You can leave this to its default value.

All of the descriptor bindings are combined into a single
`VkDescriptorSetLayout` object. Define a new class member above
`pipelineLayout`:

```c++
VDeleter<VkDescriptorSetLayout> descriptorSetLayout{device, vkDestroyDescriptorSetLayout};
VDeleter<VkPipelineLayout> pipelineLayout{device, vkDestroyPipelineLayout};
```

We can then create it using `vkCreateDescriptorSetLayout`. This function accepts
a simple `VkDescriptorSetLayoutCreateInfo` with the array of bindings:

```c++
VkDescriptorSetLayoutCreateInfo layoutInfo = {};
layoutInfo.sType = VK_STRUCTURE_TYPE_DESCRIPTOR_SET_LAYOUT_CREATE_INFO;
layoutInfo.bindingCount = 1;
layoutInfo.pBindings = &uboLayoutBinding;

if (vkCreateDescriptorSetLayout(device, &layoutInfo, nullptr, &descriptorSetLayout) != VK_SUCCESS) {
    throw std::runtime_error("failed to create descriptor set layout!");
}
```

We need to specify the descriptor set layout during pipeline creation to tell
Vulkan which descriptors the shaders will be using. Descriptor set layouts are
specified in the pipeline layout object. Modify the `VkPipelineLayoutCreateInfo`
to reference the layout object:

```c++
VkDescriptorSetLayout setLayouts[] = {descriptorSetLayout};
VkPipelineLayoutCreateInfo pipelineLayoutInfo = {};
pipelineLayoutInfo.sType = VK_STRUCTURE_TYPE_PIPELINE_LAYOUT_CREATE_INFO;
pipelineLayoutInfo.setLayoutCount = 1;
pipelineLayoutInfo.pSetLayouts = setLayouts;
```

You may be wondering why it's possible to specify multiple descriptor set
layouts here, because a single one already includes all of the bindings. We'll
get back to that in the next chapter, where we'll look into descriptor pools and
descriptor sets.

## Uniform buffer

In the next chapter we'll specify the buffer that contains the UBO data for the
shader, but we need to create this buffer first. We're going to copy new data to
the uniform buffer every frame, so this time the staging buffer actually needs
to stick around.

Add new class members for `uniformStagingBuffer`, `uniformStagingBufferMemory`,
`uniformBuffer`, and `uniformBufferMemory`:

```c++
VDeleter<VkBuffer> indexBuffer{device, vkDestroyBuffer};
VDeleter<VkDeviceMemory> indexBufferMemory{device, vkFreeMemory};

VDeleter<VkBuffer> uniformStagingBuffer{device, vkDestroyBuffer};
VDeleter<VkDeviceMemory> uniformStagingBufferMemory{device, vkFreeMemory};
VDeleter<VkBuffer> uniformBuffer{device, vkDestroyBuffer};
VDeleter<VkDeviceMemory> uniformBufferMemory{device, vkFreeMemory};
```

Similarly, create a new function `createUniformBuffer` that is called after
`createIndexBuffer` and allocates the buffers:

```c++
void initVulkan() {
    ...
    createVertexBuffer();
    createIndexBuffer();
    createUniformBuffer();
    ...
}

...

void createUniformBuffer() {
    VkDeviceSize bufferSize = sizeof(UniformBufferObject);

    createBuffer(bufferSize, VK_BUFFER_USAGE_TRANSFER_SRC_BIT, VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT | VK_MEMORY_PROPERTY_HOST_COHERENT_BIT, uniformStagingBuffer, uniformStagingBufferMemory);
    createBuffer(bufferSize, VK_BUFFER_USAGE_TRANSFER_DST_BIT | VK_BUFFER_USAGE_UNIFORM_BUFFER_BIT, VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT, uniformBuffer, uniformBufferMemory);
}
```

We're going to write a separate function that updates the uniform buffer with a
new transformation every frame, so there will be no `vkMapMemory` and
`copyBuffer` operations here.

```c++
void mainLoop() {
    while (!glfwWindowShouldClose(window)) {
        glfwPollEvents();

        updateUniformBuffer();
        drawFrame();
    }

    vkDeviceWaitIdle(device);
}

...

void updateUniformBuffer() {

}
```

## Updating uniform data

Create a new function `updateUniformBuffer` and add a call to it from the main
loop. This function will generate a new transformation every frame to make the
geometry spin around. We need to include two new headers to implement this
functionality:

```c++
#define GLM_FORCE_RADIANS
#include <glm/glm.hpp>
#include <glm/gtc/matrix_transform.hpp>

#include <chrono>
```

The `glm/gtc/matrix_transform.hpp` header exposes functions that can be used to
generate model transformations like `glm::rotate`, view transformations like
`glm::lookAt` and projection transformations like `glm::perspective`. The
`GLM_FORCE_RADIANS` definition is necessary to make sure that functions like
`glm::rotate` use radians as arguments, to avoid any possible confusion.

The `chrono` standard library header exposes functions to do precise
timekeeping. We'll use this to make sure that the geometry rotates 90 degrees
per second regardless of frame rate.

```c++
void updateUniformBuffer() {
    static auto startTime = std::chrono::high_resolution_clock::now();

    auto currentTime = std::chrono::high_resolution_clock::now();
    float time = std::chrono::duration_cast<std::chrono::milliseconds>(currentTime - startTime).count() / 1000.0f;
}
```

The `updateUniformBuffer` function will start out with some logic to calculate
the time in seconds since rendering has started with millisecond accuracy. If
you need timing to be more precise, then you can use `std::chrono::microseconds`
and divide by `1e6f`, which is short for `1000000.0f`.

We will now define the model, view and projection transformations in the
uniform buffer object. The model rotation will be a simple rotation around the
Z-axis using the `time` variable:

```c++
UniformBufferObject ubo = {};
ubo.model = glm::rotate(glm::mat4(), time * glm::radians(90.0f), glm::vec3(0.0f, 0.0f, 1.0f));
```

The `glm::rotate` function takes an existing transformation, rotation angle and
rotation axis as parameters. The `glm::mat4()` default constructor returns an
identity matrix. Using a rotation angle of `time * glm::radians(90.0f)`
accomplishes the purpose of rotation 90 degrees per second.

```c++
ubo.view = glm::lookAt(glm::vec3(2.0f, 2.0f, 2.0f), glm::vec3(0.0f, 0.0f, 0.0f), glm::vec3(0.0f, 0.0f, 1.0f));
```

For the view transformation I've decided to look at the geometry from above at a
45 degree angle. The `glm::lookAt` function takes the eye position, center
position and up axis as parameters.

```c++
ubo.proj = glm::perspective(glm::radians(45.0f), swapChainExtent.width / (float) swapChainExtent.height, 0.1f, 10.0f);
```

I've chosen to use a perspective projection with a 45 degree vertical
field-of-view. The other parameters are the aspect ratio, near and far
view planes. It is important to use the current swap chain extent to calculate
the aspect ratio to take into account the new width and height of the window
after a resize.

```c++
ubo.proj[1][1] *= -1;
```

GLM was originally designed for OpenGL, where the Y coordinate of the clip
coordinates is inverted. The easiest way to compensate for that is to flip the
sign on the scaling factor of the Y axis in the projection matrix. If you don't
do this, then the image will be rendered upside down.

All of the transformations are defined now, so we can copy the data in the
uniform buffer object to the uniform buffer. This happens in exactly the same
way as we did for vertex buffers with a staging buffer:

```c++
void* data;
vkMapMemory(device, uniformStagingBufferMemory, 0, sizeof(ubo), 0, &data);
    memcpy(data, &ubo, sizeof(ubo));
vkUnmapMemory(device, uniformStagingBufferMemory);

copyBuffer(uniformStagingBuffer, uniformBuffer, sizeof(ubo));
```

In the next chapter we'll look at descriptor sets, which will actually bind the
`VkBuffer` to the uniform buffer descriptor so that the shader can access this
transformation data.

[Full code listing](/code/descriptor_layout.cpp)