# OIT

Assume **MAX_OIT_LAYERS**(MAX_OIT_LAYERS is the number of layers of semi-transparent pixels that are accurately sorted using OIT) is **K**, and transparent objects are still sorted from far to near. 

## The rendering process is as follows:

#### **Create necessary textures and buffers**

- Allocate a **uint32 Fragment Count Texture(ROV)** to count fragments per pixel atomically
- Allocate a **uint32 Fragment Buffer** to store fragment data
- Create a **single-channel Sample Count texture** to record the number of transparent fragments per pixel

#### **Pre Transparent Pass**

- Set the render target to **Sample Count**
- Set the blend mode to **Add One One**
- In the **Pixel Shader**, set **OutColor = 1** so that **Sample Count** records the number of fragments per pixel

#### **Regular Transparent Pass**

- For each pixel, accumulate the **Fragment Count**
- If **Sample Count - Fragment Count** is less than **K**, store the fragment data into the **Fragment Buffer**
- Otherwise, blend the fragment directly into the background

#### **Sort-Resolve Screen Pass**

- Execute a full-screen pass to read data from the **Fragment Count Buffer**
- Perform sorting and blending operations
- Generate the final **Scene Color**



## **Optimizing Memory Usage with an Offset Array**

- In real-world scenarios, many pixels on the screen have far fewer transparent fragments than **K**, or do not reach **K** at all
- As a result, a significant portion of memory in the **Fragment Buffer** is wasted
- To solve this, each pixel can store an **Offset** value in the array
- This allows **Fragment Data** to be stored **compactly** in the **Fragment Buffer** based on the **Offset**, reducing memory waste

![Demonstration of the Offset Array optimization](https://github.com/moonlovelj/OrderIndependentTranslucency/blob/main/Images/OffsetArray.png)



## Fast Prefix Sum Computation

![Demonstration of the prefix sum](https://github.com/moonlovelj/OrderIndependentTranslucency/blob/main/Images/PrefixSum.png)



## Render result comparison

Below is a comparison of render quality and performance, where approximately 40 white smoke particles are placed overlappingly  to test performance. The test machine is configured with an AMD 9700X + 4070 Ti + 64GB RAM, and the screen resolution is 1920x1080.

- **OIT Off, FPS:120, Obvious blending errors occur**

![OIT Off](https://github.com/moonlovelj/OrderIndependentTranslucency/blob/main/Images/OITOff.gif)

- **UE5 MLAB with MAX_OIT_LAYERS=16, FPS: 58, Obvious artifact occur**

![UE5 MLAB](https://github.com/moonlovelj/OrderIndependentTranslucency/blob/main/Images/UE5MLAB.gif)

- **My custom OIT with MAX_OIT_LAYERS=16, FPS: 88, without obvious artifact**

![My custom OIT with MAX_OIT_LAYERS=16](https://github.com/moonlovelj/OrderIndependentTranslucency/blob/main/Images/OIT16.gif)

- **My custom OIT with MAX_OIT_LAYERS=32, FPS: 75, better quality, without obvious artifact**

![My custom OIT with MAX_OIT_LAYERS=32](https://github.com/moonlovelj/OrderIndependentTranslucency/blob/main/Images/OIT32.gif)

As can be seen from the above test that my custom OIT solution only exhibits some minor visual artifacts at the edges of the smoke due to excessive layer stacking. However, this scene is intentionally set up with overlapping particle smoke to test performance, which rarely occurs in actual production scenarios.



## Demo Video

[![OIT Demo](https://github.com/moonlovelj/OrderIndependentTranslucency/blob/main/Images/OIT.jpg)](https://www.youtube.com/watch?v=Mil08oTn4mU)



## Future Work

- More efficient prefix sum computation

- Smarter CPU memory allocation after GPU prefix sum computation (since the data is read back to the CPU asynchronously, the exact memory required for the current frame cannot be guaranteed in time)

  

## Conclusion

On platforms that support ROV, this custom OIT method can achieve accurate OIT for the nearest K layers (K can support up to 32), while blending layers beyond K using conventional distance-based blending. This ensures correct blending for nearby layers without losing distant content. Additionally, when dealing with a large number of transparent layers, this method is significantly more efficient and has better quality than both UE5's MLAB method and the linked list approach. And compared to MLAB, it is compatible with all blending modes.
