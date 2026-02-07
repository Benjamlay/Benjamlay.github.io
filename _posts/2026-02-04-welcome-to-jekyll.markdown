---
layout: post
title:  "My OpenGL 3D Scene"
date:   2026-02-04 14:10:09 +0100
categories: jekyll update
---

# Building an OpenGL 3D Scene

This project was developed as a key component of my curriculum to demonstrate a foundational understanding of modern real-time rendering. My goal was simple but challenging: **to use C++ and OpenGL to move beyond basic geometry and build a photorealistic 3D scene**.

This post isn't just a technical breakdown, it’s my personal journey of going from rendering a simple triangle to creating a photorealistic scene. We’ll explore how core concepts like **Deferred Shading**, **PBR** (Physically Based Rendering), and **SSAO** come together to transform raw geometry into a complex, beautifully lit environment, all while learning the ins and outs of the GPU.

---

## The Rendering Pipeline (Deferred Shading)

**Deferred Shading** is crucial for performance in complex scenes with many light sources. By decoupling geometry and lighting calculations, it allows adding many lights without a prohibitive performance hit, which is interesting for building a modern 3D engine.

### 💡 How it works

It separates the geometry calculation from the lighting calculation.

1.  **Geometry Pass:** The GPU stores essential information like *Base Color*, *Normal*, and *Position* for each fragment into a set of textures called the **G-Buffer**.
2.  **Lighting Pass:** The lighting is calculated only once per screen pixel using the data stored in the G-Buffer, regardless of the number of objects contributing to that pixel.

| Base Color | Normal Map | Position Map |
|:---:|:---:|:---:|
| ![](/images/BaseColorMapPicture.png) | ![](/images/NormalMapPicture.jpg) | ![](/images/PositionMapPicture.jpg) |
| *Base Color map* | *Normal map* | *Position map* |

---

## Shadow Mapping

Dynamic shadows are fundamental for grounding objects and adding realism and depth to a 3D scene. **Shadow mapping** is a widely used and relatively efficient technique for achieving this.

### 💡 How it works

The process involves a **two-pass strategy**:

1.  First, the entire scene is rendered from the light's point of view to create a **Depth Map** (or shadow map), which stores the depth of the closest surface to the light.
2.  Second, during the final rendering pass, the depth of the current pixel (from the camera's view) is compared to the corresponding depth value stored in the Depth Map.

> If the pixel's depth is greater than the depth in the map, the pixel is considered to be in shadow.

```glsl
float ShadowCalculation(vec4 fragPosLightSpace, vec3 normal, vec3 lightDir, sampler2D shadowMap)
{
    vec3 projCoords = fragPosLightSpace.xyz / fragPosLightSpace.w;
    projCoords = projCoords * 0.5 + 0.5; // Transform to [0,1] range

    if(projCoords.z > 1.0)
        return 0.0;

    float currentDepth = projCoords.z;
    
    // Bias calculation to prevent shadow acne
    float bias = max(0.005 * (1.0 - dot(normal, lightDir)), 0.001);

    float shadow = 0.0;
    vec2 texelSize = 1.0 / vec2(textureSize(shadowMap, 0));
    
    // PCF (Percentage-Closer Filtering) loop
    for(int x = -1; x <= 1; ++x)
    {
        for(int y = -1; y <= 1; ++y)
        {
            float pcfDepth = texture(shadowMap, projCoords.xy + vec2(x, y) * texelSize).r;
            shadow += currentDepth - bias > pcfDepth ? 1.0 : 0.0;
        }
    }
    shadow /= 9.0;

    return shadow;
}
```

| Light 1 POV | Light 2 POV |
|:---:|:---:|
| ![](/images/shadowmapPicture.jpg) | ![](/images/shadowmapPicture2.jpg) |
| *First shadow map* | *Second shadow map* |

---

## Physically Based Rendering (PBR) & IBL

**PBR** is a modern standard for achieving photorealism because it accurately models how light interacts with surfaces based on physical principles (like energy conservation).

**IBL (Image Based Lighting)** is essential to make materials like metal look correct by allowing them to reflect the environment naturally, which greatly enhances scene believability.

### 💡 How it works

PBR uses a physically accurate rendering equation, typically the **Cook-Torrance BRDF**, which is split into diffuse and specular components (Distribution, Fresnel, Geometry).

Instead of a simple ambient color, a **360° environment map** (often an HDR skybox) is pre-processed into two components:
1.  **Irradiance Map:** For diffuse ambient light.
2.  **Pre-Filter Map:** For blurry, rough specular reflections.

> You can notice below that the metallic sphere is reflecting the brick building.

| The Sphere | The Skybox |
|:---:|:---:|
| ![](/images/sphereMetallic.png) | ![](/images/buildingFromSkybox.png) |
| *Metallic Reflection* | *Source Environment* |

| Irradiance Map | Pre-Filter Map |
|:---:|:---:|
| ![](/images/IrradianceMapPicture.jpg) | ![](/images/PreFilterMapPicture.jpg) |
| *Diffuse Component* | *Specular Component* |

---

## Post-Processing: The Bloom Effect

The **Bloom effect** is a post-processing technique that adds a final layer of polish and visual fidelity. It simulates the optical effect where extremely bright light sources appear to bleed or glow on camera, making the lighting in the scene feel more intense and photographic.

### 💡 How it works

The process involves three main steps:

1.  **Extraction:** First, a step isolates only the brightest areas (above a certain threshold) of the rendered scene.
2.  **Blur:** Second, a blur is applied to this extracted image (often a Gaussian blur) to create the glow effect.
3.  **Re-injection:** Third, this blurred image (BloomMap) is re-injected (blended) back into the original final scene image.

| Bloom Map | Final Result |
|:---:|:---:|
| ![](/images/BloomMapPicture.jpg) | ![](/images/gifBloomEffect1.gif) |
| *Extracted Glow* | *Final Bloom Effect* |

---

## 📐 SSAO (Screen Space Ambient Occlusion)

**SSAO** is a crucial technique for enhancing the visual realism and depth of a 3D scene. It simulates subtle, non-directional shadows that occur where objects are close to each other (e.g., in corners, crevices, or under edges).

* **Efficiency:** This grounding effect is computationally inexpensive because it's calculated in "screen space" (after the geometry pass).

### 💡 How it works

SSAO approximates ambient occlusion by sampling the depth and normal information around a given pixel on the screen.

* For each pixel, the algorithm checks a small neighborhood of surrounding pixels.
* If the neighboring pixels are closer to the camera (i.e., higher depth value) than the current pixel, it means the current pixel is likely occluded and should receive less ambient light.
* This calculation uses **random sampling kernels** and a noise texture to distribute samples and reduce banding artifacts.

![SSAO Map](/images/SSAOMapPicture.jpg)
<p align="center"><i>SSAO Map showing occlusion zones</i></p>

---

## Frustum Culling

Rendering every single object in a scene, even those behind the camera or far in the distance, is a massive waste of computational power. **Frustum Culling** is a vital optimization technique used to solve this.

### 💡 How it works

The camera's view is shaped like a pyramid with the top cut off, known as the **View Frustum**. It is defined by the field of view, the aspect ratio, and the near and far clipping planes.

1.  **Bounding Volume:** Every object in the scene is wrapped in a simplified invisible shell, usually a box (AABB) or a sphere.
2.  **Intersection Test:** Before sending any geometry to the GPU, the engine checks if this bounding volume intersects with the camera's frustum.
3.  **Cull or Keep:**
    * **Outside:** If the object is outside the frustum, it is ignored (culled) completely.
    * **Inside:** If it is inside or touching the frustum, it is sent to the rendering pipeline.

> This technique ensures that the GPU only processes what the player can actually see, resulting in a significant performance boost for complex scenes.