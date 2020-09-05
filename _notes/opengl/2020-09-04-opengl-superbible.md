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
