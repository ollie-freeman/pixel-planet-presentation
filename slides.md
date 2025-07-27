---
theme: default
transition: slide
layout: intro
---

# Pixel Planets

How to create art for the talentless

<script setup>
const shaderCode = `
@group(0) @binding(0)
var<uniform> u_resolution: vec2<f32>;

@group(0) @binding(1)
var<uniform> u_time: f32;

fn rand(p: vec2<f32>) -> f32 {
  return fract(sin(dot(p, vec2<f32>(12.9898, 78.233))) * 43758.5453);
}

fn noise(p: vec2<f32>) -> f32 {
  let i = floor(p);
  let f = fract(p);
  let a = rand(i);
  let b = rand(i + vec2<f32>(1, 0));
  let c = rand(i + vec2<f32>(0, 1));
  let d = rand(i + vec2<f32>(1, 1));
  let u = f * f * (3.0 - 2.0 * f);
  return mix(mix(a, b, u.x), mix(c, d, u.x), u.y);
}

fn fbm(p: vec2<f32>, size: f32, octaves: u32) -> f32 {
  var value = 0.0;
  var amp = 0.5;
  var freq = size;
  var coord = p;

  for (var i = 0u; i < octaves; i++) {
    value += noise(coord * freq) * amp;
    coord *= 2.0;
    amp *= 0.5;
  }

  return value;
}

fn spherify(uv: vec2<f32>) -> vec2<f32> {
  let centered = uv * 2.0 - 1.0;
  let z = sqrt(1.0 - dot(centered, centered));
  let sphere = centered / (z + 1.0);
  return sphere * 0.5 + 0.5;
}

@fragment
fn fs_main(@builtin(position) coord: vec4<f32>) -> @location(0) vec4<f32> {
    let pixels = 50.0;
    var uv = floor((coord.xy / u_resolution) * pixels) / pixels;

    let d = distance(uv, vec2<f32>(0.5));
    let mask = step(d, 0.5);

    uv = spherify(uv);

    let shift = vec2<f32>(u_time * 0.01, 0.0);
    let n = fbm(uv - shift, 6.0, 5u);

    let palette = array<vec3<f32>, 6>(
        vec3<f32>(0.0,   0.106, 0.316),
        vec3<f32>(0.0,   0.149, 0.393),
        vec3<f32>(0.0,   0.175, 0.443),
        vec3<f32>(0.18,  0.393, 0.15),
        vec3<f32>(0.255, 0.45,  0.15),
        vec3<f32>(0.293, 0.471, 0.168)
    );
    let t = n * 6.0;
    let i0 = u32(floor(t));
    let i1 = min(i0 + 1u, 5u);
    let f  = fract(t);

    let w = smoothstep(0.45, 0.55, f);
    let base_color = mix(palette[i0], palette[i1], w);

    let cn = fbm(uv * 3.0 - shift * 6.0, 3.0, 3u);
    let a  = smoothstep(0.4, 0.7, cn) * 0.8;

    let col = mix(base_color, vec3<f32>(1.0), a);

    return vec4<f32>(col, mask);
}
`
</script>

<WebGpuCanvas
  :fragmentShader="shaderCode"
  :useTime="true"
  :fullscreen="false"
/>

---
layout: cover
---

# Shaders

A program that runs on the **GPU**, written in a language like **WGSL** or **GLSL**.

* Tells the GPU what **color** each pixel should be
* Runs in **parallel** for each pixel or vertex
* Allows beautiful and efficient visual effects
* Can be used for computation

---
layout: cover
---

# Why WebGPU?

* Modern alternative to WebGL
* Cross-platform
* First class citizen in some game engines
* Simple API but fast and capable
* Supports compute shaders

---
layout: cover
---

# Shader Types

* **Vertex Shader**: defines geometry (shapes)
* **Fragment Shader**: colors pixels based on coordinates and math
* **Compute Shader**: general-purpose computations

---
layout: cover
---

# Basic Vertex Shader

```wgsl
@vertex
fn vs_main(@builtin(vertex_index) index: u32) -> @builtin(position) vec4<f32> {
  ...
}
```

---
layout: cover
---

# Basic Vertex Shader

In WGSL, vertex shaders use Normalized Device Coordinates.

- The screen is a square from **-1.0** to **1.0** on both X and Y axes
- **(-1, -1)** is the **bottom-left**
- **(1, 1)** is the **top-right**
- The Z axis is used for depth (0 = near, 1 = far), but we’ll use **0.0** for 2D shaders
- Values outside this range are clipped

---
layout: cover
---

# Basic Vertex Shader

````md magic-move
```wgsl
@vertex
fn vs_main(@builtin(vertex_index) index: u32) -> @builtin(position) vec4<f32> {
  ...
}
```


```wgsl
@vertex
fn vs_main(@builtin(vertex_index) index: u32) -> @builtin(position) vec4<f32> {
  var positions = array<vec2<f32>, 6>(
    vec2<f32>(-1.0, -1.0), vec2<f32>( 1.0, -1.0), vec2<f32>(-1.0,  1.0),
    vec2<f32>(-1.0,  1.0), vec2<f32>( 1.0, -1.0), vec2<f32>( 1.0,  1.0)
  );
  return vec4<f32>(positions[index], 0.0, 1.0);
}
```
````

---
layout: cover
---

<script setup>
const shaderCode = `
@group(0) @binding(0)
var<uniform> u_resolution: vec2<f32>;

@fragment
fn fs_main(
    @builtin(position) fragCoord: vec4<f32>
) -> @location(0) vec4<f32> {
    let uv = fragCoord.xy / u_resolution;
    let sum = uv.x + uv.y;
    let edgeWidth = 0.002;
    if (abs(sum - 1.0) < edgeWidth) {
        return vec4<f32>(1.0, 1.0, 1.0, 1.0);
    } else if (sum < 1.0) {
        return vec4<f32>(1.0, 0.0, 0.0, 1.0);
    } else {
        return vec4<f32>(0.0, 1.0, 0.0, 1.0);
    }
}
`
</script>

<WebGpuCanvas
  :fragmentShader="shaderCode"
/>


---
layout: cover
---

# Basic Fragment Shader

```wgsl
@fragment
fn fs_main(@builtin(position) coord: vec4<f32>) -> @location(0) vec4<f32> {
  ...
}
```

---
layout: cover
---

# Basic Fragment Shader

`@builtin(position) coord: vec4<f32>` is the current pixel in Window Coordinates.

If the resolution is 800px × 600px:
- **(0, 0)** is the **top-left**
- **(800, 600)** is the **bottom-right**
- The fragment shader is called **480,000** times
- The output is a normalized RGBA colour (`vec4<f32>` between **0-1**)

---
layout: cover
---

# Basic Fragment Shader

````md magic-move
```wgsl
@fragment
fn fs_main(@builtin(position) coord: vec4<f32>) -> @location(0) vec4<f32> {
  ...
}
```

```wgsl
@fragment
fn fs_main(@builtin(position) coord: vec4<f32>) -> @location(0) vec4<f32> {
  let uv = coord.xy / vec2<f32>(800.0, 600.0);
  return vec4<f32>(uv, 0.0, 1.0);
}
```
````

---
layout: cover
---

<script setup>
const shaderCode = `
@group(0) @binding(0)
var<uniform> u_resolution: vec2<f32>;

@fragment
fn fs_main(@builtin(position) coord: vec4<f32>) -> @location(0) vec4<f32> {
  let uv = coord.xy / u_resolution;
  return vec4<f32>(uv, 0.0, 1.0);
}
`
</script>

<WebGpuCanvas
  :fragmentShader="shaderCode"
/>

---
layout: cover
---

# Basic Fragment Shader

````md magic-move
```wgsl
@fragment
fn fs_main(@builtin(position) coord: vec4<f32>) -> @location(0) vec4<f32> {
  let uv = coord.xy / vec2<f32>(800.0, 600.0);
  return vec4<f32>(uv, 0.0, 1.0);
}
```

```wgsl
@group(0) @binding(0)
var<uniform> u_resolution: vec2<f32>;

@fragment
fn fs_main(@builtin(position) coord: vec4<f32>) -> @location(0) vec4<f32> {
  let uv = coord.xy / u_resolution;
  return vec4<f32>(uv, 0.0, 1.0);
}
```
````

---
layout: cover
---

# Pixel Effect

````md magic-move
```wgsl
@fragment
fn fs_main(@builtin(position) coord: vec4<f32>) -> @location(0) vec4<f32> {
  let uv = coord.xy / u_resolution;
  return vec4<f32>(uv, 0.0, 1.0);
}
```

```wgsl
@fragment
fn fs_main(@builtin(position) coord: vec4<f32>) -> @location(0) vec4<f32> {
  let pixels = 25.0;
  let uv = floor((coord.xy / u_resolution) * pixels) / pixels;
  return vec4<f32>(uv, 0.0, 1.0);
}
```
````

---
layout: cover
---

<script setup>
const shaderCode = `
@group(0) @binding(0)
var<uniform> u_resolution: vec2<f32>;

@fragment
fn fs_main(@builtin(position) coord: vec4<f32>) -> @location(0) vec4<f32> {
  let pixels = 25.0;
  let uv = floor((coord.xy / u_resolution) * pixels) / pixels;
  return vec4<f32>(uv, 0.0, 1.0);
}
`
</script>

<WebGpuCanvas
  :fragmentShader="shaderCode"
/>


---
layout: cover
---

# What is Noise?

Procedural noise is used to simulate randomness.

* Creates organic textures
* Can be controlled with scale, seed, octaves
* Used for terrain, clouds, materials
* Fundamental building block of shader art

---
layout: cover
---

# Simple Noise

````md magic-move
```wgsl
fn rand(p: vec2<f32>) -> f32 {
  return fract(sin(dot(p, vec2<f32>(12.9898, 78.233))) * 43758.5453);
}

fn noise(p: vec2<f32>) -> f32 {
  let i = floor(p);
  let f = fract(p);
  let a = rand(i);
  let b = rand(i + vec2<f32>(1, 0));
  let c = rand(i + vec2<f32>(0, 1));
  let d = rand(i + vec2<f32>(1, 1));
  let u = f * f * (3.0 - 2.0 * f);
  return mix(mix(a, b, u.x), mix(c, d, u.x), u.y);
}
```

```wgsl
@fragment
fn fs_main(@builtin(position) coord: vec4<f32>) -> @location(0) vec4<f32> {
  let uv = coord.xy / u_resolution;
  let n = noise(uv);

  return vec4<f32>(vec3<f32>(n), 1.0);
}
```
````

---
layout: cover
---

<script setup>
const shaderCode = `
@group(0) @binding(0)
var<uniform> u_resolution: vec2<f32>;

fn rand(p: vec2<f32>) -> f32 {
  return fract(sin(dot(p, vec2<f32>(12.9898, 78.233))) * 43758.5453);
}

fn noise(p: vec2<f32>) -> f32 {
  let i = floor(p);
  let f = fract(p);
  let a = rand(i);
  let b = rand(i + vec2<f32>(1, 0));
  let c = rand(i + vec2<f32>(0, 1));
  let d = rand(i + vec2<f32>(1, 1));
  let u = f * f * (3.0 - 2.0 * f);
  return mix(mix(a, b, u.x), mix(c, d, u.x), u.y);
}

@fragment
fn fs_main(@builtin(position) coord: vec4<f32>) -> @location(0) vec4<f32> {
  let uv = coord.xy / u_resolution;
  let n = noise(uv);

  return vec4<f32>(vec3<f32>(n), 1.0);
}
`
</script>

<WebGpuCanvas
  :fragmentShader="shaderCode"
/>

---
layout: cover
---

# Varying Noise

````md magic-move
```wgsl
@fragment
fn fs_main(@builtin(position) coord: vec4<f32>) -> @location(0) vec4<f32> {
  let uv = coord.xy / u_resolution;
  let n = noise(uv);

  return vec4<f32>(vec3<f32>(n), 1.0);
}
```

```wgsl
@group(0) @binding(1)
var<uniform> u_time: f32;

@fragment
fn fs_main(@builtin(position) coord: vec4<f32>) -> @location(0) vec4<f32> {
  let uv = coord.xy / u_resolution;

  let frequency = 0.5;
  let scale = mix(1.0, 20.0, 0.5 * (1.0 + sin(u_time * frequency)));

  let shift_speed = 5.0;
  let shift = vec2<f32>(u_time * shift_speed, 0.0);

  let n = noise(uv * scale - shift);

  return vec4<f32>(vec3<f32>(n), 1.0);
}
```
````

---
layout: cover
---

<script setup>
const shaderCode = `
@group(0) @binding(0)
var<uniform> u_resolution: vec2<f32>;

@group(0) @binding(1)
var<uniform> u_time: f32;

fn rand(p: vec2<f32>) -> f32 {
  return fract(sin(dot(p, vec2<f32>(12.9898, 78.233))) * 43758.5453);
}

fn noise(p: vec2<f32>) -> f32 {
  let i = floor(p);
  let f = fract(p);
  let a = rand(i);
  let b = rand(i + vec2<f32>(1, 0));
  let c = rand(i + vec2<f32>(0, 1));
  let d = rand(i + vec2<f32>(1, 1));
  let u = f * f * (3.0 - 2.0 * f);
  return mix(mix(a, b, u.x), mix(c, d, u.x), u.y);
}

@fragment
fn fs_main(@builtin(position) coord: vec4<f32>) -> @location(0) vec4<f32> {
  let uv = coord.xy / u_resolution;

  let frequency = 0.5;
  let scale = mix(1.0, 20.0, 0.5 * (1.0 + sin(u_time * frequency)));

  let shift_speed = 5.0;
  let shift = vec2<f32>(u_time * shift_speed, 0.0);

  let n = noise(uv * scale - shift);

  return vec4<f32>(vec3<f32>(n), 1.0);
}
`
</script>

<WebGpuCanvas
  :fragmentShader="shaderCode"
  :useTime="true"
/>

---
layout: cover
---

# FBM Noise

FBM = Fractal Brownian Motion

* Combines several noise layers
* Each layer has more detail (frequency ↑, amplitude ↓)
* Creates way more natural noise

---
layout: cover
---

# FBM Noise

```wgsl
fn fbm(p: vec2<f32>, size: f32, octaves: u32) -> f32 {
  var value = 0.0;
  var amp = 0.5;
  var freq = size;
  var coord = p;

  for (var i = 0u; i < octaves; i++) {
    value += noise(coord * freq) * amp;
    coord *= 2.0;
    amp *= 0.5;
  }

  return value;
}
```

---
layout: cover
---

# FBM Noise

````md magic-move
```wgsl
@fragment
fn fs_main(@builtin(position) coord: vec4<f32>) -> @location(0) vec4<f32> {
  let uv = coord.xy / u_resolution;

  ...

  let n = noise(uv * scale - shift);

  return vec4<f32>(vec3<f32>(n), 1.0);
}
```

```wgsl
@fragment
fn fs_main(@builtin(position) coord: vec4<f32>) -> @location(0) vec4<f32> {
  let uv = coord.xy / u_resolution;

  ...

  let size = 6.0;
  let octaves = 5;
  let n = fbm(uv * scale - shift, size, octaves);

  return vec4<f32>(vec3<f32>(n), 1.0);
}
```
````

---
layout: cover
---

<script setup>
const shaderCode = `
@group(0) @binding(0)
var<uniform> u_resolution: vec2<f32>;

@group(0) @binding(1)
var<uniform> u_time: f32;

fn rand(p: vec2<f32>) -> f32 {
  return fract(sin(dot(p, vec2<f32>(12.9898, 78.233))) * 43758.5453);
}

fn noise(p: vec2<f32>) -> f32 {
  let i = floor(p);
  let f = fract(p);
  let a = rand(i);
  let b = rand(i + vec2<f32>(1, 0));
  let c = rand(i + vec2<f32>(0, 1));
  let d = rand(i + vec2<f32>(1, 1));
  let u = f * f * (3.0 - 2.0 * f);
  return mix(mix(a, b, u.x), mix(c, d, u.x), u.y);
}

fn fbm(p: vec2<f32>, size: f32, octaves: u32) -> f32 {
  var value = 0.0;
  var amp = 0.5;
  var freq = size;
  var coord = p;

  for (var i = 0u; i < octaves; i++) {
    value += noise(coord * freq) * amp;
    coord *= 2.0;
    amp *= 0.5;
  }

  return value;
}

@fragment
fn fs_main(@builtin(position) coord: vec4<f32>) -> @location(0) vec4<f32> {
  let uv = coord.xy / u_resolution;

  let frequency = 0.5;
  let scale = mix(1.0, 20.0, 0.5 * (1.0 + sin(u_time * frequency)));

  let shift_speed = 5.0;
  let shift = vec2<f32>(u_time * shift_speed, 0.0);

  let size: f32 = 6.0;
  let octaves: u32 = 5;
  let n = fbm(uv * scale - shift, size, octaves);

  return vec4<f32>(vec3<f32>(n), 1.0);
}
`
</script>

<WebGpuCanvas
  :fragmentShader="shaderCode"
  :useTime="true"
/>

---
layout: cover
---

# So what's the point?

Use noise value as blend factor between colors:

````md magic-move
```wgsl
@fragment
fn fs_main(@builtin(position) coord: vec4<f32>) -> @location(0) vec4<f32> {
  let uv = coord.xy / u_resolution;

  ...

  let n = fbm(uv * scale - shift, size, octaves);

  return vec4<f32>(vec3<f32>(n), 1.0);
}
```

```wgsl
@fragment
fn fs_main(@builtin(position) coord: vec4<f32>) -> @location(0) vec4<f32> {
  let uv = coord.xy / u_resolution;

  ...

  let n = fbm(uv * scale - shift, size, octaves);

  let land = vec3<f32>(0.3, 0.5, 0.2);
  let ocean = vec3<f32>(0.0, 0.2, 0.5);
  let color = mix(ocean, land, n);

  return vec4<f32>(color, 1.0);
}
```
````

---
layout: cover
---

<script setup>
const shaderCode = `
@group(0) @binding(0)
var<uniform> u_resolution: vec2<f32>;

@group(0) @binding(1)
var<uniform> u_time: f32;

fn rand(p: vec2<f32>) -> f32 {
  return fract(sin(dot(p, vec2<f32>(12.9898, 78.233))) * 43758.5453);
}

fn noise(p: vec2<f32>) -> f32 {
  let i = floor(p);
  let f = fract(p);
  let a = rand(i);
  let b = rand(i + vec2<f32>(1, 0));
  let c = rand(i + vec2<f32>(0, 1));
  let d = rand(i + vec2<f32>(1, 1));
  let u = f * f * (3.0 - 2.0 * f);
  return mix(mix(a, b, u.x), mix(c, d, u.x), u.y);
}

fn fbm(p: vec2<f32>, size: f32, octaves: u32) -> f32 {
  var value = 0.0;
  var amp = 0.5;
  var freq = size;
  var coord = p;

  for (var i = 0u; i < octaves; i++) {
    value += noise(coord * freq) * amp;
    coord *= 2.0;
    amp *= 0.5;
  }

  return value;
}

@fragment
fn fs_main(@builtin(position) coord: vec4<f32>) -> @location(0) vec4<f32> {
  let uv = coord.xy / u_resolution;

  let shift_speed = 0.2;
  let shift = vec2<f32>(u_time * shift_speed, 0.0);

  let size: f32 = 6.0;
  let octaves: u32 = 5;
  let n = fbm(uv - shift, size, octaves);

  let land = vec3<f32>(0.3412, 0.5294, 0.2314);
  let ocean = vec3<f32>(0.0, 0.2078, 0.4627);
  let color = mix(ocean, land, clamp(n - 0.2, 0., 1.));

  return vec4<f32>(color, 1.0);
}
`
</script>

<WebGpuCanvas
  :fragmentShader="shaderCode"
  :useTime="true"
/>

---
layout: cover
---

# Circle Mask

````md magic-move
```wgsl
@fragment
fn fs_main(@builtin(position) coord: vec4<f32>) -> @location(0) vec4<f32> {
  let uv = coord.xy / u_resolution;

  ...

  let color = mix(ocean, land, n);

  return vec4<f32>(color, 1.0);
}
```

```wgsl
@fragment
fn fs_main(@builtin(position) coord: vec4<f32>) -> @location(0) vec4<f32> {
  let uv = coord.xy / u_resolution;

  let center = vec2<f32>(0.5);
  let d_center = distance(uv, center);

  ...

  let color = mix(ocean, land, n);

  let a = step(d_center, 0.5);

  return vec4<f32>(color, a);
}
```
````

---
layout: cover
---

<script setup>
const shaderCode = `
@group(0) @binding(0)
var<uniform> u_resolution: vec2<f32>;

@group(0) @binding(1)
var<uniform> u_time: f32;

fn rand(p: vec2<f32>) -> f32 {
  return fract(sin(dot(p, vec2<f32>(12.9898, 78.233))) * 43758.5453);
}

fn noise(p: vec2<f32>) -> f32 {
  let i = floor(p);
  let f = fract(p);
  let a = rand(i);
  let b = rand(i + vec2<f32>(1, 0));
  let c = rand(i + vec2<f32>(0, 1));
  let d = rand(i + vec2<f32>(1, 1));
  let u = f * f * (3.0 - 2.0 * f);
  return mix(mix(a, b, u.x), mix(c, d, u.x), u.y);
}

fn fbm(p: vec2<f32>, size: f32, octaves: u32) -> f32 {
  var value = 0.0;
  var amp = 0.5;
  var freq = size;
  var coord = p;

  for (var i = 0u; i < octaves; i++) {
    value += noise(coord * freq) * amp;
    coord *= 2.0;
    amp *= 0.5;
  }

  return value;
}

@fragment
fn fs_main(@builtin(position) coord: vec4<f32>) -> @location(0) vec4<f32> {
  let uv = coord.xy / u_resolution;

  let center = vec2<f32>(0.5);
  let d_center = distance(uv, center);

  let shift_speed = 0.2;
  let shift = vec2<f32>(u_time * shift_speed, 0.0);

  let size: f32 = 6.0;
  let octaves: u32 = 5;
  let n = fbm(uv - shift, size, octaves);

  let land = vec3<f32>(0.3412, 0.5294, 0.2314);
  let ocean = vec3<f32>(0.0, 0.2078, 0.4627);
  let color = mix(ocean, land, clamp(n - 0.2, 0., 1.));

  let a = step(d_center, 0.5);

  return vec4<f32>(color, a);
}
`
</script>

<WebGpuCanvas
  :fragmentShader="shaderCode"
  :useTime="true"
/>

---
layout: cover
---

# Spherify Effect

This is also known as a fisheye effect.

1. Start with UVs in the range **(0, 0)** to **(1, 1)**
2. Center the UVs around **(0, 0)**
3. Apply a spherical distorition formula

---
layout: cover
---

# Spherify Effect

```wgsl
fn spherify(uv: vec2<f32>) -> vec2<f32> {
  let centered = uv * 2.0 - 1.0;
  let z = sqrt(1.0 - dot(centered, centered));
  let sphere = centered / (z + 1.0);
  return sphere * 0.5 + 0.5;
}
```

---
layout: cover
---

# Spherify Effect

````md magic-move
```wgsl
@fragment
fn fs_main(@builtin(position) coord: vec4<f32>) -> @location(0) vec4<f32> {
  let uv = coord.xy / u_resolution;

  let center = vec2<f32>(0.5);
  let d_center = distance(uv, center);

  ...

  return vec4<f32>(color, a);
}
```

```wgsl
@fragment
fn fs_main(@builtin(position) coord: vec4<f32>) -> @location(0) vec4<f32> {
  var uv = coord.xy / u_resolution;

  let center = vec2<f32>(0.5);
  let d_center = distance(uv, center);

  uv = spherify(uv);

  ...

  return vec4<f32>(color, a);
}
```
````

---
layout: cover
---

<script setup>
const shaderCode = `
@group(0) @binding(0)
var<uniform> u_resolution: vec2<f32>;

@group(0) @binding(1)
var<uniform> u_time: f32;

fn rand(p: vec2<f32>) -> f32 {
  return fract(sin(dot(p, vec2<f32>(12.9898, 78.233))) * 43758.5453);
}

fn noise(p: vec2<f32>) -> f32 {
  let i = floor(p);
  let f = fract(p);
  let a = rand(i);
  let b = rand(i + vec2<f32>(1, 0));
  let c = rand(i + vec2<f32>(0, 1));
  let d = rand(i + vec2<f32>(1, 1));
  let u = f * f * (3.0 - 2.0 * f);
  return mix(mix(a, b, u.x), mix(c, d, u.x), u.y);
}

fn fbm(p: vec2<f32>, size: f32, octaves: u32) -> f32 {
  var value = 0.0;
  var amp = 0.5;
  var freq = size;
  var coord = p;

  for (var i = 0u; i < octaves; i++) {
    value += noise(coord * freq) * amp;
    coord *= 2.0;
    amp *= 0.5;
  }

  return value;
}

fn spherify(uv: vec2<f32>) -> vec2<f32> {
  let centered = uv * 2.0 - 1.0;
  let z = sqrt(1.0 - dot(centered, centered));
  let sphere = centered / (z + 1.0);
  return sphere * 0.5 + 0.5;
}

@fragment
fn fs_main(@builtin(position) coord: vec4<f32>) -> @location(0) vec4<f32> {
  var uv = coord.xy / u_resolution;

  let center = vec2<f32>(0.5);
  let d_center = distance(uv, center);

  uv = spherify(uv);

  let shift_speed = 0.2;
  let shift = vec2<f32>(u_time * shift_speed, 0.0);

  let size: f32 = 6.0;
  let octaves: u32 = 5;
  let n = fbm(uv - shift, size, octaves);

  let land = vec3<f32>(0.3412, 0.5294, 0.2314);
  let ocean = vec3<f32>(0.0, 0.2078, 0.4627);
  let color = mix(ocean, land, clamp(n - 0.2, 0., 1.));

  let a = step(d_center, 0.5);

  return vec4<f32>(color, a);
}
`
</script>

<WebGpuCanvas
  :fragmentShader="shaderCode"
  :useTime="true"
/>

---
layout: cover
---

# Bringing It Together

* Re-add pixel effect
* Use distinct colour bands instead of blending
* Add a cloud layer with another FBM noise layer

---
layout: cover
---

<script setup>
const shaderCode = `
@group(0) @binding(0)
var<uniform> u_resolution: vec2<f32>;

@group(0) @binding(1)
var<uniform> u_time: f32;

fn rand(p: vec2<f32>) -> f32 {
  return fract(sin(dot(p, vec2<f32>(12.9898, 78.233))) * 43758.5453);
}

fn noise(p: vec2<f32>) -> f32 {
  let i = floor(p);
  let f = fract(p);
  let a = rand(i);
  let b = rand(i + vec2<f32>(1, 0));
  let c = rand(i + vec2<f32>(0, 1));
  let d = rand(i + vec2<f32>(1, 1));
  let u = f * f * (3.0 - 2.0 * f);
  return mix(mix(a, b, u.x), mix(c, d, u.x), u.y);
}

fn fbm(p: vec2<f32>, size: f32, octaves: u32) -> f32 {
  var value = 0.0;
  var amp = 0.5;
  var freq = size;
  var coord = p;

  for (var i = 0u; i < octaves; i++) {
    value += noise(coord * freq) * amp;
    coord *= 2.0;
    amp *= 0.5;
  }

  return value;
}

fn spherify(uv: vec2<f32>) -> vec2<f32> {
  let centered = uv * 2.0 - 1.0;
  let z = sqrt(1.0 - dot(centered, centered));
  let sphere = centered / (z + 1.0);
  return sphere * 0.5 + 0.5;
}

@fragment
fn fs_main(@builtin(position) coord: vec4<f32>) -> @location(0) vec4<f32> {
    let pixels = 50.0;
    var uv = floor((coord.xy / u_resolution) * pixels) / pixels;

    let d = distance(uv, vec2<f32>(0.5));
    let mask = step(d, 0.5);

    uv = spherify(uv);

    let shift = vec2<f32>(u_time * 0.01, 0.0);
    let n = fbm(uv - shift, 6.0, 5u);

    let palette = array<vec3<f32>, 6>(
        vec3<f32>(0.0,   0.106, 0.316),
        vec3<f32>(0.0,   0.149, 0.393),
        vec3<f32>(0.0,   0.175, 0.443),
        vec3<f32>(0.18,  0.393, 0.15),
        vec3<f32>(0.255, 0.45,  0.15),
        vec3<f32>(0.293, 0.471, 0.168)
    );
    let t = n * 6.0;
    let i0 = u32(floor(t));
    let i1 = min(i0 + 1u, 5u);
    let f  = fract(t);

    let w = smoothstep(0.45, 0.55, f);
    let base_color = mix(palette[i0], palette[i1], w);

    let cn = fbm(uv * 3.0 - shift * 6.0, 3.0, 3u);
    let a  = smoothstep(0.4, 0.7, cn) * 0.8;

    let col = mix(base_color, vec3<f32>(1.0), a);

    return vec4<f32>(col, mask);
}
`
</script>

<WebGpuCanvas
  :fragmentShader="shaderCode"
  :useTime="true"
/>

---
layout: cover
---

# Now you can make planets.

* What if you were viewing the planet from above the North Pole?
* How would you do Jupiter's bands?
* What about the sun? Hint: different noise
