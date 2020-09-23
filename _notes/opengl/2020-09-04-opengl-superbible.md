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

6. The **geometry shader** is the last shader stage before the rasterizer. The geometry shader **runs once per primitive** and has access to all of the input vertex data for all of the vertices that make up the primitive being processed.

- The geometry shader is also unique among the shader stages in that it is able to increase or reduce the amount of data flowing through the pipeline in a programmatic way.  The tessellation shader can also modify the amount of data through the tessellation level but in an implicit way.

- Geometry shaders include two functions—**EmitVertex()** and **EndPrimitive()**—that explicitly produce vertices that are sent to primitive assembly and rasterization.

- After the front-end has run (which includes vertex shading, tessellation, and geometry shading), a fixed-function part of the pipeline performs a series of tasks that take the vertex representation of our scene and convert it into a series of **pixels**, which in turn need to be colored and written to the screen.

7. The first step in this process is **primitive assembly**, which is the grouping of vertices into lines and triangles.

8. Once primitives have been constructed from their individual vertices, they are **clipped** against the displayable region.

- As vertices exit the front end of the pipeline, their position is said to be in **clip space**. A vertex has a vec4 type, known as a **homogeneous coordinate**, which is used in **projective geometry** because much of the math simpler than in regular Cartesian space.

- Although the output of the front end is a four-component homogeneous coordinate, **clipping occurs in Cartesian space**.

- To convert from homogeneous coordinates to Cartesian coordinates, **OpenGL performs a perspective division**, which involves dividing all four components of the position by the last, w component.

- After the projective division, the resulting position is in **normalized device space**: in OpenGL, the visible region of normalized device space is the volume that extends from −1.0 to 1.0 in the x and y dimensions and from 0.0 to 1.0 in the z dimension. [-1.0; 1.0] x [-1.0; 1.0] x [0.0; 1.0]

- Any geometry that is contained in this region may become visible to the user and anything outside of it should be discarded.

- If a primitive’s vertices all lie on the “outside” of any one plane, then the whole thing is thrown away. If all of primitive’s vertices are on the “inside” of all the planes (and therefore inside the view volume), then it is passed through unaltered. 

- Primitives that are partially visible (which means that they cross one of the planes) must be handled specially.

- However, the window that you’re drawing to has coordinates that usually start from (0, 0) at the bottom left and range to (w − 1,h − 1), where w and h are the width and height of the window in pixels.

- To place your geometry into the window, OpenGL applies the **viewport transform**, which applies a scale and offset to the vertices’ normalized device coordinates to move them into **window coordinates**.

- The scale and bias to apply are determined by the viewport bounds, which you can set by calling **glViewport()** and **glDepthRange()**.

- Before a triangle is processed further, it may be optionally passed through a stage called **culling**, which determines whether the triangle faces toward (**front-facing**) or away (**back-facing**) from the viewer and can decide whether to actually go ahead and draw it based on the result of this computation.

- It is very common to discard triangles that are back-facing because when an object is closed, any back-facing triangle will be hidden by another front-facing triangle. **glFrontFace()**

- To determine whether a triangle is front- or back-facing, OpenGL will determine its **signed area** in window space.

- By default, OpenGL will render all triangles, regardless of which way they face. To turn on culling, call **glEnable()** with cap set to **GL_CULL_FACE**. To change which types of triangles are culled, call **glCullFace()** with face set to GL_FRONT, GL_BACK, or GL_FRONT_AND_BACK.

9. Finally, the parts of the primitive that are determined to be potentially visible are sent to a fixed-function subsystem called the **rasterizer**. This block determines which pixels are covered by the primitive (point, line, or triangle) and sends the list of pixels on to the next stage—that is, fragment shading.

- Rasterization is the process of determining which fragments might be covered by a primitive such as a line or a triangle.

10. The **fragment shader** is the last programmable stage in OpenGL’s graphics pipeline. This stage is responsible for determining the color of each fragment before it is sent to the framebuffer for possible composition into the window.

- After the rasterizer processes a primitive, it produces a list of fragments that need to be colored and passes this list to the fragment shader.

- The term **fragment** is used to describe an element that may ultimately contribute to the final color of a pixel. The pixel may not end up being the color produced by any particular invocation of the fragment shader due to a number of other effects such as depth or stencil tests, blending, and multi-sampling.

- In a real-world application, the fragment shader would normally be substantially more complex and be responsible for performing calculations related to lighting, applying materials, and even determining the depth of the fragment.

- The fragment shader has several built-in variables such as **gl_FragCoord**, which contains the position of the fragment within the window.

11. The **framebuffer** is the last stage of the OpenGL graphics pipeline. It can represent the visible content of the screen and a number of additional regions of memory that are used to store per-pixel values other than color.

- The **state** held by the framebuffer includes information such as where the data produced by your fragment shader should be written, what the format of that data should be, etc.. It is stored in a framebuffer object.

- Another component of the framebuffer is the **pixel operation state** but it is not part of the framebuffer object:

	- **scissor test**: which tests your fragment against a rectangle that you can define. If it’s inside the rectangle, then it will be processed further; if it’s outside, it will be thrown away.
	
	- **stencil test**: compare the value provided by the application with the value in the stencil buffer. This buffer has one value for each pixel.
	
	- **depth test**: compares the fragment’s z coordinate against the contents of the depth buffer. This buffer has a value for each pixel; it contains the depth (which is related to distance from the viewer) of each pixel.

	- Next, the fragment’s color is sent to either the **blending or logical operation stage** depending whether it is floating-point or normalized integer values.
	
- OpenGL is capable of using a wide range of functions that take components of the output of your fragment shader and of the current content of the framebuffer and calculate new values that are written back to the framebuffer. 

- OpenGL also includes the **compute shader** stage, which can almost be thought of as a separate pipeline that runs indepdendently of the other graphics-oriented stages. Compute shaders are a way of getting at the computational power possessed by the graphics processor in the system.

- Each compute shader operates on a single unit of work known as a **work item**.

- These items are, in turn, collected together into small groups called **local workgroups**.

- Collections of these workgroups can be sent into OpenGL’s compute pipeline to be processed.

- All processing performed by a compute shader is explicitly written to memory by the shader itself, rather than being consumed by a subsequent pipeline stage.

- To compile one, you create a shader object with the type GL_COMPUTE_SHADER, attach your GLSL source code to it with glShaderSource(), compile it with glCompileShader(), and then link it into a program with glAttachShader() and glLinkProgram(). The result is a program object with a compiled compute shader in it that can be launched to do work for you.

- One of OpenGL’s greatest strengths is that it can be **extended** and enhanced by hardware manufacturers, operating system vendors, and even publishers of tools and debuggers.

- **Extensions** are listed in the OpenGL extension registry on the OpenGL Web site.

- There are three major classifications of extensions: vendor (written by 1 vendor like AMD or NV), EXT (wirtten by 2 or more vendors), and ARB (official part of OpenGL).

## Chapter 4 - Maths for 3D Graphics

- The **dot product** between vectors is useful to get the angle between the two vectors. It is used extensively during **lighting** calculations between the surface normal and the vector pointing toward a light source in diffuse light calculations.

- The **cross product** is used in many applications from finding the surface normals of triangles to constructing transformation matrices.

- By default, OpenGL uses a **column-major** or a **column-primary** layout from matrices: the first four elements represent the first column of the matrix, the next four elements represent the second column, and so on.

- In memory, **GLfloat matrix[16];** has a column-major matrix ordering.

- In memory, **GLfloat matrix[4][4];** is laid out in a row-major order.

- In 4x4 matrix, notice that the last row of the matrix is all 0s with the exception of the very last element, which is 1. The upper-left 3 × 3 sub-matrix of the matrix shown in Figure 4.5 represents a **rotation or orientation**. The last column of the matrix represents a **translation or position**.

- Any position in space and any desired orientation can be uniquely defined by a 4 × 4 matrix.

- If you multiply a vertex expressed in the identity coordinate system (written as a column matrix or vector) by this matrix, the result is a new vertex that has been transformed to the new coordinate system.

- Matrix multiplication is **associative**: A * (B * v) = (A * B) * v. It is possible to stack a whole bunch of transforms together by multiplying the matrices that represent those transforms and using the resulting matrix as a single term in the final product.

- **Projection**: is the process of squishing 3D data down into 2D data. The projection is the type of transformation (orthographic or perspective) that occurs during vertex processing.

1. Most of your vertex data will typically begin life in **object space**, which is also commonly known as **model space**. The positions of vertices are interpreted relative to a local origin.

2. The **world space** is where coordinates are stored relative to a fixed, global origin. Once in world space, all objects exist in a common frame. Often, this is the space in which lighting and physics calculations are performed.

3. **View or eye or camera coordinates** are relative to the position of the observer (hence the terms “camera” and “eye”) regardless of any transformations that may occur; eye coordinates represent a virtual fixed coordinate system that is used as a common frame of reference.

- Positive x and y are pointed right and up, respectively, from the viewer’s perspective. Positive z travels away from the origin toward the user, and negative z values travel farther away from the viewpoint into the screen. The screen lies at the z coordinate 0.

4. **Clip space** is always a four-dimensional homogenous coordinate. Upon exiting clip space, all four of the vertex’s components are divided through by the w component. The result of the division is considered to be in **normalized device coordinate space (NDC space)**.

- This allows for effects such as perspective foreshortening and projection.

- When your vertex shader writes to gl_Position, this coordinate is considered to be in clip space.

- A summary of the coordinate spaces used in 3D graphics:

	- **Model space** or **object space**: positions relative to a local origin.
	- **World space**: positions relative to a global origin.
	- **View or Camera or Eye space**: positions relative to the viewer.
	- **Clip space**: positions of vertices after projection into a nonlinear homogeneous coordinate.
	- **Normalized device coordinate (NDC) space**: Vertex coordinates are said to be in NDC after their clip space coordinates have been divided by their own w component.
	- **Window space**: Positions of vertices in pixels, relative to the origin of the window.

- Coordinate transform are: **translatio**, **rotation**, and **scaling**.

- A **translation matrix** moves the vertices along one or more of the three axes. The translation amount tx, ty, tz along each of the axes, are the first three elements of the last column.

- **Position vectors** are almost always encoded using four components with w (the last) being 1.0.

- **Direction vectors** are encoded either simply using three components, or as four components with w being zero. Thus, multiplying a four-component direction vector by a translation matrix doesn’t change it at all.

- The form of a **rotation matrix** depends on the axis about which we wish to rotate. It is possible to multiply these three matrices together to produce a composite transform matrix in order to rotate by a given amount around each of the three axes.

- **Euler angles** are a set of three angles3 that represent orientation in space. Each angle represents a rotation around one of three orthogonal vectors that define our frame. However, Euler angles also come with a serious pitfall, the **gimbal lock**: when a rotation by one angle reorients one of the axes to be aligned with another of the axes, removing a degree of freedom from the system.

- Multiplying by a sequence of matrices can apply a **sequence of transformations**. Order is important: you should always multiply a matrix by a vector and read the sequence of transformations in **reverse order**.

- A **quaternion** is a four-dimensional quantity that is similar in some ways to a complex number: it has a real part and **three imaginary parts (i, j, and k)**.

- q = (x + yi + zj + wk)

- i^2 = j^2 = k^2 = ikj = −1

- The product of any two of i, j, and k gives whichever one was not part of that product. Example: i = jk.

- A rotation around an axis can be represented by a quaternion where the real part is the angle and axis is the imaginary or the vector part. A sequence of rotations can be represented by a series of quaternions multiplied together.

- If you multiply matrices representing rotations around the Cartesian axes and you gimbal lock can occur, using quaternions, gimbal lock cannot occur.

- The **model-view transform** transforms from model space (relative to the object's origin) to view space (relative to the viewer). This process establishes the **vantage point / viewpoint** of the scene.

- By **default, the point of observation in a perspective projection** is at the origin (0,0,0) looking down the negative z axis (into the monitor or screen). In a perspective projection, objects drawn with positive z values are behind the observer.

- In an orthographic projection, however, the viewer is assumed to be infinitely far away on the positive z axis and can see everything within the viewing volume.

- The **model-world transform** transforms the positions in model space to positions in world space.

- Determining the viewing transformation is like placing and pointing a camera at the scene.

- The transform that moves coordinates from world space to view space is sometimes called the **world–view transform**.

- Concatenating the model–world and world–view transform matrices by multiplying them together yields the model–view matrix.

- Using a single composite transform to move the model into view space is more efficient than moving it first into world space and then into view space.

- The second advantage has more to do with the numerical accuracy of single-precision floating-point numbers: in world space, precision depends on how far the vertices are from the world origin; in view space, then precision is dependent on how far vertices are from the viewer.

- The **lookat matrix** represents a rotation that will point a camera in the correct direction and a translation that will move the origin to the center of the camera.

- The **forward vector** is a vector that represents the direction of view from the camera to the point of interest.

- The **sideways vector** is orthogonal (compute the cross product) to the forward vector and the **up vector** (indicating the upward direction from the camera).

- The **up vector** is obtained taking the cross product of the forward vector and our sideways vector to produce a third that is orthogonal to both and that represents up with respect to the camera.

- These three vectors are of unit length and are all orthogonal to one another. Given these three vectors, we can construct a rotation matrix.

- To transform objects into the camera’s frame, not only do we need to orient everything correctly, but we also need to move the origin to the position of the camera. We do this by simply translating the resulting vectors by the negative of the camera’s position.

- So to build the lookat matrix we need the eye position, the center position and the up direction.

- The **projection transformation** is applied to your vertices after the model–view transformation: defines the viewing volume and establishes clipping planes.

- The **clipping planes** are plane equations in 3D space that OpenGL uses to determine whether geometry can be seen by the viewer.

- More specifically, the projection transformation specifies how a finished scene (after all the modeling is done) is projected to the final image on the screen.

- In an **orthographic, or parallel, projection**, lines and polygons are mapped directly to the 2D screen using parallel lines. It means no matter how far away something is, it is still drawn the same size, just flattened against the screen.

- Orthographic projections are used most often for 2D drawing purposes where you want an exact correspondence between pixels and drawing units. You also can use an orthographic projection for 3D renderings when the depth of the rendering has a very small depth in comparison to the distance from the viewpoint.

- A **perspective projection** shows scenes more as they appear in real life. **Foreshortening** makes distant objects appear smaller than nearby objects of the same size. Lines in 3D space that might be parallel do not always appear parallel to the viewer; they appear to converge at some distant point.

- Perspective projections are used for rendering scenes that contain wide-open spaces or objects that need to have foreshortening applied. For the most part, perspective projections are typical for 3D graphics.

- Once your vertices are in view space, we need to get them into clip space, which we do by applying our projection matrix.

- A commonly used **perspective matrix** is a **frustum matrix**, that produces a perspective projection such that clip space takes the shape of a rectangular frustum, which is a **truncated rectangular pyramid**.

- Its parameters are the distance to the near and far planes and the world-space coordinate of the left, right, top, and bottom clipping planes.

- Another common method for constructing a perspective matrix is to directly specify a field of view as an angle (in degrees, perhaps), an aspect ratio (generally derived by dividing the window’s width by its height), and the view-space positions of the near and far planes. It produces only symmetric frustra.

- An **orthographic projection matrix** is simply a scaling matrix that linearly maps view-space coordinates into clip-space coordinates.

- The parameters to construct the orthographic projection matrix are the left, right, top, and bottom coordinates in view space of the bounds of the scene, and the position of the near and far planes.

- **Linear interpolation** can be applied to scalar values; two-dimensional values such as points on a graph; three-dimensional values such as coordinates in 3D space, colors, and so on; or even higher-dimension quantities such as matrices. Usually it is applied to angles, positions, and other coordinates.

- GLSL includes a built-in function specifically for this purpose: **vec4 mix(vec4 A, vec4 B, float t);**

- A **curve** can be represented by three or more **control points**. For most curves, there are more than three control points, two of which form the **endpoints**;

- The **quadratic Bézier curve** (3 control points) or the **cubic Bézier curve** (4 control points) can be implemented very easily in GLSL using the mix function.

- The cubic Bézier curve includes the quadratic one so that it can be reused.

- Now that we see this pattern, we can take it further and produce even higher-order curves. For example, a **quintic Bézier curve** (one with five control points).

- In practice, curves with more than four control points are not commonly used. Rather, we use **splines**.

- A spline is effectively a long curve made up of several smaller curves (such as Béziers) that locally define their shape. One or more of the interior control points are either shared or linked in some way between adjacent segments. Any number of curves can be joined together in this way, allowing arbitrarily long paths to be formed.

- A **cubic Bézier spline** is constructed from a sequence of cubic Bézier curves. This is also known as a **cubic B-spline**.

- Bézier curve segments are both C^1 (continuous first derivative) and C^2 (continuous second derivative) continuous.

- To ensure that we maintain continuity over the welds of a spline, we need to ensure that each segment starts off where the previous ended in terms of position, direction of movement, and rate of change.

- A cubic B-spline represented this way (as a set of weld positions and velocities) is known as a **cubic Hermite spline**, or sometimes simply a cspline. 