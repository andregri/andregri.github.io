---
layout: post
author: andrea
tags: opengl
---

## Chapter1

- OpenGL is an interface that your application can use to access and control the graphics subsystem of the device on which it runs. These standard interfaces are called application programming interfaces (APIs), of which OpenGL is one.

- OpenGL Architectural Review Board (**ARB**)

- Current GPUs consist of large numbers of small programmable processors called **shader cores** that run mini-programs called **shaders**. Each core has a relatively low throughput, processing a single instruction of the shader in one or more clock cycles and normally lacking advanced features such as out-of-order execution, branch prediction, super-scalar issues, and so on. However, each GPU might contain anywhere from a few tens to a few thousands of these cores.

- The graphics system is broken into a number of **stages**, each represented either by a shader (which is programmable, which means that it executes the shader that you supply) or by a **fixed-function** (it’s just that you don’t supply that code, but rather the GPU manufacturer generally supplies it as part of a driver, firmware, or other system software).

- (Vertex Fetch) -> [Vertex Shader] -> [Tassellation Control Shader] -> (Tassellation) -> [Tassellation Evaluation Shader] -> [Geometry Shader] -> (Rasterization) -> [Fragment Shader] -> (Framebuffer Operations)

  (Fixed-Function) [Shader]

- In 2008, the ARB decided it would “fork” the OpenGL specification into two profiles. The first is the modern, **core profile**, which removes a number of legacy features. The **compatibility profile** maintains backward compatibility with all revisions of OpenGL back to version 1.0.

- The fundamental unit of rendering in OpenGL is known as the **primitive**. OpenGL supports many types of primitives, but the three basic renderable primitive types are points, lines, and triangles. As polygons, triangles are always **convex**, so filling rules are easy to devise and follow. **Concave** polygons can always be broken down into two or more triangles.

- The **rasterizer** is dedicated hardware that converts the three-dimensional representation of a triangle into a series of pixels that need to be drawn onto the screen.

- Points, lines, and triangles are formed from collections of one, two, or three vertices, respectively. A **vertex** is simply a point within a coordinate space.

- The graphics pipeline is broken down into two major parts:
	
	1. The first part, often known as the **front end**, processes vertices and primitives, eventually forming them into the points, lines, and triangles that will be handed off to the rasterizer. This is known as **primitive assembly**. After going through the rasterizer, the geometry has been converted from what is essentially a **vector representation** into a large number of independent **pixels**.
	
	2.  Pixels are handed off by the **back end**, which includes depth and stencil testing, fragment shading, blending, and updating of the output image.
	
## Chapter 2

- OpenGL works by connecting a number of mini-programs called **shaders** together with **fixed-function glue**. Usually, the output of the graphic pipeline is pixels.

- OpenGL shaders are written in a language called the **OpenGL Shading Language**, or **GLSL**. It is similar to C.

- The source code for your shader is placed into a **shader object** and **compiled**, and then multiple shader objects can be **linked** together to form a **program object**.

	shader source -> shader object -> compiler -> linker -> program object

- Each program object can contain shaders for one or more shader stages:

  - vertex shaders
  - tessellation control
  - evaluation shaders
  - geometry shaders
  - fragment shaders
  - compute shaders
  
- The minimal pipeline consists of only a **vertex shader** (or a compute shader) but if you wish to see any pixels on the screen, you also need a **fragment shader**.

- <code class="glsl hljs inline">#version 450 core</code> tells the shader compiler that we intend to use version 4.5 of the shading language. The keyword <code class="glsl hljs inline">core</code> to indicate that we intend to use only features from the core profile of OpenGL.

- the declaration of our <code class="glsl hljs inline">main</code> function, which is where the shader starts executing. Pay attention: it must return void!

- All variables that start with **gl_** are part of OpenGL and connect shaders to each other or to the various parts of fixed functionality in OpenGL.

- In the vertex shader, **gl_Position** represents the output position of the vertex in the OpenGL’s **clip space**, which is the coordinate system expected by the next stage of the OpenGL pipeline. gl_Position is a built-in variables provided by GLSL.

- In fragment shaders, the value of **output variables** will be sent to the window or screen.

- Create a **vertex array object (VAO)**, which is an object that represents the **vertex fetch stage** of the OpenGL pipeline and is used to supply input to the vertex shader.

- To create the VAO, we call the OpenGL function **glCreateVertexArrays()**; to attach it to our context, we call **glBindVertexArray()**.

1. Most things in OpenGL are represented by objects (like vertex array objects) and we create them using a **creation function**.

2. Then let OpenGL know that we want to use them by binding them to the context using a **binding function**.

- **glUseProgram()** to tell OpenGL to use our program object for rendering.

- The **glDrawArrays()** function sends vertices into the OpenGL pipeline. For each vertex, the vertex shader is executed. The <code class="glsl hljs inline">mode</code> parameter, which tells OpenGL what type of graphics primitive we want to render.

- GLSL includes a special input to the vertex shader called **gl_VertexID**, which is the index of the vertex that is being processed at the time. The gl_VertexID input starts counting from the value given by the <code class="glsl hljs inline">first</code> parameter of glDrawArrays() and counts upward one vertex at a time for <code class="glsl hljs inline">count</code> vertices (the third parameter of glDrawArrays()).

- We can use gl_VertexID to assign a different position to each vertex

## Chapter 3 - The OpenGL pipeline

- In GLSL, the mechanism for getting data in and out of shaders is to declare global variables with the <code class="glsl hljs inline">in</code> and <code class="glsl hljs inline">out</code> storage qualifiers. <code class="glsl hljs inline">in</code> and <code class="glsl hljs inline">out</code> can be used to form **conduits** from shader to shader and pass data between them. Anything you write to an output variable in one shader is sent to a similarly named variable declared with the in keyword in the subsequent stage.

- Reading and writing **built-in variables** such as gl_VertexID and gl_Position allow to communicate with fixed-function blocks.

1. The **vertex fetching** or **vertex pulling** is the first stage and it is a fixed-function stage. Provides inputs to the vertex shader.

2. The **vertex shader** is the first programmable stage in the OpenGL pipeline. We use the <code class="glsl hljs inline">in</code> keyword to bring inputs into the vertex shader. It is automatically filled in by the fixed-function vertex fetch stage. The variable becomes known as a **vertex attribute**.

- We can tell vertex fetch stage what to fill the variable with by using one of the many variants of the vertex attribute functions, **glVertexAttrib_(GLuint index, _)**.

- In the vertex shader, <code class="glsl hljs inline">layout (location = 0) in vec4 offset;</code> is an input to the vertex shader. This is a **layout qualifier**, which we have used to set the location of the vertex attribute to zero. This location is the value we’ll pass in <code class="glsl hljs inline">index</code> to refer to the attribute.

- Each time we call one of the glVertexAttrib*() functions (of which there are many), it will update the value of the vertex attribute that is passed to the vertex shader.

- We can group together a number of variables into an **interface block** to communicate a number of different pieces of data between stages.

- The interface block has both a **block name** (VS_OUT, uppercase) and an **instance name** (vs_out, lowercase). Interface blocks are matched between stages using the block name (VS_OUT in this case), but are referenced in shaders using the instance name.

-  Note that interface blocks are **only for moving data** from shader stage to shader stage — you can’t use them to group together inputs to the vertex shader or outputs from the fragment shader.

- **Tessellation** is the process of breaking a **high-order primitive** (which is known as a **patch** in OpenGL) into many smaller, simpler primitives such as triangles for rendering.

- The tessellation phase sits directly after the vertex shading stage in the OpenGL pipeline and is made up of three parts:

	- the tessellation control shader (TCS) or control shader,
	- the fixed-function tessellation engine,
	- the tessellation evaluation shader (TES) or evaluation shader.

3. The **control shader** takes its input from the vertex shader and is primarily responsible for determine the level of tessellation to be sent to the tessellation engine and the generation of data to be sent to the tessellation evaluation shader.

- Tessellation in OpenGL works by breaking down high-order surfaces known as patches into points, lines, or triangles. Each patch is formed from a number of **control points**. The number of control points per patch is configurable and set by calling **glPatchParameteri()**. By default, the number of control points per patch is three. The maximum number of control points that can be used to form a single patch is implementation defined.

- The vertex shader runs once per control point, while the tessellation control shader runs in **batches** on groups of control points where the **size of each batch** is the same as the number of vertices per patch.

- The number of control points per patch can be changed such that the number of control points that is output by the tessellation control shader can differ from the number of control points that it consumes. The number of control points produced by the control shader is set using an output layout qualifier in the control shader’s source code: **layout (vertices = N) out;** where, N is the number of control points per patch.

- The control shader is responsible also for setting the **tessellation factors** for the resulting patch that will be sent to the fixed-function tessellation engine through the **gl_TessLevelInner** and **gl_TessLevelOuter** built-in output variables. Higher numbers would produce a more densely tessellated output, and lower numbers would yield a more coarsely tessellated output.

- The built-in input variable **gl_InvocationID** contains the zero-based index of the control point within the patch being processed by the current invocation of the tessellation control shader.

4. The **tessellation engine** is a fixed-function part of the OpenGL pipeline that takes high-order surfaces represented as patches and breaks them down into simpler primitives such as points, lines, or triangles.

- The tessellation control shader processes the incoming control points and sets tessellation factors that are used to break down the patch. After the tessellation engine produces the output primitives, the vertices representing them are picked up by the tessellation evaluation shader.

- It produces a number of output vertices representing the primitives it has generated.

5. The **evaluation shader** runs an invocation for each vertex produced by the tessellator. When the tessellation levels are high, the tessellation evaluation shader could run an extremely large number of times.

- **gl_TessCoord** is the barycentric coordinate of the vertex generated by the tessellator.

- To see the results of the tessellator, we need to tell OpenGL to draw only the outlines of the resulting triangles **glPolygonMode()**.