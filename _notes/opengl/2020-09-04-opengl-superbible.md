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