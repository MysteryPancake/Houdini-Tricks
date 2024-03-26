# CGWiki DLC
Various Houdini tips and tricks I use a bunch. Hope someone finds this helpful!

## Rabbit Hole
These articles grew too long to fit here. They're the most interesting in my opinion, so be sure to check them out!

- [Vexember 2023](Vexember.md)
- [3D Signed Distance Functions](./Houdini_SDFs.md)
- [Lerp and Fit](Lerp.md)
- [Waveforms](./Waveforms.md)
- [Easings](./Easings.md)
- [Normalized Device Coordinates](./NDC.md)
- [4D Geometry](./4D.md)
- [Time Smoothing](https://mysterypancake.github.io/Houdini-Fun/TimeSmoothing) (interactive!)
- [Sound](./Sound.md) (music production!)

## Simple spring solver
Need to overshoot an animation or smooth it over time to reduce bumps? Introducing the simple spring solver!

<p align="left">
  <img src="./images/springsolver2.gif?raw=true" height="280">
  <img src="./images/springsolver.gif?raw=true" height="280">
</p>

I stole this from an article on [2D wave simulation](https://gamedevelopment.tutsplus.com/make-a-splash-with-dynamic-2d-water-effects--gamedev-236t) by Michael Hoffman. The idea is to set a target position and set the acceleration towards the target. This causes a natural overshoot when the object flies past the target, since the velocity takes time to flip. Next you apply damping to stop it going too crazy.

First add a target position to your geometry:

```js
v@targetP = v@P;
```

Next add a solver. Inside the solver, add a point wrangle with this VEX:

```js
float freq = 100.0;
float damping = 5.0;

// Find direction towards target
vector dir = v@targetP - v@P;

// Accelerate towards it (@TimeInc to handle substeps)
vector accel = dir * freq * f@TimeInc * f@TimeInc;
v@v += accel;

// Dampen velocity to prevent infinite overshoot
v@v /= 1.0 + damping * f@TimeInc;
v@P += v@v;
```

To adjust motion over time, plug the current geometry into the second input and use it instead of `v@targetP`:

```js
// Find direction towards target
vector dir = v@opinput1_P - v@P;
```

**UPDATE:** The spring solver in [MOPs](https://www.motionoperators.com/) has better damping:

```js
float mass = 1.0;
float k = 0.4;
float damping = 0.9;

// Find direction towards target
vector dir = v@targetP - v@P;

// Accelerate towards it
vector force = k * dir;
vector accel = force / mass;
v@v += accel;

// Dampen velocity to prevent infinite overshoot
v@v *= damping;
v@P += v@v;
```

## RBDs: Make an aimbot (find velocity to hit a target)
Want to prepare for the next war but can't solve projectile motion? Never fear, the Ballistic Path node is all you need.

[![Aimbot tutorial](https://img.youtube.com/vi/Ed2_62BlOFA/mqdefault.jpg)](https://youtu.be/Ed2_62BlOFA)

### Hit a static target
1. Connect your projectile to a Ballistic Path node.
2. Set the Launch Method to "Targeted" and disable drag.
3. Add a `@targetP` attribute to your projectile. Set it to the centroid of the target object.

```js
v@targetP = getbbox_center(1);
```

4. You should see an arc. Transfer the velocity of the first point of the arc to your projectile.

```js
v@v = point(1, "v", 0);
```

5. Connect everything to a RBD Solver.

6. Use "Life" to set the height of the path, and lower the "FPS" to reduce unneeded points.

<img src="./images/aimbot_static.gif?raw=true" height="320">

### Hit a moving target
Use the same method as before, but sample the target's position forwards in time.

1. On the Ballistic Path node, set the Targeting Method to "Life".
2. Copy the "Life" attribute. It's the number of seconds until we hit the target. We need to find where the target is at that time.
3. Add a Time Shift node to the target (before the centroid is calculated). Set it to the current time plus the "Life" attribute.

<img src="./images/aimbot_moving.gif?raw=true" height="320">

[Download the HIP file!](./hips/aimbot.hipnc?raw=true)

### Hit multiple targets
If your "Life" is the same for all projectiles, extract multiple centroids and transfer velocities from the first point of each arc based on connectivity. Try enabling "Path Point Index" on Ballistic Path and blasting all non-zero indices.

If your "Life" changes per target, use a for loop instead.

<img src="./images/aimbot.gif?raw=true" height="320">

## Smooth steps
Smoothstep's evil uncle, smooth steps. This helps for staggering animations, like points moving along lines.

<img src="./images/vexember/vexemberhilbert.gif?raw=true" width="500">

Start with regular steps. This is the integer component:

<img src="./images/waveforms/steps.png?raw=true" width="600">

Use modulo to form a line per step, then clamp it below 1. This is the fractional component:

<img src="./images/waveforms/stepclamp.png?raw=true" width="600">

Add them together to achieve smooth steps:

<img src="./images/waveforms/linearsteps.png?raw=true" width="600">

```js
float x = f@Time; // Replace with whatever you want to step
float width = 2; // Size of each step
float steepness = 1; // Gradient of each step

int int_step = floor(x / width); // Integer component, steps
float frac_step = min(1, x % width * steepness); // Fractional component, lines
float smooth_steps = int_step + frac_step; // Both combined, smooth steps
```

## Generating circles
Sometimes you need to generate circles without relying on built-in nodes, like to know the phase.

Luckily it's easy, just use `sin()` on one axis and `cos()` on the other:

<img src="./images/movecircle.webp?raw=true" width="200" align="left">

```js
float theta = chf("theta");
float radius = chf("radius");
v@P = set(cos(theta), 0, sin(theta)) * radius;
```

See [Waveforms](./Waveforms.md) for more about sine and cosine.

<br clear="left" />

To draw a circle, add points while moving between 0 and `2*PI`:

<img src="./images/pointcircle.png?raw=true" width="200" align="left">

```js
int num_points = chi("point_count");
float radius = chf("radius");

for (int i = 0; i < num_points; ++i) {
    // Sin/cos range from 0 to 2*PI, so remap from 0-1 to 0-2*PI
    float theta = float(i) / num_points * 2 * PI;
    // Use sin and cos on either axis to form a circle
    vector pos = set(cos(theta), 0, sin(theta)) * radius;
    addpoint(0, pos);
}
```

<br clear="left" />

To connect the points, you can use `addprim()`:

<img src="./images/circleconnected.png?raw=true" width="200" align="left">

```js
int num_points = chi("point_count");
float radius = chf("radius");
int points[];

for (int i = 0; i < num_points; ++i) {
    // Sin/cos range from 0 to 2*PI, so remap from 0-1 to 0-2*PI
    float theta = float(i) / num_points * 2 * PI;
    // Use sin and cos on either axis to form a circle
    vector pos = set(cos(theta), 0, sin(theta)) * radius;
    // Add the point to the array for polyline
    int id = addpoint(0, pos);
    append(points, id);
}
// Connect all the points with a polygon
addprim(0, "poly", points);
```

<br clear="left" />

[Download the HIP file!](./hips/circle.hipnc?raw=true)

## Sweep in VEX

Ever wanted to remake Sweep in VEX? Me neither, but let's do it anyway!

<img src="./images/sweep1.png?raw=true" width="200" align="left">

First we need to know the orientation of each point along the curve.

The easiest way is with the Orientation Along Curve node. Enable the X and Y vectors `@out` and `@up`.

The `@out` and `@up` vectors define a 3D plane we can slap circles on.

For example, multiply `sin()` by one vector and `cos()` by the other, then add them together. This gets a point around a circle flattened to the plane.

```js
v@P += sin(phase) * v@up - sin(phase) * v@out;
```

<br clear="left" />

<img src="./images/sweep2.png?raw=true" width="200" align="left">

Using that in a for loop, we can replace each point with a correctly oriented circle.

```js
int cols = chi("columns");
float radius = chf("radius");
float angle = radians(chf("angle"));
int num_points = npoints(0);

for (int i = 0; i < cols; ++i) {
    // Column factor (0-1 exclusive)
    float col_factor = float(i) / cols;
    // Row factor (0-1 inclusive)
    float row_factor = float(i@ptnum) / (num_points - 1);

    // Scale and rotation ramps
    float scale_ramp = chramp("scale_ramp", row_factor);
    float roll_ramp = chramp("roll_ramp", row_factor);

    // Generate circles on the plane defined by v@out and v@up
    float phase = 2 * PI * (col_factor + roll_ramp) + angle;
    vector offset = cos(phase) * v@out - sin(phase) * v@up;

    addpoint(0, v@P + offset * radius * scale_ramp);
}

// Remove original points
removepoint(0, i@ptnum);
```

<br clear="left" />

<img src="./images/sweep3.png?raw=true" width="200" align="left">

Now we just need to connect the circles with quads. To make a quad, we need the index of each corner.

The tricky bit is looping back to the start after we reach the end of each ring, which takes a bit of modulo.

```js
int num_points = npoints(0);
int cols = chi("columns");

// Skip last connections when path is open
int closed = chi("close_path");
if (!closed && i@ptnum >= num_points - cols) return;

// Row and column indexes per point
i@ptrow = i@ptnum / cols;
i@ptcol = i@ptnum % cols;

// Point indexes of the 4 corners of each quad
int corner1 = i@ptnum;
int corner2 = i@ptrow * cols + (i@ptnum + 1) % cols;
int corner3 = (corner2 + cols) % num_points;
int corner4 = (corner1 + cols) % num_points;

// Add quads
int prim_id = addprim(0, "poly", corner1, corner2, corner3, corner4);

// Row and column indexes per prim
setprimattrib(0, "primrow", prim_id, i@ptrow);
setprimattrib(0, "primcol", prim_id, i@ptcol);
```

<br clear="left" />

The same code works for custom cross sections, though it's easier to use Copy to Points to orient each cross section.

Just make sure the points are sorted and `cols` matches the point count of the cross section!

<p align="left">
  <img src="./images/sweep4.png?raw=true" height="250">
  <img src="./images/sweep5.png?raw=true" height="250">
</p>

If `cols` doesn't match the point count, never fear. You'll get cool trippy looking shapes!

<p align="left">
  <img src="./images/sweepshape1.png?raw=true" height="300">
  <img src="./images/sweepshape2.png?raw=true" height="300">
</p>

[Download the HIP file!](./hips/vex_sweep.hiplc?raw=true)

## Sweep varying cross sections
I showed this to Lara Belaeva, who pushes Sweep to its limits on LinkedIn. She tried it already, but had an interesting point:

> If I decided to build my own Sweep I would try to do it similar to what we have in Plasticity. In Plasticity you can take several different cross-sections, put them in different regions of the curve, and the Sweep creates the blends between them across the curve. I tried to make such a tool but it's still so-so. Houdini's Sweep also can use different cross sections, but doesn't create blends between them

It would be cool if Sweep supported different cross sections. But what if it does already?

Try using normals from Orientation Along Curve, then copy different cross sections onto each point with Copy to Points.

Plug that into Sweep's second input and you'll see we can use different cross sections after all!

<img src="./images/sweep8.png?raw=true" height="300">

Although it works, if you look closely it resampled each cross section to the same number of points. Lara tried this too:

> I just resampled these cross-sections with a constant number of points and then they were sort of placed along the curve based on curveu attribute, and then the algorithm connected 1-1-1, 2-2-2, 3-3-3 points of these cross-sections with polylines. And then these polylines were turned into a mesh with Loft.

So what if we want the exact same cross sections? PolyBridge comes to our rescue!

Plug the cross sections into a PolyBridge node instead. Set the source group to all prims from 0 to the second last:

```js
0-`nprims(0)-2`
```

Set the destination group to all prims from 1 to the last:

```js
1-`nprims(0)-1`
```

Now the cross sections connect perfectly without any resampling!

<p align="left">
  <img src="./images/sweep8.png?raw=true" height="200">
  <img src="./images/sweep7.png?raw=true" height="200">
</p>

[Download the HIP file!](./hips/sweep_varying_cross_sections.hiplc?raw=true)

## Reusing VEX code in multiple wrangles
Most programming languages have ways to share and reuse code. C has `#include`, JavaScript has `import`, but what about VEX?

VEX has `#include` as well, but sadly it only works if you put the file in a specific Houdini directory.

Luckily there's a secret way to reuse code without `#include`! I found it in a couple of LABS nodes.

First add any node with a string property. It can be a wrangle, a null or anything else. In the string property, type the functions you want to share.

```js
vector addToPos(vector p) {
    return p + {1, 2, 3};
}
```

Now you can import and run those functions in any other wrangle with `chs("../path/to/string_property")` enclosed in backticks!

```js
// Append the code string and run it (like #include)
`chs("../path/to/string_property")`

v@P = addToPos(v@P);
```

**UPDATE:** Van and WaffleboyTom said this is evil since it causes the code to recompile. Use if you dare!

[Download the HIP file!](./hips/including_vex_code.hip?raw=true)

## Nearest point to any attribute
`nearpoint()` finds the closest point to `@P`, but what if you need the closest point to something else?

The shortest way is abusing `pcfind()`, which takes any input as the position channel:

```js
string attrib = "density";
float target = 16.0;
int nearest_id = pcfind(0, attrib, target, 99999.9, 1)[0];
```

Another option is using Attribute Swap to move the attribute to `@P`. Keep in mind this only works for certain types of attributes.

To find an exact match, use `findattribval()` instead.

## Unwrap
Swalsch told us about a top secret alternative to the above, known as [unwrap](https://www.sidefx.com/docs/houdini/vex/functions/geounwrap.html).

It changes the context of any VEX function, allowing you to override functions to work with any attribute.

For example, `nearpoint()` uses `@P`. Normally you have to round trip to use another attribute like `@uv`:

```js
// Get the geometry position closest to a UV coordinate
vector world_pos = uvsample(0, "P", "uv", chv("uv_coordinate"));

// Find the nearest point to that position
i@near_id = nearpoint(0, world_pos);
```

The direct way is using unwrap to replace the context:

```js
// All in one, thanks @swalsch!
i@near_id = nearpoint("unwrap:uv opinput:0", chv("uv_coordinate"));
```

[Download the HIP file!](./hips/geounwrap.hipnc?raw=true)

## Sampling environment maps
A cool trick from [John Kunz](https://www.johnkunz.com/) is sampling a HDRI using VEX. It's a cheap way to get environment mapping without leaving the viewport.

<img src="./images/hdrisample.png?raw=true" height="320">

```js
// Insert your camera position here
vector cam_pos = fromNDC("/obj/cam1", {0, 0, 0});

// John Kunz magic
vector r = normalize(reflect(normalize(v@P - cam_pos), v@N));
vector uv = set(atan2(-r.z, -r.x) / PI + 0.5, r.y * 0.5 + 0.5, 0);
v@Cd = texture("$HFS/houdini/pic/hdri/HDRIHaven_skylit_garage_2k.rat", uv.x, uv.y);
```

[Download the HIP file!](./hips/hdrisample.hipnc?raw=true)

## Smoothing volumes with VEX
Levin on the CGWiki Discord wanted to blur volumes in VEX. You can do it by sample neighbors in a box and averaging them together. This is slower than the built-in volume nodes, but might be useful one day:

```js
float density_sum = 0;
int num_samples = 0;

int voxel_radius = chi("voxel_radius");
for (int x = -voxel_radius; x <= voxel_radius; ++x) {
    for (int y = -voxel_radius; y <= voxel_radius; ++y) {
        for (int z = -voxel_radius; z <= voxel_radius; ++z) {
            // Sample voxel at offset index
            vector voxel = set(i@ix + x, i@iy + y, i@iz + z);
            float density = volumeindex(0, 0, voxel);

            // Add to sum and sample count
            density_sum += density;
            ++num_samples;
        }
    }
}

f@density = density_sum / num_samples;
```

<img src="./images/volumesmoothing.png?raw=true" height="320">

[Download the HIP file!](./hips/volume_smoothing.hiplc?raw=true)

## Applying orient to packed prims
Copy to Points automatically applies quaternions like `@orient`, but what if you need the same effect without Copy to Points?

Normally you'd set the `transform` intrinsic, but this replaces everything. To just replace the `@orient`, set `pointinstancetransform` to 1.

```js
setprimintrinsic(0, "pointinstancetransform", i@elemnum, 1);
```

Thanks to WaffleboyTom for this tip!

## Snapping to the floor
Often it's nice to organise geometry by snapping it to the floor. Here's a few ways to do it!

[Download the HIP file!](./hips/snaptofloor.hipnc?raw=true)

### Match Size
The easiest way is using a Match Size node with "Justify Y" set to "Min". It snaps the position only, and won't affect the rotation.

<img src="./images/centersnap.png?raw=true" height="320">

### Dihedral
To affect the rotation too, swalsch suggested using `dihedral()`. You can use it to rotate the normal towards a down vector.

First the object needs prim normals, which you can add with a Normal node set to "Primitives".

<img src="./images/primnormals.png?raw=true" height="320">

Next pick a prim to snap to the floor, get its normal and use `dihedral()` to build the rotation matrix.

```js
int primIndex = 2670;

// Snap object to prim center
vector centroid = prim(0, "P", primIndex);
vector baseN = prim(0, "N", primIndex);
@P -= centroid;

// Rotate normal vector towards down vector
matrix3 rotMat = dihedral(baseN, {0, -1, 0});
@P *= rotMat;
```

<img src="./images/floorsnapsingle.png?raw=true" height="480">

To use multiple prims, make a prim group and average out the normals within that group.

```js
// Get prims in a prim group called "bottom"
int prims[] = expandprimgroup(0, "bottom");
int primCount = len(prims);

// Find average position and normal
vector posSum = 0, normalSum = 0;
foreach (int primIndex; prims) {
    posSum += prim(0, "P", primIndex);
    normalSum += prim(0, "N", primIndex);
}

// Snap object to average position (same as getbbox_center(0, "bottom"))
v@P -= posSum / primCount;

// Rotate average normal towards down vector
matrix3 rotMat = dihedral(normalSum / primCount, {0, -1, 0});
v@P *= rotMat;
```

### Extract Transform
The hackiest way is abusing Extract Transform. You flatten the prims to the floor, then approximate the transform for it.

This affects position and rotation, but isn't as good as `dihedral()` since it won't flip the object past 180 degrees.

1. Blast all the prims except the ones you want to snap. Use a Transform node to center them and scale them to 0 on the Y axis.

<img src="./images/floorflattenprims.png?raw=true" height="480">

2. Use Extract Transform to find the transform. Make sure "Extraction Method" is "Translation and Rotation" to avoid scaling and skewing!
3. Use Transform Pieces to apply the transform, forcing the prims to the floor as much as possible.

<img src="./images/floorextracttransform.png?raw=true" height="480">

## Vellum: Stop wobbling, be rigid and bouncy
Vellum is usually wobbly like jelly, making hard objects tricky to achieve without an RBD Solver.

If you absolutely need Vellum, a great technique comes from Matt Estela.

1. Go to the "Advanced" tab of the Vellum Solver and disable "Max Acceleration".
2. For any shapes you want to make rigid, add a "Shape Match" constraint and make it super stiff.
3. Disable anything to do with smoothing or softening and lower the thickness.

Keep the topology as basic as possible and try increasing the substeps to make Shape Match even more stiff.

## Stabilize and unstabilize geometry
Otherwise known as swapping reference frames, rest to animated, world to local, frozen to unfrozen...

Perhaps the best trick in Houdini is moving geometry to a rest pose, doing something and moving it back to an animated pose.

It fixes tons of issues like broken collisions, VDBs jumping around, plus aliasing and quantization artifacts.

### Rigid geometry
Extract Transform and Transform Pieces are your best friends.

1. Use Time Shift to freeze the animated geometry. This is your rest pose.
2. If possible, blast everything except a single prim for optimization.
3. Use Extract Transform to calculate the transform from the animated prim to the frozen prim.
5. Use Transform Pieces to stabilize the geometry. Make sure to tick "Invert Transformation"!
6. Do stuff while it's stabilized.
7. Use Transform Pieces to move the geometry back to the animated pose.

As well as Transform Pieces, you can set Extract Transform to output a matrix and transform manually in VEX:

```js
// Extract Transform matrix from input 1
matrix mat = point(1, "transform", 0);
// Forward transform
v@P *= mat;
// Inverse transform
v@P *= invert(mat);
```

<img src="./images/extracttransform.webp?raw=true" width="500">

[Download the HIP file!](./hips/extracttransform.hipnc?raw=true)

<img src="./images/inverttransform.webp?raw=true" width="500">

[Download the HIP file!](./hips/inverttransform.hipnc?raw=true)

### Bendy geometry
For simple cases, Point Deform is your best friend.

1. Use Time Shift to freeze the animated geometry. This is your rest pose.
2. Point Deform the animated geometry with the inputs in the wrong order, so it deforms from animated to rest.
3. Do stuff while it's frozen.
4. Point Deform back with the inputs in the right order, so it deforms from rest to animated.

If you need better interpolation, try `primuv()` or Attribute Interpolate. This works great for proxy geometry, for example remeshed cloth.

1. Use `xyzdist()` to map the positions of the good geometry onto the proxy geometry, both in rest position.

```js
xyzdist(1, v@P, i@near_prim, v@near_uv);
```

2. Simulate the proxy geometry.
2. In another wrangle, use `primuv()` to match the good geometry's position to the animated proxy.

```js
v@P = primuv(1, "P", i@near_prim, v@near_uv);
```

## Primuv vs actual UVs
A common misconception is `primuv()` uses the actual UV map of the geometry. This would cause problems if the UVs overlapped.

Instead it uses intrinsic UVs. Intrinsic UVs are indexed by prim and range from 0 to 1 per prim. Since each prim is separated by index, it'll never overlap.

|Regular UVs|Intrinsic UVS|
|---|---|
|<img src="./images/regularuv.png?raw=true">|<img src="./images/primuv.png?raw=true">|

If you want to use the actual UVs, use `uvsample()` instead.

## Select inside or outside
Sometimes you need to select the inside or outside of double-sided geometry, for example to make single-sided geometry if Fuse doesn't work.

The normals are great whenever you need to select anything by direction. Usually you can use a Group node set to "Keep By Normals", then use "Backface from" to pick the interior.

If it screws up, here's another approach. Assuming the interior points inwards and the exterior points outwards, the normals of the interior should point roughly towards the center. That means to detect the interior, you can compare the direction of the normal with the direction towards the center.

1. Pick a center point. Extract Centroid is good for this.

```js
vector center = point(1, "P", 0);
```

2. Find the direction towards the center.

```js
vector dir = normalize(center - v@P);
```

3. Compare it to the normal using a dot product. This tells you how much the normal faces the center (-1 to 1).

```js
float correlation = dot(dir, v@N);
@group_inside = correlation > ch("threshold");
```

<img src="./images/inside.png?raw=true" height="320">

[Download the HIP file!](./hips/inside.hipnc?raw=true)

## Fluids: Fix gap between surfaces
Usually liquids resting on a surface have a small gap due to the collision geometry, easier to see once rendered.

A tip from Raphael Gadot is to transfer normals from the surface onto the liquid with some falloff. This greatly improves the blending.

## Optimize everything
Is your scene slow? Don't blame Houdini, it's likely you haven't optimized properly.

- Using subdivided geometry? PolyReduce and Remesh it down to the simplest form.
- Using Copy to Points? Pack the geometry and use instancing. If not possible, make a few variants in a for loop and use packed versions of those.
- Using Vellum? Keep the overall substeps at 5-10 and focus on the relevant substeps, like collision substeps or constraint substeps.
- Using Alembic or USD? Freeze non-animated geometry with a Time Shift node so it never gets cached twice.
- Using surface collisions? Use Convex Hull, Convex Decomposition, PolyReduce and Remesh to simplify detailed geometry and improve collisions.
- Using volume collisions? Split animated and non-animated geometry into separate volumes, freeze the non-animated one with Time Shift, then merge with VDB Combine.
- Using Pyro? Keep the total voxel count as low as possible, a million is usually plenty. Convert to VDB to prune blank voxels from memory.
- Use File Caches religiously, and be sure to save your caches on a fast SSD instead of a HDD!

Use Houdini's [performance monitor](https://www.sidefx.com/docs/houdini/ref/panes/perfmon.html) to track down what's slowest.

## Be careful with velocity
Velocity is easy to overlook and hard to get right. I've rendered full shots before realising I forgot to put velocity on deforming geo, transfer it to packed geo, or it doesn't line up.

A great tip from Lewis Taylor is to double check velocities from POP sims. It sometimes ignores POP forces and calculates an incorrect result.

For checking velocities, a tip from Ben Anderson is Time Shift a frame backward, template it and display velocity. You should see a line between the past and present position.

## Be careful combining VDBs
Combining multiple pairs of VDBs is often unpredictable, for example combining two sims by density may skip velocity. 

Make sure to combine each VDB pair separately, then feed all pairs into a merge node.

## Be careful with typecasting
I used to do this to generate random velocities between -0.5 and 0.5. See if you can spot the problem.

```js
v@v = rand(i@ptnum) - 0.5;
```

The issue is VEX uses the float version of `rand()`, making the result 1D:

```js
float x = rand(i@ptnum);
v@v = x - 0.5;
```

<img src="./images/velocity_1d.png?raw=true" height="320">

To get a 3D result, there are two options. Either explicitly declare 0.5 as a vector:

```js
v@v = rand(i@ptnum) - vector(0.5);
```

Or explicity declare `rand()` as a vector:

```js
vector x = rand(i@ptnum);
v@v = x - 0.5;
```

This happens a lot, so always explicitly declare types to be safe!

## Be careful with random
Even with fixed typecasting, there's still a problem. See if you can spot it:

```js
v@v = rand(i@ptnum) - vector(0.5);
```

What shape would you expect to see? Surely a sphere, since it's centered at 0 and random in all directions?

Unfortunately it's a cube, since the range is -0.5 to 0.5 on all axes separately.

<img src="./images/velocity_cube.png?raw=true" height="320">

### Random direction, random length
To get a sphere and random vector lengths, use `sample_sphere_uniform()`:

```js
v@v = sample_sphere_uniform(rand(i@ptnum));
```

Roughly equivalent to the following:

```js
v@v = normalize(rand(i@ptnum) - vector(0.5)) * rand(i@ptnum + 1);
```

<img src="./images/velocity_sphere.png?raw=true" height="320">

### Random direction, constant length
To get a sphere and normalized vector lengths, use `sample_direction_uniform()`:

```js
v@v = sample_direction_uniform(rand(i@ptnum));
```

Roughly equivalent to the following:

```js
v@v = normalize(rand(i@ptnum) - vector(0.5));
```

<img src="./images/velocity_direction.png?raw=true" height="320">

## Split vector magnitude and direction
Sometimes you need to change part of a vector but not the other, like to randomize velocity but inherit the magnitude. It's easy with rotation, but here's a more general approach:

```js
// Deconstruction
vector dir = normalize(v@v);
float mag = length(v@v);

// Modify magnitude or direction here
dir = sample_direction_uniform(rand(@ptnum));

// Reconstruction, make sure direction is normalized
v@v = dir * mag;
```

## POP: Make particles look like fluid
A key characteristic of fluid is how it sticks together, forming clumps and strands. POP Fluid tries to emulate this, but it doesn't look as good as FLIP.

To get nicer clumps, a tip from Raphael Gadot is to use Attribute Blur set to "Proximity". Though it won't affect the motion, it looks incredible on still frames.

## Smoke / Fluids: Fix moving colliders
Fluids often screw up whenever colliders move, like water in a moving cup or smoke in an elevator. Either the collider deletes the volume as it moves, or velocity doesn't transfer properly from the collider.

A great fix comes from Raphael Gadot: Stabilise the collider, freeze it in place. Simulate in local space, apply forces in relative space, then invert back to world space. This works best for enclosed containers or pinned geometry, since it's hard to mix local and world sims.

### 1. Relative gravity
1. Add an `@up` vector in world space (before Transform Pieces).

```js
v@up = {0, 1, 0};
```

2. Add a Gravity Force node to your sim (after Transform Pieces). Use the transformed `@up` vector as your gravity force.

```js
Force X = -9.81 * point(-1, 0, "up", 0)
Force Y = -9.81 * point(-1, 0, "up", 1)
Force Z = -9.81 * point(-1, 0, "up", 2)
```

Make sure the force is "Set Always"!

### 2. Relative acceleration
1. Add a Trail node set to "Calculate Velocity", then enable "Calculate Acceleration". It's faster to do this after packing so it only trails one point.

2. Add another Gravity Force node, using negative `@accel` as your force vector.

```js
Force X = -point(-1, 0, "accel", 0)
Force Y = -point(-1, 0, "accel", 1)
Force Z = -point(-1, 0, "accel", 2)
```

Make sure the force is "Set Always"!

### 3. Stabilise (world to local)
1. Pick a face on the collider you want to stabilise. Blast everything except that face.
2. Time freeze that face with a Time Shift node.
3. Use an Extract Transform node to compare the frozen face to the moving face. That tells you how the collider moves over time, allowing you to cancel out the movement.
4. Pack everything else. Make sure to enable "No Point Velocities".
5. Plug the Pack into a Transform Pieces node, then plug Extract Transform into the other input.
6. You should see the collider frozen in place. If not, try swapping the Time Shift to the other input in your Extract Transform node.
7. Unpack and do your sim in local space.
8. Pack the sim result.
9. Add another Transform Pieces node with the same Extract Transform input. This time, set it to "Invert Transformation" to go back to world space.
10. Unpack the world space result.

If you want to deal with open containers, the easiest way is to do a separate sim when the fluid exits the container. This is done by killing points outside the container, then feeding the killed points into the other sim. Make sure to nuke all point attributes to keep it clean for the next sim.

Another tip is use "Central Difference" when calculating the velocity. This gives the fluid more time to move away from the collider.

## Cloth: Fix missing preroll
Cloth sims work best with preroll starting in a neutral rest pose. For example, the character starts in an A-pose or T-pose before transitioning into the animation. If anim screwed you over, never fear! Preroll can be added in Houdini.

### With FBX
1. Export the animated character as FBX. Make sure to include the skeleton!
2. Import the character with a FBX Character Import node.
3. Use a Skeleton Blend node to blend from the rest skeleton to the animated skeleton. If the rest skeleton has a bad pose, fix it with the Rig Pose node. Alternatively, export another FBX posed to your liking. FBX Character Import that animated skeleton as the rest skeleton.
4. Use the Time Shift node to move the animation forward so it doesn't bleed into the preroll. This can also be done in the "Timing" menu of FBX Character Import.
5. Use Bone Deform to animate the skin based on Skeleton Blend.

<img src="./images/fbxtransition.webp?raw=true" height="320">

[Download the HIP file!](./hips/fbxtransition.hipnc?raw=true)

### Without FBX
One option is using a Blend Shapes node, but you'll find the limbs usually clip through the body as it swaps from the T-pose to the animated pose. My sketchy method of improving this is using Extract Transform and Transform Pieces.

1. Use blast to isolate a face from the character's chest.
2. Use Extract Transform and Transform Pieces to transform the T-pose to match the animated pose.

In other words, the T-posed character flies over to the animated position and rotates to match it. That gives you an easier time using Blend Shapes, since it only has to move the arms and legs a short distance to match the animated pose.

To avoid ruffling the clothes, skip the flying step. Just move the clothes directly to the new T-pose location using Transform Pieces.

Another option could be Labs Straight Skeleton 3D. It generates a skeleton from any mesh which could help with blending, but I haven't tried it myself.

## Cloth: Fix rest pose clipping
Cloth sims screw up from clipping, especially when clipped from the start. One option is growing the character into the cloth. 

1. Disable gravity in the cloth sim.
2. Use Smooth, Peak or VDB Reshape to shrink the character.
3. Animate the character growing.
4. Enable gravity in the cloth sim.
5. Use the Time Shift node to move the animation forward so it doesn't bleed into the growing. 

## Cloth: Layer stacking
One little known feature of Vellum Cloth (at least to me) is layering. It can improve the physics of overlapping garments, like jackets on top of t-shirts. 

1. In Vellum Configure Cloth, use the "Layer" setting to define the ordering, bottom to top.
2. On the Vellum Solver under "Collisions", enable "Layer Shock". Lower layers are simulated much heavier than higher layers. 

## Karma: Fix motion blur
Motion blur in Karma can be pretty unpredictable, especially with packed instances.

A great fix comes from [Matt Estela](https://tokeru.com/cgwiki/UsdGuide18.html#motion-blur): just add a Cache node set to "Rolling Window". Usually I use 1 frame before and 1 frame after.

This is faster than the new Motion Blur node, which caches the entire timeline at once. It also fixes issues with animated materials.

## Attribute min / max / average...
Use Attribute Promote set to "Detail" with the appropriate mode.

## Alternating rows for brick walls
The tricky part about modelling brick walls is the alternating pattern. Every second row is slid across by half a brick's width. How would you create this pattern? Manual interpolation? Primuv?

An easy way is working subtractively. Take the base curve and resample it. This gives you the first row. For the second row, subdivide the first. Use a "Group by Range" node to select every second point, then delete them with a "Dissolve" node.

## Access context geometry inside solver
If you need geometry in a context that doesn't provide it (like the forces of a Vellum Solver), just drop down a SOP Solver. You can use Object Merge inside a SOP Solver to grab geometry from anywhere else too. Great for feedback loops!

## Pyro: Fix mushrooms
Many techniques work depending on the situation. Sometimes more randomisation is needed, other times the velocity needs reducing.

A common technique is cranking up the disturbance. Controlling it by speed helps add it where mushrooms are likely to form.

## Negative frame ranges
Seems obvious but worth noting: Unlike some software, Houdini supports negative frame ranges.

For preroll you can always start simulating on a negative frame without needing to time shift anything.

<img src="./images/negative_framerange.png?raw=true">

## Fluids: Fix density loss 
Don't take this section seriously. These are just techniques which seem to work for me.

Density loss often happens when Surface Tension is enabled. Droplets tend to disappear when bunched too close together, so try disabling it before anything else. 

Grid Scale and Particle Radius Scale also affect the density. According to [SideFX](https://www.sidefx.com/docs/houdini/nodes/dop/flipsolver.html), if the Particle Radius Scale divided by the Grid Scale is at least sqrt(3)/2, it will never be underresolved. 

No idea if this affects density, but just in case here's the minimum:

```js
Particle Radius Scale = Grid Scale × (sqrt(3)/2)
Grid Scale = Particle Radius Scale / (sqrt(3)/2)
```
