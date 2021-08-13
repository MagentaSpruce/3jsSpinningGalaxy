# 3jsSpinningGalaxy
This Threejs project is part of an earlier project that is now being added to in order to create realistic 3D animations.

A general overview of the pertinent code is below. Since this project is starting from the end of a previous project, the previous projects code will be provided in full at the beginning of this readMe. However the walkthrough will only start at replacing the pointsMaterial to the shaderMaterial. To see how the galaxy was originally created please see the readMe inside of the galaxyGenerator project.

To start, here is the final code from the galaxyGenerator project:
```JavaScript
import './style.css'
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js'
import * as dat from 'dat.gui'

/**
 * Base
 */
// Debug
const gui = new dat.GUI()

// Canvas
const canvas = document.querySelector('canvas.webgl')

// Scene
const scene = new THREE.Scene()

/**
 * Galaxy
 */
const parameters = {}
parameters.count = 200000
parameters.size = 0.005
parameters.radius = 5
parameters.branches = 3
parameters.spin = 1
parameters.randomness = 0.5
parameters.randomnessPower = 3
parameters.insideColor = '#ff6030'
parameters.outsideColor = '#1b3984'

let geometry = null
let material = null
let points = null

const generateGalaxy = () =>
{
    if(points !== null)
    {
        geometry.dispose()
        material.dispose()
        scene.remove(points)
    }

    /**
     * Geometry
     */
    geometry = new THREE.BufferGeometry()

    const positions = new Float32Array(parameters.count * 3)
    const colors = new Float32Array(parameters.count * 3)

    const insideColor = new THREE.Color(parameters.insideColor)
    const outsideColor = new THREE.Color(parameters.outsideColor)

    for(let i = 0; i < parameters.count; i++)
    {
        const i3 = i * 3

        // Position
        const radius = Math.random() * parameters.radius

        const branchAngle = (i % parameters.branches) / parameters.branches * Math.PI * 2

        const randomX = Math.pow(Math.random(), parameters.randomnessPower) * (Math.random() < 0.5 ? 1 : - 1) * parameters.randomness * radius
        const randomY = Math.pow(Math.random(), parameters.randomnessPower) * (Math.random() < 0.5 ? 1 : - 1) * parameters.randomness * radius
        const randomZ = Math.pow(Math.random(), parameters.randomnessPower) * (Math.random() < 0.5 ? 1 : - 1) * parameters.randomness * radius

        positions[i3    ] = Math.cos(branchAngle) * radius + randomX
        positions[i3 + 1] = randomY
        positions[i3 + 2] = Math.sin(branchAngle) * radius + randomZ

        // Color
        const mixedColor = insideColor.clone()
        mixedColor.lerp(outsideColor, radius / parameters.radius)

        colors[i3    ] = mixedColor.r
        colors[i3 + 1] = mixedColor.g
        colors[i3 + 2] = mixedColor.b
    }

    geometry.setAttribute('position', new THREE.BufferAttribute(positions, 3))
    geometry.setAttribute('color', new THREE.BufferAttribute(colors, 3))

    /**
     * Material
     */
    material = new THREE.PointsMaterial({
        size: parameters.size,
        sizeAttenuation: true,
        depthWrite: false,
        blending: THREE.AdditiveBlending,
        vertexColors: true
    })

    /**
     * Points
     */
    points = new THREE.Points(geometry, material)
    scene.add(points)
}

generateGalaxy()

gui.add(parameters, 'count').min(100).max(1000000).step(100).onFinishChange(generateGalaxy)
gui.add(parameters, 'radius').min(0.01).max(20).step(0.01).onFinishChange(generateGalaxy)
gui.add(parameters, 'branches').min(2).max(20).step(1).onFinishChange(generateGalaxy)
gui.add(parameters, 'randomness').min(0).max(2).step(0.001).onFinishChange(generateGalaxy)
gui.add(parameters, 'randomnessPower').min(1).max(10).step(0.001).onFinishChange(generateGalaxy)
gui.addColor(parameters, 'insideColor').onFinishChange(generateGalaxy)
gui.addColor(parameters, 'outsideColor').onFinishChange(generateGalaxy)

/**
 * Sizes
 */
const sizes = {
    width: window.innerWidth,
    height: window.innerHeight
}

window.addEventListener('resize', () =>
{
    // Update sizes
    sizes.width = window.innerWidth
    sizes.height = window.innerHeight

    // Update camera
    camera.aspect = sizes.width / sizes.height
    camera.updateProjectionMatrix()

    // Update renderer
    renderer.setSize(sizes.width, sizes.height)
    renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2))
})

/**
 * Camera
 */
// Base camera
const camera = new THREE.PerspectiveCamera(75, sizes.width / sizes.height, 0.1, 100)
camera.position.x = 3
camera.position.y = 3
camera.position.z = 3
scene.add(camera)

// Controls
const controls = new OrbitControls(camera, canvas)
controls.enableDamping = true

/**
 * Renderer
 */
const renderer = new THREE.WebGLRenderer({
    canvas: canvas
})
renderer.setSize(sizes.width, sizes.height)
renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2))

/**
 * Animate
 */
const clock = new THREE.Clock()

const tick = () =>
{
    const elapsedTime = clock.getElapsedTime()

    // Update controls
    controls.update()

    // Render
    renderer.render(scene, camera)

    // Call tick again on the next frame
    window.requestAnimationFrame(tick)
}

tick()
```

This project will use shaders to animate each particle by using a vertex shader which will be much better for performances and frame rates. Each vertex will be made to rotate with those at the center spinning faster to create a life-linke galaxy animation. To begin this process I replaced the pointsMaterial with a shaderMaterial.
```JavaScript
  material = new THREE.ShaderMaterial({
```

Since now shader is being used properties like size and sizeAttentuation will no longer work because they were for points material so they will need to be added in manually later. Currently the screen is black because now there is no size set for the material particles so the GPU sees nothing bc no size. To fix this I added in a vertexShader and set the gl_pointSize.
```JavaScript
  material = new THREE.ShaderMaterial({
    depthWrite: false,
    blending: THREE.AdditiveBlending,
    vertexColors: true,
    vertexShader: `void main()
    {
        vec4 modelPosition = modelMatrix * vec4(position, 1.0);
        vec4 viewPosition = viewMatrix * modelPosition;
        vec4 projectedPosition = projectionMatrix * viewPosition;
        gl_Position = projectedPosition;

        gl_PointSize = 2.0;
    
    }`,
  });
  ```

Now the screen is no longer black but shows red particles since the shader material comes with some preset values, like red color. Now that the vertex shader is done the fragmentShader is added.
```JavaScript
    fragmentShader: `
    void main()
    {
        gl_FragColor = vec4(1.0, 1.0, 1.0, 1.0);
    }`,
```

Now the particles are white. Now that the shaders have been set up and shwn to work inside of the script.js file they are each moved into seperate shader files. To do this a new folder was created in the src folder called shaders. Inside the shaders folder another folder named galaxy was created. Inside of the galaxy folder two files are created, fragment.glsl and vertex.glsl. Then the shaders were c&p over to the new files (not shown). Next those two new files were imported into script.js.
```JavaScript
import galaxyVertexShader from './shaders/galaxy/vertex.glsl';
import galaxyFragmentShader from './shaders/galaxy/fragment.glsl';
```

Next the two imports are used inside of the shaderMaterial.
```JavaScript
    vertexShader: galaxyVertexShader,
    fragmentShader: galaxyFragmentShader,
```

A white particled galaxy should be displayed back on screen. Since the pointsMaterial is no longer being used (shaderMaterial is) then everything that the pointsMaterial was doing which is required for the animation had to be redone. To start this process the base size was handled so as to be controllable outside of the vertex.glsl file via Dat.GUI. To make this possible a unfirom was created.
```JavaScript
    uniforms: {
      uSize: { value: 8 },
    },
```

Now the uniform is being sent but not used. To rectify that the uniform was retrieved inside of the vertex shader and used there.
```JavaScript
uniform float uSize;

        gl_PointSize = uSize;
```

In a real galaxy the stars are different sizes. To have a different size for each vertex a random value is sent from the attributes. **Always add randomness to particle sizes for realism** To achieve this an aScale attributes was added to the geometry containing one random value for each vertex.
```JavaScript
  const scales = new Float32Array(parameters.count * 1);
  ...
    scales[i] = Math.random();
  ...
  geometry.setAttribute("aScale", new THREE.BufferAttribute(scales, 1));
```

Next the attribute was retrieved and used inside of vertex.glsl.
```JavaScript
attribute float aScale;

void main()
    {
...
        gl_PointSize = uSize * aScale;
```

Next the pixel ratio is adjusted to ensure all users see the same rendering quality. 
```JavaScript
    uniforms: {
      uSize: { value: 8 * renderer.getPixelRatio() },
    },
```

This generates an error because the renderer does not exist yet. So the generateGalaxy() function call was moved to right above the tick() (not shown). Next the particles size attentuation is adjusted by taking code from the Three.js dependency and use it inside vertex.glsl.
```JavaScript
 gl_PointSize *= ( 1.0 / - viewPosition.z );
 ```
 
 Next a pattern was drawn inside of the square particles using the uv.
 ```JavaScript
        //Disc
        float strength = distance(gl_PointCoord, vec2(0.5));
        strength = step(0.5, strength);
        strength = 1.0 - strength;
        gl_FragColor = vec4(vec3(strength), 1.0);
```

Next a diffuse point pattern was created.
```JavaScript
        float strength = distance(gl_PointCoord, vec2(0.5));
        strength *= 2.0;
        strength = 1.0 - strength;
        gl_FragColor = vec4(vec3(strength), 1.0);
```

To go even further a light point pattern was also added.
```JavaScript
        float strength = distance(gl_PointCoord, vec2(0.5));
        strength = 1.0 - strength;
        strength = pow(strength, 10.0);
        gl_FragColor = vec4(vec3(strength), 1.0);
```

Next the particle sizes were increased.
```JavaScript
    uniforms: {
      uSize: { value: 30 * renderer.getPixelRatio() },
```

Now that the particles, size and pixel ration have been taken care of colors can be added usging a varying from the vertex to the fragment.
```JavaScript
varying vec3 vColor;
    vColor = color;
```

The varying declaration was added to the fragment file and used.
```JavaScript
varying vec3 vColor;

 void main()
    {
...
        vec3 color = mix(vec3(0.0), vColor, strength);
        gl_FragColor = vec4(color, 1.0);
```



At this point the galaxy is back to how it was upon the initial render before the shader was used. Now the next steps are to animate the galaxy. To do this a new uniform was created.
```JavaScript
      uTime: { value: 0 },
```

Next that value was made to be updated inside the tick() function.
```JavaScript
  //Update Materials
  material.uniforms.uTime.value = elapsedTime;
```

Next the uTime was used inside the vertex file.
```JavaScript
uniform float uTime;
...

void main()
    {
        vec4 modelPosition = modelMatrix * vec4(position, 1.0);
        modelPosition.y += uTime;
        ...
```

The vertices will only be animated on the x and z to prevent the galaxy from moving up and down out of the screen. 
```JavaScript

        //Spin
        float angle = atan(modelPosition.x, modelPosition.z);
        float distanceToCenter = length(modelPosition.xz);
        float angleOfOffset = (1.0 / distanceToCenter) * uTime * 0.2;
        angle += angleOfOffset;
```

Now that the variable inside the vertex has been updated the modelPosition can be updated using the new angles.
```JavaScript
        modelPosition.x = cos(angle) * distanceToCenter;
        modelPosition.z = sin(angle) * distanceToCenter;
```

Now the issue is preventing the galaxy from spinning in on itself creating a ribbon. To do this, instead of applying the randomness before the spin on the base posotion, the randomness is removed from the base position and added to an attribute so it can be added after the spin.
```JavaScript
  const randomness = new Float32Array(parameters.count *3);
  
      randomness[i3 + 0] = randomX;
    randomness[i3 + 1] = randomY;
    randomness[i3 + 2] = randomZ;
    
      geometry.setAttribute(
    "aRandomness",
    new THREE.BufferAttribute(randomness, 3)
  );
```

Now the randomness can be retrieved inside vertex.glsl.
```JavaScript
attribute vec3 aRandomness;

        //Randomness
        modelPosition.x += aRandomness.x;
        modelPosition.y += aRandomness.y;
        modelPosition.z += aRandomness.z;
```

***End Walkthrough
