---
title: An Engine Programmer's Guide to Android External Textures
date: 2026-05-30 00:53:35 +0800
categories: [Graphics]
tags: [Android, Graphics]
---

If you're an engine progrmmar (or you're working with a game/rendering engine) and the engine supports Android platform, you have likely heard of "External Textures." But what exactly are they, and how do they work under the hood? In this article, I will walk you through the core mechanics of external textures on Android.

By the end of this guide, you will understand the requirements for integrating live camera and video streams into an Android-supported engine, and how to properly interface with Android's native graphics components.

## What are External Textures?

### Problem Statement

Before defining external textures, let's clarify the problem they solve.

A camera or video decoder frame is not usually born as a regular GPU `Texture2D`. A normal `Texture2D` usually means that the application owns a GPU texture object, knows its format, and can sample it with a regular `sampler2D`.

Camera and video frames are different. They are produced by Android system components such as the camera HAL or hardware video decoder, and are usually written into hardware-backed buffers allocated by Android's graphics allocator. These buffers may use YUV formats, multi-plane layouts, vendor-private memory layouts, or tiling/compression schemes that the application should not interpret directly.

When you want to ingest frames from a camera or media decoder, apply custom processing, and render them to the screen, the naive approach is to turn each frame into a regular engine texture. In many engines, that means converting the producer-owned hardware buffer into something that looks like a normal `Texture2D`.

If we forced every frame into a regular `Texture2D`, the pipeline would look like this:

1. Copy the massive pixel data from hardware memory into CPU-visible memory.
2. Convert the data from YUV into a GPU-friendly RGB format.
3. Upload to GPU texture as a Texture2D

Executing this on the CPU every frame destroys rendering performance, drains the battery, and quickly overheats the device. This is the bottleneck External Textures were introduced to solve.

### External Texture

An external texture is a way for rendering code to sample an image buffer that was produced outside the rendering API's normal texture allocation path. Instead of creating a regular texture, filling it from CPU memory, and then sampling it, the app asks Android to connect a producer such as `Camera` or `MediaCodec` to a consumer that the GPU can sample from. 

By sampling the hardware-backed buffer through the graphics stack, we avoid the expensive app-level CPU copy, CPU-side YUV-to-RGB conversion, and texture upload loop. In the common camera/video rendering path, this gives us the zero-copy behavior we want from the engine's point of view: the frame stays in graphics-friendly memory and the shader samples it through a backend-specific external image path.

For formats such as camera YUV buffers, the graphics stack can also perform the required format conversion during sampling, so the shader can work with RGB-like color values instead of manually decoding YUV planes.

### The Producer-Consumer architecture in Android

Let's first look at how the Producer-Consumer architecture works in Android.

Take the Camera API as an example: the camera acts as the "Producer," providing a stream of captured frame data. To receive this data, we provide a "Consumer", which is a surface which consume the stream:

```kotlin
// 1. We have a Surface, but the Camera doesn't know (or care) what is behind it.
val myConsumerSurface: Surface = getSurfaceFromSomewhere() 

// 2. We tell the Camera to send its output to this Surface
val captureRequestBuilder = cameraDevice.createCaptureRequest(CameraDevice.TEMPLATE_PREVIEW)
captureRequestBuilder.addTarget(myConsumerSurface)

// 3. We create the session, officially connecting the Producer (Camera) to the Consumer (Surface)
cameraDevice.createCaptureSession(
    listOf(myConsumerSurface),
    object : CameraCaptureSession.StateCallback() {
        override fun onConfigured(session: CameraCaptureSession) {
            // Start pumping frames into the Surface
            session.setRepeatingRequest(captureRequestBuilder.build(), null, null)
        }
        
        override fun onConfigureFailed(session: CameraCaptureSession) {
            // Handle failure
        }
    },
    null
)
```

Note: You can also do entirely in C++, but for this article, I'm going to assume we have a Java/Kotlin app with a native C++ rendering engine library under the hood.


Notice what is happening in the code above: the Camera API only asks for a `Surface`. A `Surface` in Android is essentially a handle to a buffer queue. The Camera (the Producer) does not know, nor does it care, where those pixels are actually going. It just pushes frames into the Surface.

The flow is roughly like this:

```
Camera / MediaCodec
        |
        v
      Surface
        |
        v
   BufferQueue
        |
        +--> SurfaceTexture -> GL_TEXTURE_EXTERNAL_OES -> OpenGL ES
        |
        +--> ImageReader    -> HardwareBuffer          -> Vulkan/OpenGLES import
```

Let's talk about the most common Consumer first: `SurfaceTexture`.

## SurfaceTexture

`SurfaceTexture` is the usual OpenGL ES path for Android external textures. It provides a `Surface` that producers can write into, while exposing the latest queued buffer as a `GL_TEXTURE_EXTERNAL_OES` texture to the rendering thread. Basically, you generate an OpenGL texture ID and bind it to `GL_TEXTURE_EXTERNAL_OES`. You then pass that texture ID into a new `SurfaceTexture`, which you wrap in an Android `Surface`.

```kotlin
// 1. Generate a standard OpenGL texture ID
val textures = IntArray(1)
GLES20.glGenTextures(1, textures, 0)
val textureId = textures[0]

// 2. Bind it to the EXTERNAL_OES target (Crucial: DO NOT use GL_TEXTURE_2D)
GLES20.glBindTexture(GLES11Ext.GL_TEXTURE_EXTERNAL_OES, textureId)
GLES20.glTexParameterf(GLES11Ext.GL_TEXTURE_EXTERNAL_OES, GLES20.GL_TEXTURE_MIN_FILTER, GLES20.GL_LINEAR.toFloat())
GLES20.glTexParameterf(GLES11Ext.GL_TEXTURE_EXTERNAL_OES, GLES20.GL_TEXTURE_MAG_FILTER, GLES20.GL_LINEAR.toFloat())
GLES20.glTexParameteri(GLES11Ext.GL_TEXTURE_EXTERNAL_OES, GLES20.GL_TEXTURE_WRAP_S, GLES20.GL_CLAMP_TO_EDGE)
GLES20.glTexParameteri(GLES11Ext.GL_TEXTURE_EXTERNAL_OES, GLES20.GL_TEXTURE_WRAP_T, GLES20.GL_CLAMP_TO_EDGE)

// 3. Wrap it in a SurfaceTexture, and then a Surface
val surfaceTexture = SurfaceTexture(textureId)
val consumerSurface = Surface(surfaceTexture)
// Now you can pass `consumerSurface` to the Camera API!

```

For each frame, you call `surfaceTexture.updateTexImage()` before rendering to sync the latest content to the GPU. You must also call `surfaceTexture.getTransformMatrix(matrix)` to fetch the correct transformation matrix. This matrix handles device rotation and sensor cropping, and you must multiply your texture coordinates by this matrix in your vertex shader.

```kotlin
private val transformMatrix = FloatArray(16)

// Called every frame from your rendering thread (must have an active EGL context)
fun onDrawFrame() {
    // 1. Sync the latest hardware buffer to the OpenGL texture
    surfaceTexture.updateTexImage()
    
    // 2. Fetch the transformation matrix (handles sensor rotation and cropping)
    surfaceTexture.getTransformMatrix(transformMatrix)
    
    // 3. Pass the matrix and texture ID down to your C++ engine to draw
    nativeEngine.draw(textureId, transformMatrix)
}
```

Because you now have a GL texture object that points to the external image stream, you can sample it in your shader and apply post-processing effects:

```kotlin
// Fragment Shader
#extension GL_OES_EGL_image_external : require

precision mediump float;

// Use samplerExternalOES instead of sampler2D
uniform samplerExternalOES uCameraTexture;

// The texture coordinates MUST be multiplied by the transformMatrix in the Vertex Shader
varying vec2 vTexCoord; 

void main() {
    // Zero-copy sampling! The hardware handles YUV -> RGB conversion right here.
    vec4 cameraColor = texture2D(uCameraTexture, vTexCoord);
    
    // Apply any post-processing (e.g., Grayscale, LUTs, etc.)
    gl_FragColor = cameraColor; 
}
```
From the application's point of view, this avoids the expensive CPU readback and re-upload path. The engine samples the producer's buffer through `samplerExternalOES` instead of copying the frame into a regular `GL_TEXTURE_2D`.

However, you might notice a catch: `SurfaceTexture` explicitly relies on the `GL_TEXTURE_EXTERNAL_OES` OpenGL extension. What if our engine runs on Vulkan?

## ImageReader

The GLES path works because `SurfaceTexture` knows how to expose queued Android buffers as `GL_TEXTURE_EXTERNAL_OES`. Vulkan does not use `SurfaceTexture` this way. For a Vulkan backend, a more suitable path is to acquire Android `HardwareBuffer` objects and import them into Vulkan with the `VK_ANDROID_external_memory_android_hardware_buffer` extension.

(Note: You may also use `ImageReader` in GLES, but we emphasize Vulkan here because Vulkan can't go through `SurfaceTexture` path and could only use `ImageReader`)

A practical Android-side producer/consumer setup for this is `ImageReader` configured with `ImageFormat.PRIVATE` and GPU sampling usage.

First we create an `ImageReader` on the Kotlin side:

```kotlin
// 1. Create an ImageReader that requests hardware-backed buffers.
// The overload with usage flags requires API 29+.
// ImageFormat.PRIVATE allows the Android graphics allocator (Gralloc) to choose 
// the optimal hardware format for the GPU.
val imageReader = ImageReader.newInstance(
    width, height, 
    ImageFormat.PRIVATE, 
    2, // Max images in the queue
    HardwareBuffer.USAGE_GPU_SAMPLED_IMAGE // Tell Android this will be sampled by the GPU
)

// 2. Connect the Producer (Camera) to our new Consumer
captureRequestBuilder.addTarget(imageReader.surface)
```

To get the latest data, set a listener and call `reader.acquireLatestImage()` when a frame is available:

```kotlin
// Process frames as they arrive
imageReader.setOnImageAvailableListener({ reader ->
    // Grab the latest frame
    val image = reader.acquireLatestImage() ?: return@setOnImageAvailableListener
    
    // Extract the HardwareBuffer (API 28+).
    val hardwareBuffer = image.hardwareBuffer
    
    if (hardwareBuffer != null) {
        // Pass the hardware buffer across the JNI bridge to your C++ engine
        nativeEngine.drawWithHardwareBuffer(hardwareBuffer)
        
        // Close the hardware buffer at the end.
        hardwareBuffer.close()

        // Important: The native side must not keep using the Java `HardwareBuffer` object after the Java side closes it
        // unless it has acquired or imported the underlying native resource correctly.
    }
    // Close the image at the end.
    image.close()
}, backgroundHandler)
```

Note: The API levels are easy to mix up:

1. `AHardwareBuffer` exists on the native side from API 26.
2. `Image.hardwareBuffer` is available from API 28.
3. The `ImageReader.newInstance(..., usage)` overload used below is available from API 29.
4. Vulkan import requires `VK_ANDROID_external_memory_android_hardware_buffer`.

On the native engine side, the Vulkan flow is roughly:

1. Convert the Java `HardwareBuffer` into an `AHardwareBuffer`.
2. Query its Vulkan properties with `vkGetAndroidHardwareBufferPropertiesANDROID`.
3. Create a compatible `VkImage`.
4. Import the Android hardware buffer memory with `VkImportAndroidHardwareBufferInfoANDROID`.
5. Bind the imported memory to the `VkImage`.
6. Create the image view and sampler.
7. If the buffer uses an external or YUV format, configure the required external format and sampler YCbCr conversion path.
8. Synchronize producer and consumer access before sampling.

In pseudo-code, the native side looks like this:

```cpp
#include <android/hardware_buffer_jni.h>
#include <vulkan/vulkan.h>

void importHardwareBufferToVulkan(JNIEnv* env, jobject jHardwareBuffer) {
    AHardwareBuffer* hardwareBuffer =
            AHardwareBuffer_fromHardwareBuffer(env, jHardwareBuffer);
    if (!hardwareBuffer) {
        return;
    }

    // 1. Query the Vulkan-side properties of this Android hardware buffer.
    VkAndroidHardwareBufferPropertiesANDROID ahbProps = {
        .sType = VK_STRUCTURE_TYPE_ANDROID_HARDWARE_BUFFER_PROPERTIES_ANDROID,
    };

    vkGetAndroidHardwareBufferPropertiesANDROID(
            device,
            hardwareBuffer,
            &ahbProps);

    // 2. Create a VkImage compatible with the hardware buffer.
    // In real code, image format / external format / usage must match ahbProps.
    VkImageCreateInfo imageInfo = {
        .sType = VK_STRUCTURE_TYPE_IMAGE_CREATE_INFO,
        .imageType = VK_IMAGE_TYPE_2D,
        .extent = { width, height, 1 },
        .mipLevels = 1,
        .arrayLayers = 1,
        .samples = VK_SAMPLE_COUNT_1_BIT,
        .tiling = VK_IMAGE_TILING_OPTIMAL,
        .usage = VK_IMAGE_USAGE_SAMPLED_BIT,
    };

    VkImage image = VK_NULL_HANDLE;
    vkCreateImage(device, &imageInfo, nullptr, &image);

    // 3. Import the AHardwareBuffer as Vulkan device memory.
    VkImportAndroidHardwareBufferInfoANDROID importInfo = {
        .sType = VK_STRUCTURE_TYPE_IMPORT_ANDROID_HARDWARE_BUFFER_INFO_ANDROID,
        .buffer = hardwareBuffer,
    };

    VkMemoryDedicatedAllocateInfo dedicatedInfo = {
        .sType = VK_STRUCTURE_TYPE_MEMORY_DEDICATED_ALLOCATE_INFO,
        .pNext = &importInfo,
        .image = image,
    };

    VkMemoryAllocateInfo allocInfo = {
        .sType = VK_STRUCTURE_TYPE_MEMORY_ALLOCATE_INFO,
        .pNext = &dedicatedInfo,
        .allocationSize = ahbProps.allocationSize,
        .memoryTypeIndex = findCompatibleMemoryType(ahbProps.memoryTypeBits),
    };

    VkDeviceMemory memory = VK_NULL_HANDLE;
    vkAllocateMemory(device, &allocInfo, nullptr, &memory);
    vkBindImageMemory(device, image, memory, 0);

    // 4. Create image view, sampler, descriptor, and any required
    // sampler YCbCr conversion before sampling from the shader.
}
```

The important point is that the engine cannot assume the buffer is a normal RGBA image. The format and memory requirements must come from the Android hardware buffer properties.

Once the image, view, sampler, descriptor, and possible YCbCr conversion are set up correctly, the shader may look simple:

```glsl
// Vulkan Fragment Shader (GLSL)
#version 450

layout(binding = 0) uniform sampler2D uCameraTexture;

layout(location = 0) in vec2 vTexCoord;
layout(location = 0) out vec4 outColor;

void main() {
    outColor = texture(uCameraTexture, vTexCoord);
}
```

## Integration to the Engine Examples

### Filament

In Filament, there are two ways you can import external textures. The first one is the ordinary `Texture` class. You create a `Texture` object but specify it is an `external()` resource. After that, you must call `setExternalImage()` to give a handle of the external image, and also call it whenever there is an update to the image.

The second one is a `Stream`, where the filament `Texture` automatically updates its content from the external source, so that you don't have to update the image by yourselves.

### Same as 

https://docs.unity3d.com/ScriptReference/Texture2D.CreateExternalTexture.html

// Godot?


## References

https://source.android.com/docs/core/graphics/arch-bq-gralloc

https://source.android.com/docs/core/graphics/arch-st
