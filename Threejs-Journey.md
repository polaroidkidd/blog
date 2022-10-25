# Notes on my Threejs-Journey
The contents of this article/summary are based off of the excelent course [Threejs-Journey](https://threejs-journey.com/) by Bruno Simon.


**Author: Daniel Einars**

**Date Published: 25.10.2022**

**Date Edited: 25.10.2022**


##  1. Basics
This part summarises my notes on all the basics regarding the use of threejs such as creating a scene, transforming objects, animations etc.

###  1.1. Basic Scene
Before we do anything, we need to create a scene. This is essentially the container for everything else and you create it like this
```javascript
const scene = new THREE.Scene()
```
###  1.2. Objects

Objects are things you render to the scene. They can be anything from simple shapes (pyramids, cubes, spheres etc.) to imported models, particles, lights etc.
In order to create a simple box we need the geometry (shape) and the mesh (what the surface looks like)
```javascript
const geometry = new THREE.BoxGeometry(1, 1, 1) // the 1, 1, 1 are the width, height and depth. 
const material = new THREE.MeshBasicMaterial({ color: 0xff0000 })
```
We then combine the geometry and material to create a mesh, which we then add to the scene
```javascript
const mesh = new THREE.Mesh(geometry, material)
scene.add(mesh)
```

### 1.3. Camera
In order to actually see anything, we need a camera to view things. There are a bunch of different cameras for different purposes, but they all inherit from the base camera class (don't use this one, but one of the specifics). From the docs: 
  * ArrayCamera:  ArrayCamera can be used in order to efficiently render a scene with a predefined set of cameras. This is an important performance aspect for rendering VR scenes. An instance of ArrayCamera always has an array of sub cameras. It's mandatory to define for each sub camera the viewport property which determines the part of the viewport that is rendered with this camera.
  * CubeCamera: Creates 6 cameras that render to a WebGLCubeRenderTarget.
  * OrthographicCamera: In this projection mode, an object's size in the rendered image stays constant regardless of its distance from the camera.  This can be useful for rendering 2D scenes and UI elements, amongst other things.
  * PerspectiveCamera: This projection mode is designed to mimic the way the human eye sees. It is the most common projection mode used for rendering a 3D scene.
  * StereoCamera: Dual PerspectiveCameras used for effects such as 3D Anaglyph or Parallax Barrier.

For now, we'll be using the perspective camera like this
```javascript
const camera = new THREE.PerspectiveCamera(75, sizes.width / sizes.height, 0.1, 100)
```
The arguments are
    1. Field of View
    2. Aspect Ratio
    3. Near (how close you can move with the camera before objects start disappearing)
    4. Far (opposite of near)

Be mindful to use sensible defaults for near and far because if you choose values which are too large you'll run into performance issues and if you choose values too small you won't see rendered objects. It's ok to start between 0.1 and 100, and if you notice you need more "space", simply increase the far argument

Also, don't forget to add the camera to the scene and  to move the camera back from the object, otherwise you'll render the object and the camera into the same coordinates and you won't be able to see anything
```javascript
camera.position.z = 3
scene.add(camera)
```

### 1.4. Transforming Objects

We have four properties which we can use to transform objects. Those are

  * position (x, y, z coordinates)
  * scale
  * rotation
  * quaternion (also rotation but more math-y)

We can change an object's position in two ways. You can either set the `x`, `y` and `z` coordinate separately or you can call the `mesh.position.set(x, y, z)` function.

Since the position property is a Vector3 class, it also has other functions such as `position.length()` which will return the vector's length. You can use it to calculate the distance to the camera by using `mesh.position.distanceTo(camera.position)` and you can also normalize the vector by calling the `mesh.position.normalize()` function.

### 1.5. Axes Helper

Sometimes it's useful to know which axis is whereas you might have rotated the camera as well as the object. In order to have this appear, use the following code snippit
```javascript
const axesHelper = new THREE.AxesHelper(2) // takes size as an argument
scene.add(axesHelper)
```

### 1.6. Rotating and Scaling Objects

Scaling objects is pretty straightforward. Do this by setting the scale value of the appropriate axis like this
```javascript
mesh.scale.x = 2
mesh.scale.y = 0.25
mesh.scale.z = 0.5 
```

Rotating is only a tad trickier. If you want to rotate an object, imagine you're putting a rod through the center of one of the axis and then rotate it by degrees or radians. Bruno gives three good examples of this

>  * If you spin on the y axis, you can picture it like a carousel.
>  * If you spin on the x axis, you can imagine that you are rotating the wheels of a car you'd be in.
>  * And if you rotate on the z axis, you can imagine that you are rotating the propellers in front of an aircraft you'd be in.

Rotations are applied as follows
```javascript
mesh.rotation.x = Math.PI * 0.25
mesh.rotation.y = Math.PI * 0.25
```
Depending on the order in which rotations are applied, you might end up with something called "gimbal lock". Wikipedia gives a decent [explanation](https://en.wikipedia.org/wiki/Gimbal_lock) of it
> Gimbal lock is the loss of one degree of freedom in a three-dimensional, three-gimbal mechanism that occurs when the axes of two of the three gimbals are driven into a parallel configuration, "locking" the system into rotation in a degenerate two-dimensional space. 

In order to avoid this you simply have to change the order in which rotations are applied like this
```javascript
object.rotation.reorder('yxz')
```

### 1.7. Grouping Objects

Sometimes you'll have spent a large amount of time developing a scene, only to figure out that a part of it is too small, or needs to be repositioned. Because you don't want to move every item individually, you can add them to a group and apply all transformations as a group. The code for this is fairly simple:

```javascript
const group = new THREE.Group() // create new group
group.scale.y = 2 // no change since nothing has been added to the group yet
group.rotation.y = 0.2
scene.add(group) // don't forget to add the group to the scene

const cube1 = new THREE.Mesh(
    new THREE.BoxGeometry(1, 1, 1),
    new THREE.MeshBasicMaterial({ color: 0xff0000 })
)
cube1.position.x = - 1.5
group.add(cube1) // add item to group

const cube2 = new THREE.Mesh(
    new THREE.BoxGeometry(1, 1, 1),
    new THREE.MeshBasicMaterial({ color: 0xff0000 })
)
cube2.position.x = 0
group.add(cube2) // add another item to the group

const cube3 = new THREE.Mesh(
    new THREE.BoxGeometry(1, 1, 1),
    new THREE.MeshBasicMaterial({ color: 0xff0000 })
)
cube3.position.x = 1.5 // transformation is applied to all items in the group
group.add(cube3)
```

### 1.8. Animations
As with any animation in javascript we need to make use of `requestAnimationFrame`. This function accepts a function, which is called when the next frame is available. Any code you need to run on every frame should be placed inside this. Because some mashines are faster than others and you don't want to waste resources, you should aim for animation at 60fps. Some libraries provied functions for that (such as gsap), but threejs also provides a solution. Animating simple things is very similar to animating anything else using javascript. 
```javascript
// get the threejs clock
const clock = new THREE.Clock()

const tick = () =>
{
    // get the elapsed time
    const elapsedTime = clock.getElapsedTime()

    // Update objects with the elapsed time
    camera.position.x = Math.cos(elapsedTime)
    camera.position.y = Math.sin(elapsedTime)
    camera.lookAt(mesh.position)

    // ...
}

// call animation function
tick()
```

Note that you can also use the js native way and get the current time using `Date.now()`, calculate the delta within the `tick()` function and then apply delta to the rotation. **Do not do this when using the `THREE.Clock()` function as it breaks things**

### 1.9. Controls
This chapter largly deals with moving the camera around. There are a number of different controls provided by threejs (look at the [documentation](https://threejs.org/docs/index.html?q=controls#examples/en/controls/ArcballControls) for more information). I'm largly copy&pasting these descriptions


  * [ArcballControls](https://threejs.org/docs/index.html?q=controls#examples/en/controls/ArcballControls): Arcball controls allow the camera to be controlled by a virtual trackball with full touch support and advanced navigation functionality.
  * [DragControls](https://threejs.org/docs/index.html?q=controls#examples/en/controls/DragControls):  This class can be used to provide a drag'n'drop interaction.
  * [FlyControls](https://threejs.org/docs/index.html?q=controls#examples/en/controls/FlyControls): FlyControls enables a navigation similar to fly modes in DCC tools like Blender. You can arbitrarily transform the camera in 3D space without any limitations (e.g. focus on a specific target). These are controls which are used when flying a spaceship.
  * [FirstPersonControls](https://threejs.org/docs/index.html?q=controls#examples/en/controls/FirstPersonControls): Like FlyControls, but with a fixed "up" axis. The FlyControls can do a barrel roll, the FirstPersonControls cannot
  * [OrbitControls](https://threejs.org/docs/index.html?q=controls#examples/en/controls/OrbitControls):  Orbit controls allow the camera to orbit around a target.
  * [PointerLockControls](https://threejs.org/docs/index.html?q=controls#examples/en/controls/PointerLockControls): The implementation of this class is based on the Pointer Lock API. PointerLockControls is a perfect choice for first person 3D games as it centers and hides the mouse cursor.
  * [TrackballControls](https://threejs.org/docs/index.html?q=controls#examples/en/controls/TrackballControls): https://threejs.org/docs/index.html?q=controls#examples/en/controls/TrackballControls
  * [TransformControls](https://threejs.org/docs/index.html?q=controls#examples/en/controls/TransformControls): This class can be used to transform objects in 3D space by adapting a similar interaction model of DCC tools like Blender. Unlike other controls, it is not intended to transform the scene's camera.***TransformControls expects that its attached 3D object is part of the scene graph.***
  * [OrbitControls](https://threejs.org/docs/#examples/en/controls/OrbitControls): These allow a user to orbit around. This class comes with a bunch of extra configuration options which makes the use more natural.

### 1.10. Orbit Controls
For some weird reason you have to import the controls from the examples directory like this

```javascript
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js'
const controls = new OrbitControls(camera, canvas) // attach it to the canvas and the camera
```


*... to be continued ...*