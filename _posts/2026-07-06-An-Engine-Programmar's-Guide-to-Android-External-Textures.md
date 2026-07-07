---
title: An Engine Programmer’s Guide to Android External Textures
date: 2026-05-30 00:53:35 +0800
categories: [Graphics]
tags: [Android, Graphics]
---

If you're an engine progrmmar (or you're working with a game/rendering engine) and the engine supports Android platform, you have likely heard of "External Textures." But what exactly is it? and how does it work under the hood? But what exactly are they, and how do they work under the hood? In this article, I will walk you through the core mechanics of external textures on Android.

By the end of this guide, you will understand the requirements for integrating live camera and video streams into an Android-supported engine, and how to properly interface with Android's native graphics components.

## What are External Textures?

When developing graphics applications on Android, you often need to ingest frames from a camera or a media decoder (video), apply custom processing, and render them to the screen. However, hardware components like the camera sensor or video decoder write their output into specialized, hardware-backed memory (typically in a YUV color format)(TODO: ref). This raw data is not easily accessible by standard application RAM, and the YUV format is not natively friendly to standard GPU rendering pipelines.

If you try to process this without an external texture, the CPU is forced to execute two expensive steps for every single frame:

1. Copy the massive pixel data from hardware memory into standard RAM.
2. Convert the data from YUV into a GPU-friendly RGB format.

Executing this on the CPU every frame destroys rendering performance, drains the battery, and quickly overheats the device. This is exactly the bottleneck External Textures were introduced to solve.

### Benefits of External Textures

An External Texture creates a bridge that allows the GPU to map and read directly from the hardware buffer. This creates a zero-copy pipeline, bypassing the CPU entirely so no data needs to be moved. Even better, when your shader samples an external texture, the graphics hardware automatically handles the YUV-to-RGB color space conversion on the fly (TODO: ref). You get hardware-accelerated color conversion without writing complex decoding logic or incurring additional performance costs.

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

Note: You can also do entirely in C++, but for this article, I'm going to assume we have a Java/Kotlin app with a native C++ rendering engine library under the hood to align with Filament :P


Notice what is happening in the code above: the Camera API only asks for a `Surface`. A `Surface` in Android is essentially a handle to a buffer queue. The Camera (the Producer) does not know, nor does it care, where those pixels are actually going. It just pushes frames into the Surface.

In Android, when you want to process camera frames, you generally have to choose between two main types of Consumers to create that `Surface`: an `ImageReader` or a `SurfaceTexture`, each serving a fundamentally different pipeline:
- SurfaceTexture (for GPU)
- ImageReader (for CPU)

## SurfaceTexture

`SurfaceTexture` is the consumer you want for rendering. It provides a `Surface` that is backed directly by an OpenGL ES texture, which is the concrete implementation of the "External Texture" concept we discussed earlier. 

Basically, you generate an OpenGL texture ID and bind it to `GL_TEXTURE_EXTERNAL_OES`. You then pass that texture ID into a new `SurfaceTexture`, which you can finally wrap in a standard Android `Surface`.

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

Because you now have a standard OpenGL texture ID, you can easily apply post-processing effects and custom shaders to the external texture as below:

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
And, as we mentioned earlier, this process is entirely zero-copy, and the hardware handles the YUV-to-RGB conversion automatically during the texture sampling phase.

However, you might notice a catch: `SurfaceTexture` explicitly relies on the `GL_TEXTURE_EXTERNAL_OES` OpenGL extension. What if our engine runs on Vulkan? Or what if we don't need to apply any extra shader effects to these frames at all?
  
## ImageReader

Unlike `SurfaceTexture`, we don't need to generate GL ID when using `ImageReader`. Instead, we can access the image byte buffer with CPU directly, and we can also control how many images we want to hold in the queue at the same time.

```kotlin
// 1. Create an ImageReader that requests hardware-backed buffers (API 26+)
// ImageFormat.PRIVATE allows the Android graphics allocator (Gralloc) to choose 
// the optimal hardware format for the GPU.
val imageReader = ImageReader.newInstance(
    width, height, 
    ImageFormat.PRIVATE, 
    2, // Max images in the queue
)

// 2. Connect the Producer (Camera) to our new Consumer
captureRequestBuilder.addTarget(imageReader.surface)
```

To get the latest data, we will set the listen to call `reader.acquireLatestImage()` when available:

```kotlin
// Process frames as they arrive
imageReader.setOnImageAvailableListener({ reader ->
    // Grab the latest frame
    val image = reader.acquireLatestImage() ?: return@setOnImageAvailableListener
    
    // Do something...

    // Close the image at the end.
    image.close()
}, backgroundHandler)
```

When you want to save the camera data as a bitmap to the disk, or something you don't need GPU engaged, it is fairly useful. But here I want to emphasize another case: rendering with Vulkan.
As we mentioned before, there is no `SurfaceTexture` extension in Vulkan, so we need to leverage `ImageReader`. Then you might be worried about the performance, because `ImageReader` would stream the data to CPU, and if we want to render with Vulkan, that means we need to upload it onto GPU again, and lose the zero-copy benefit! However, the good news is, we don't have to do so (if you have API version 24+ and `VK_ANDROID_external_memory_android_hardware_buffer` entension).

Back to the creation of `ImageReader`, we can let Android know we are gonna sample the texture with GPU:

```kotlin
val imageReader = ImageReader.newInstance(
    width, height, 
    ImageFormat.PRIVATE, 
    2, // Max images in the queue
    HardwareBuffer.USAGE_GPU_SAMPLED_IMAGE // <-- !!New: Tell Android this will be sampled by the GPU
)

...(same as above)
```

And when registerign the listener, we can get the hardware buffer after we acquire the new image:
```kotlin
imageReader.setOnImageAvailableListener({ reader ->
    // Grab the latest frame
    val image = reader.acquireLatestImage() ?: return@setOnImageAvailableListener
    
    // <--!!New: Extract the AHardwareBuffer (available since API 26)
    val hardwareBuffer = image.hardwareBuffer
    
    if (hardwareBuffer != null) {
        // <--!!New: Pass the hardware buffer across the JNI bridge to your C++ engine
        nativeEngine.drawWithHardwareBuffer(hardwareBuffer)
        
        // <--!!New: Close the hardware buffer at the end.
        hardwareBuffer.close()
    }
    // Close the image at the end.
    image.close()
}, backgroundHandler)
```

On the native engine side, you may do something like:

```C++
#include <android/hardware_buffer_jni.h>
#include <vulkan/vulkan.h>

extern "C" JNIEXPORT void JNICALL
Java_com_your_app_NativeEngine_drawWithHardwareBuffer(JNIEnv *env, jobject instance, jobject jHardwareBuffer) {
    
    // 1. Convert the Java HardwareBuffer into a native AHardwareBuffer pointer
    AHardwareBuffer* hardwareBuffer = AHardwareBuffer_fromHardwareBuffer(env, jHardwareBuffer);
    
    if (!hardwareBuffer) return;

    // 2. Setup Vulkan structures to import the hardware buffer memory.
    // (Ensure your Vulkan instance enabled VK_ANDROID_external_memory_android_hardware_buffer)
    VkImportAndroidHardwareBufferInfoANDROID importInfo = {};
    importInfo.sType = VK_STRUCTURE_TYPE_IMPORT_ANDROID_HARDWARE_BUFFER_INFO_ANDROID;
    importInfo.pNext = nullptr;
    importInfo.buffer = hardwareBuffer;
    
    VkMemoryAllocateInfo allocInfo = {};
    allocInfo.sType = VK_STRUCTURE_TYPE_MEMORY_ALLOCATE_INFO;
    allocInfo.pNext = &importInfo;
    
    // Note: In a real engine, you must use vkGetAndroidHardwareBufferPropertiesANDROID 
    // to query the correct memory type index and allocation size before allocating.
    
    // 3. Allocate Vulkan memory backed directly by the camera's hardware buffer
    vkAllocateMemory(device, &allocInfo, nullptr, &deviceMemory);
    
    // 4. Bind this memory to a VkImage, create a VkImageView, and sample it.
    // ... standard Vulkan rendering logic ...
}
```

After that, you can enjoy your zero-copy image and zero-cost YUV to RGB conversion in the shader with no additional modification!

```
// Vulkan Fragment Shader (GLSL)
#version 450

// Notice: No special extensions required!
// Just a standard 2D sampler.
layout(binding = 0) uniform sampler2D uCameraTexture;

layout(location = 0) in vec2 vTexCoord;
layout(location = 0) out vec4 outColor;

void main() {
    // The hardware still handles the YUV -> RGB conversion for you,
    // but the shader just treats it like a normal RGBA texture fetch.
    outColor = texture(uCameraTexture, vTexCoord);
}
```

## Integration to the Engine Examples

### Filament

// TODO

// Unity?
// Godot?


## References

[1] https://source.android.com/docs/core/graphics/arch-bq-gralloc

https://dev.epicgames.com/community/learning/knowledge-base/KKKp/unreal-engine-how-to-create-external-textures-on-android

https://source.android.com/docs/core/graphics/arch-st