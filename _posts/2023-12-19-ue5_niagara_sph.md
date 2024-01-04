---
title: 'Create SPH Fluid using UE5 Niagara System'
show_date: true
permalink: /posts/2023/12/ue5_niagara_sph/
tags:
  - UE5
  - Niagara System
  - Game Effect
  - Fluid Simulation
  - SPH
toc: true
toc_sticky: true
author_profile: false
header:
  image: /assets/images/ue5_niagara_sph/sph_header_image.png
---

It is an implementation note of SPH using UE5 Niagara System, related about usage of Simulation Stage, Grid 3D, screen space renderring using depth buffer and so on.

Github repository: https://github.com/CheapMeow/UE5-NiagaraSPH

I also learn other's tutorial: https://www.bilibili.com/video/BV1pr4y1v78B

## SPH Introduction

Basic knowledge about SPH can see other tutorial.

[Matthias Müller et al, "Particle-based fluid simulation for interactive applications"](https://dl.acm.org/doi/10.5555/846276.846298)

## Niagara System Setup

Basic knowledge about Niagara System can see other tutorial.

To setup, change `Sim Target` to `GPUCompute Sim`, because our Niagara System may calculate millions of particles.

Change `Calculate Bounds Mode` to `Fixed`, because particles are upload to GPU so it is hard to calculate bounds every frame.

Change `Life Cycle Mode` to `Self`, in order to set `Loop Behaviour` as `Infinite`.

### Add Neighbor Grid3D from empty

One data structure we should use is `Neighbor Grid3D`, we will store our particle indice into grid, for convience of searching neighbor particles in 8 grids around certain particle.

In Niagara System window, find `Parameter` window (not `Details` windows), find `Emitter Attributes`, click "+" button, search `neighbor` in popup, then you will find `Neighbor Grid3D` item, confirm.

You know the adding parameters of Niagara System is similar to actor blueprint.

<figure style="width: 647px" class="align-center">
<img src="/assets/images/ue5_niagara_sph/neighbor_grid.png">
<figcaption align = "center">Fig: Neighbor Grid</figcaption>
</figure>

After creating a `Neighbor Grid3D` parameter, you should drag it into you emitter. Then `Neighbor Grid3D` will take effect in this emitter.

<figure style="width: 247px" class="align-center">
<img src="/assets/images/ue5_niagara_sph/after_drag_neighbor_grid_to_emitter.png">
<figcaption align = "center">Fig: After Drag Neighbor Grid to Emitter</figcaption>
</figure>

### Using module script from content example

After you add `Neighbor Grid3D` variable, you should also define some variable about extent, transform matrix, etc. To skip these details at the first time, I copy two modules `Initialize Neighbor Grid` and `Populate Neighbor Grid` from offical Content Example.

<figure style="width: 428px" class="align-center">
<img src="/assets/images/ue5_niagara_sph/copy_modules.png">
<figcaption align = "center">Fig: Copy Modules</figcaption>
</figure>

However, after doing this, I add `Spawn Rate` and then click `Fix issue`, try to add `Emitter State`. But UE log "fail to add emitter state" becuase "could not find location".

I don't know why, so I create a emitter from existing copy, and delete redundant module from it. So now I get no error.

<figure style="width: 440px" class="align-center">
<img src="/assets/images/ue5_niagara_sph/copy_other_emitter_to_get_emitter_state.png">
<figcaption align = "center">Fig: Copy Other Emitter to Get Emitter State</figcaption>
</figure>

When setup your copied module, be carefull to your default value .

For example, `Populate Neighbor Grid` has a Neighbor Grid 3D input, its default value is different from your existing Neighbor Grid 3D in your Niagara System.

<figure style="width: 721px" class="align-center">
<img src="/assets/images/ue5_niagara_sph/wrong_default_neighborgrid.png">
<figcaption align = "center">Fig: Default Neighbor Grid 3D input is different from existing Neighbor Grid 3D</figcaption>
</figure>

<figure style="width: 454px" class="align-center">
<img src="/assets/images/ue5_niagara_sph/right_value_in_module.png">
<figcaption align = "center">Fig: Right Input value should be your existing Neighbor Grid 3D</figcaption>
</figure>

### Setup Parameters and Simulaton Stage

Next we should add some parameters. Here is an overview of Niagara System and you will what is new.

<figure style="width: 591px" class="align-center">
<img src="/assets/images/ue5_niagara_sph/overview_1.png">
<figcaption align = "center">Fig: Overview 1</figcaption>
</figure>

<figure style="width: 269px" class="align-center">
<img src="/assets/images/ue5_niagara_sph/overview_2.png">
<figcaption align = "center">Fig: Overview 2</figcaption>
</figure>

Parameter value:

|Module|Parameter Name|Value|
|:-:|:-:|:-:|
|Set (SYSTEM) WorldGridExtent|WorldGridExtent|(6,6,6)|
|Initialize Neighbor Grid|MaxNeighborsPerCell|20|
|Initialize Neighbor Grid|NumCellsX|25|
|Initialize Neighbor Grid|NumCellsY|25|
|Initialize Neighbor Grid|NumCellsZ|25|
|Emitter Spawn|RestDensity|1|
|Emitter Spawn|PointMass|0.05|
|Emitter Spawn|GasConstantK|150|
|Emitter Spawn|Viscosity|10|
|Emitter Spawn|Gravity|(0,0,-9.8)|
|Spawn Rate|SpawnRate|200|
|Initialize Particle|Lifetime|50|
|Initialize Particle|Uniform Sprite Size|0.1|
|Set (PARTICLES) Density|Density|0|
|Set (PARTICLES) Pressure|Pressure|0|
|Set (PARTICLES) Velocity|Velocity|(0,10,0)|
|Populate Neighbor Grid|NeighborGrid|Your Neighbor Grid|
|Populate Neighbor Grid|Simulation To Grid Unit Transform|WorldToGridUnit|
|Solve Density and Pressure|Emitter Name|SPHEmitter1|
|Solve Force|Emitter Name|SPHEmitter1|
|Advent|MaxAccel|10|
|Advent|MaxVelocity|30|
|Advent|WallDamping|-0.1|

These values are set based on experience. They are only suitable for reference but not necessarily the most appropriate.

## Solve Density and Pressure

### Module View

<figure style="width: 1017px" class="align-center">
<img src="/assets/images/ue5_niagara_sph/solve_pressure_input.png">
<figcaption align = "center">Fig: Solve Pressure Input</figcaption>
</figure>

<figure style="width: 490px" class="align-center">
<img src="/assets/images/ue5_niagara_sph/solve_pressure_output.png">
<figcaption align = "center">Fig: Solve Pressure Output</figcaption>
</figure>

```hlsl
Density = 0;
Pressure = 0;

#if GPU_SIMULATION
const int3 IndexOffsets [ 27 ] = 
{
    int3(-1,-1,-1),
    int3(-1,-1, 0),
    int3(-1,-1, 1),
    int3(-1, 0,-1),
    int3(-1, 0, 0),
    int3(-1, 0, 1),
    int3(-1, 1,-1),
    int3(-1, 1, 0),
    int3(-1, 1, 1),

    int3(0,-1,-1),
    int3(0,-1, 0),
    int3(0,-1, 1),
    int3(0, 0,-1),
    int3(0, 0, 0),
    int3(0, 0, 1),
    int3(0, 1,-1),
    int3(0, 1, 0),
    int3(0, 1, 1),

    int3(1,-1,-1),
    int3(1,-1, 0),
    int3(1,-1, 1),
    int3(1, 0,-1),
    int3(1, 0, 0),
    int3(1, 0, 1),
    int3(1, 1,-1),
    int3(1, 1, 0),
    int3(1, 1, 1),
};

// Derive the Neighbor Grid Index from the world position
float3 UnitPos;
NeighborGrid.SimulationToUnit(Position, SimulationToUnit, UnitPos);

int3 Index;
NeighborGrid.UnitToIndex(UnitPos, Index.x,Index.y,Index.z);

int3 NumCells;
NeighborGrid.GetNumCells(NumCells.x, NumCells.y, NumCells.z);

// loop over all neighbors in this cell
int MaxNeighborsPerCell;
NeighborGrid.MaxNeighborsPerCell(MaxNeighborsPerCell);

float3 CellSize = WorldGridExtent / NumCells;

float DesitySum = 0;
float SmoothRadius = CellSize.x;  // In UE, the length unit is cm, but now just see it as m
float h2 = SmoothRadius * SmoothRadius;

float KernelPoly6 = 315.0 / (64.0 * 3.141592 * pow(SmoothRadius, 9));  // 1/m^9

float temp_max = 0;
for(int xxx = 0; xxx < 27; ++xxx)
{
    for(int i = 0; i < MaxNeighborsPerCell; i++)
    {
        const int3 IndexToUse = Index + IndexOffsets[xxx];
        
        int NeighborLinearIndex;
        NeighborGrid.NeighborGridIndexToLinear(IndexToUse.x, IndexToUse.y, IndexToUse.z, i, NeighborLinearIndex);
        
        int CurrNeighborIdx;
        NeighborGrid.GetParticleNeighbor(NeighborLinearIndex, CurrNeighborIdx);
        
        // temp bool used to catch valid/invalid results for direct reads
        bool myBool; 
        float3 OtherPos;
        DirectReads.GetVectorByIndex<Attribute="Position">(CurrNeighborIdx, myBool, OtherPos);

        const float3 vectorFromOtherToSelf = Position - OtherPos;
        const float r = length(vectorFromOtherToSelf);  // In UE, the length unit is cm, but now just see it as m
        const float h2_r2 = h2 - r * r;

        if(IndexToUse.x >= 0 && IndexToUse.x < NumCells.x &&
           IndexToUse.y >= 0 && IndexToUse.y < NumCells.y &&
           IndexToUse.z >= 0 && IndexToUse.z < NumCells.z &&
           CurrNeighborIdx != -1 && r < SmoothRadius)
        {
            if(SelfParticleIndex == CurrNeighborIdx)
            {
                DesitySum += pow(h2, 3);  // m^3
            }
            else
            {
                DesitySum += pow(h2_r2, 3);  // m^3
            }
        }
    }
}

Density = KernelPoly6 * PointMass * DesitySum;  // kg/m^3
Pressure = max((Density - RestDensity) * GasConstantK, 0);  // N/m^2
#endif
```

### Problem: NeighborGrid.GetCellSize return 0

This method has an error, that `NeighborGrid.GetCellSize` always return 0.

I have post a thread in UE forum,

[UE5 Niagara How does Neighbor Grid 3D GetCellSize working? It always return 0](https://forums.unrealengine.com/t/ue5-niagara-how-does-neighbor-grid-3d-getcellsize-working-it-always-return-0/1564078)

### Solution

but now just quickly fix it by passing extra value.

Solution is to add `WorldGridExtent` input, and calcuate cellsize by myself.

```hlsl
float3 CellSize = WorldGridExtent / NumCells;
```

### Problem: pow(-0.01, 3) return NaN

After that, I found power of negative number may return NaN, such as `pow(-0.01, 3)` return NaN.

<figure style="width: 529px" class="align-center">
<img src="/assets/images/ue5_niagara_sph/pow_return_NaN.png">
<figcaption align = "center">Fig: pow(-0.01, 3) return NaN</figcaption>
</figure>

### Solution

So solution is add a condition statement `r < SmoothRadius`, to avoid power of negative number.

## Solve Force

### Module View

<figure style="width: 1001px" class="align-center">
<img src="/assets/images/ue5_niagara_sph/solve_force_input.png">
<figcaption align = "center">Fig: Solve Force Input</figcaption>
</figure>

<figure style="width: 376px" class="align-center">
<img src="/assets/images/ue5_niagara_sph/solve_force_output.png">
<figcaption align = "center">Fig: Solve Force Output</figcaption>
</figure>

```hlsl
OutAcceleration = float3(0,0,0);

#if GPU_SIMULATION
const int3 IndexOffsets [ 27 ] = 
{
    int3(-1,-1,-1),
    int3(-1,-1, 0),
    int3(-1,-1, 1),
    int3(-1, 0,-1),
    int3(-1, 0, 0),
    int3(-1, 0, 1),
    int3(-1, 1,-1),
    int3(-1, 1, 0),
    int3(-1, 1, 1),

    int3(0,-1,-1),
    int3(0,-1, 0),
    int3(0,-1, 1),
    int3(0, 0,-1),
    int3(0, 0, 0),
    int3(0, 0, 1),
    int3(0, 1,-1),
    int3(0, 1, 0),
    int3(0, 1, 1),

    int3(1,-1,-1),
    int3(1,-1, 0),
    int3(1,-1, 1),
    int3(1, 0,-1),
    int3(1, 0, 0),
    int3(1, 0, 1),
    int3(1, 1,-1),
    int3(1, 1, 0),
    int3(1, 1, 1),
};

// Derive the Neighbor Grid Index from the world position
float3 UnitPos;
NeighborGrid.SimulationToUnit(Position, SimulationToUnit, UnitPos);

int3 Index;
NeighborGrid.UnitToIndex(UnitPos, Index.x,Index.y,Index.z);

int3 NumCells;
NeighborGrid.GetNumCells(NumCells.x, NumCells.y, NumCells.z);

// loop over all neighbors in this cell
int MaxNeighborsPerCell;
NeighborGrid.MaxNeighborsPerCell(MaxNeighborsPerCell);

float3 CellSize = WorldGridExtent / NumCells;  // cm

float SmoothRadius = CellSize.x;  // In UE, the length unit is cm, but now just see it as m
float h2 = SmoothRadius * SmoothRadius;  // m^2

float KernelSpiky = -45.0 / (3.141592 * pow(SmoothRadius, 6));  // 1/m^6
float KernelViscosity = 45.0 / (3.141592 * pow(SmoothRadius, 6));  // 1/m^6

for(int xxx = 0; xxx < 27; ++xxx)
{
    for(int i = 0; i < MaxNeighborsPerCell; i++)
    {
        const int3 IndexToUse = Index + IndexOffsets[xxx];
        
        int NeighborLinearIndex;
        NeighborGrid.NeighborGridIndexToLinear(IndexToUse.x, IndexToUse.y, IndexToUse.z, i, NeighborLinearIndex);
        
        int CurrNeighborIdx;
        NeighborGrid.GetParticleNeighbor(NeighborLinearIndex, CurrNeighborIdx);
        
        // avoid r divided by 0
        if (SelfParticleIndex == CurrNeighborIdx)
        {
            continue;
        }

        // temp bool used to catch valid/invalid results for direct reads
        bool myBool; 
        float3 OtherPos;
        DirectReads.GetVectorByIndex<Attribute="Position">(CurrNeighborIdx, myBool, OtherPos);
        float OtherDensity;
        DirectReads.GetFloatByIndex<Attribute="Density">(CurrNeighborIdx, myBool, OtherDensity);
        float OtherPressure;
        DirectReads.GetFloatByIndex<Attribute="Pressure">(CurrNeighborIdx, myBool, OtherPressure);
        float3 OtherVel;
        DirectReads.GetVectorByIndex<Attribute="Velocity">(CurrNeighborIdx, myBool, OtherVel);

        const float3 vectorFromOtherToSelf = Position - OtherPos;
        const float3 dirFromOtherToSelf = normalize(vectorFromOtherToSelf);  // unit dir
        const float r = length(vectorFromOtherToSelf);  // In UE, the length unit is cm, but now just see it as m

        if(IndexToUse.x >= 0 && IndexToUse.x < NumCells.x &&
           IndexToUse.y >= 0 && IndexToUse.y < NumCells.y &&
           IndexToUse.z >= 0 && IndexToUse.z < NumCells.z &&
           CurrNeighborIdx != -1 && r < SmoothRadius)
        {
            const float h_r = SmoothRadius - r; // m
            const float h2_r2 = h2 - r * r; // m^2

            // F_Pressure
            const float3 pterm = -PointMass * KernelSpiky * h_r * h_r * (Pressure + OtherPressure) / (2.f * Density * OtherDensity) * dirFromOtherToSelf;

            // F_Viscosity
            const float3 vterm = PointMass * KernelViscosity * Viscosity * h_r * (OtherVel - Velocity) / (Density * OtherDensity);

            OutAcceleration += pterm + vterm;  // m/s^2
        }
    }
}
#endif
```

## Advent

### Module View

<figure style="width: 580px" class="align-center">
<img src="/assets/images/ue5_niagara_sph/advent_input.png">
<figcaption align = "center">Fig: Advent Input</figcaption>
</figure>

<figure style="width: 635px" class="align-center">
<img src="/assets/images/ue5_niagara_sph/advent_output.png">
<figcaption align = "center">Fig: Advent Output</figcaption>
</figure>

```hlsl
#pragma once

OutVelocity = float3(0.0, 0.0, 0.0);
OutPosition = float3(0.0, 0.0, 0.0);

#if GPU_SIMULATION

Acceleration += Gravity;  // m/s^2

// acceleration limit
if(length(Acceleration) > MaxAccel)
{
	Acceleration = normalize(Acceleration) * MaxAccel;  // m/s^2
}

// advent
Velocity += 100 * Acceleration * DeltaTime;  // cm/s

// velocity limit
if(length(Velocity) > MaxVelocity)
{
	Velocity = normalize(Velocity) * MaxVelocity;  // cm/s
}

// advent
Position += Velocity * DeltaTime;  // cm

// wall avoidance
if(Position.x < -WorldGridExtent.x/2)
{
    Velocity.x *= WallDamping;
    Position.x = -WorldGridExtent.x/2;
}
if(Position.x > WorldGridExtent.x/2)
{
    Velocity.x *= WallDamping;
    Position.x = WorldGridExtent.x/2;
}
if(Position.y < -WorldGridExtent.y/2)
{
    Velocity.y *= WallDamping;
    Position.y = -WorldGridExtent.y/2;
}
if(Position.y > WorldGridExtent.y/2)
{
    Velocity.y *= WallDamping;
    Position.y = WorldGridExtent.y/2;
}
if(Position.z < -WorldGridExtent.z/2)
{
    Velocity.z *= WallDamping;
    Position.z = -WorldGridExtent.z/2;
}
if(Position.z > WorldGridExtent.z/2)
{
    Velocity.z *= WallDamping;
    Position.z = WorldGridExtent.z/2;
}

OutPosition = Position;  // cm
OutVelocity = Velocity;  // cm

#endif // GPU_SIMULATION
```

### Problem: Oscillation may be too violent

You may find your particles have violent oscillation and never become idle, which is unexpected.

### Solution

Simulation method has common problem about stability, a direct solution is to reduce your timestep.

```hlsl
DeltaTime = DeltaTime / 5;
```

However, it will make your particle effect very slow, it is unbearable for gamer.

Or you can increase viscoity, reduce acceleration limit and velocity limit.

## Current Effect

Current effect is as below, the particles should be stacked multiple layer, and have little oscillation.

<figure style="width: 345px" class="align-center">
<img src="/assets/images/ue5_niagara_sph/effect_1.gif">
<figcaption align = "center">Fig: Current Effect</figcaption>
</figure>

### Problem: Effect doesn't show up in level

If you drag your Niagara System to your level and then there is nothing show up.

### Solution

If particles is too small, then, maybe because of LOD or some built-in engine behaviour, you can't see it in the level. To make particles visible, you should change `Sprite Size`.

<figure style="width: 536px" class="align-center">
<img src="/assets/images/ue5_niagara_sph/too_small_to_see.png">
<figcaption align = "center">Fig: Too small to see particles</figcaption>
</figure>

<figure style="width: 273px" class="align-center">
<img src="/assets/images/ue5_niagara_sph/change_sprite_size.png">
<figcaption align = "center">Fig: Change Sprite Size</figcaption>
</figure>

## Render Setup

Traditional method of rendering water surface is marching cube method. It generates density field from particles, then extract polygon mesh isosurface. It can be done on GPU, but very expensive.

Another way is to project all all particles into screen space. A direct idea is using depth buffer, but I think that depth buffer can only tells where the particles are. However, later I learned that normal can be calculated from depth buffer. 

### Another Neighbor Grid 3D Used for Renderring in Screen Space

At first, project all particles into screen space. More specifically, you create an empty depth buffer, then project each particle's world position into screen space position, and draw a circle with given radius, in the depth buffer, with the particle's depth. For sure, small depth value will override large depth value.

However, if you draw circle into depth buffer, you may encounter memory access conflict becuase you are running kernel function on each particles. Then each thread may need to write same pixel of depth buffer at the same time. You should take strategy to solve the conflict, such as mutex lock. It definitly hurts parallel performance.

Think in the opposite way, after creating an empty depth buffer, for each pixels in screen space, find particles projected in the screen space position of pixel. To do this, create a new Neighbor Grid 3D, which is used for accelerating searching particles. From the view of occupying space, the Neighbor Grid 3D should occupys full space of screen, so one cell of Neighbor Grid 3D may overlaps with many pixels.

Now kernel function on each pixels may read particles attributes at the same time, but there is not conflict problem.

<figure style="width: 600px" class="align-center">
<img src="/assets/images/ue5_niagara_sph/sph_render.svg">
<figcaption align = "center">Fig: Render each pixels</figcaption>
</figure>

### Use Grid 2D to Store Depth Value

Here Grid 2D is used to stored depth and then write it into Render Target.

To initialize Grid 2D, you can use bulit-in module `Grid 2D Set Resolution`. But if can't find it after clicking plus symbol on the right side of `Emitter Spawn`, the reason may be you should uncheck `Library Only`.

[Grid2d collection no cellnums setting shown in ue5 niagara](https://forums.unrealengine.com/t/grid2d-collection-no-cellnums-setting-shown-in-ue5-niagara/531971/4)

Similarly, you can use built-in module `Neighbor Grid 3D Set Resolution` to initialize your Render Neighbor Grid 3D.

You can also set half float format for Grid 2D.

<figure style="width: 587px" class="align-center">
<img src="/assets/images/ue5_niagara_sph/set_grid_2d_half_float.png">
<figcaption align = "center">Fig: Set Half Float Format for Grid 2D</figcaption>
</figure>

### Texture Render Target and Render Target 2D

You should be familiar with Render Target file in Content Browser, but you may don't know how to link it with Niagara System.

First you should create a Texture Render Target with `User` namespace.

Be caution, it is not Render Target 2D. For example, you can see I create a Render Target 2D named `RT_Depth` and a Texture Render Target named `TextureRenderTarget_Depth`.

<figure style="width: 510px" class="align-center">
<img src="/assets/images/ue5_niagara_sph/RT_2D_and_Texture_RT.png">
<figcaption align = "center">Fig: Texture Render Target and Render Target 2D</figcaption>
</figure>

Then crate a Render Target 2D also named `RT_Depth` with `Emitter` namespace, drag it into `Emitter Spawn` to initialize it.

Then you will see, in property `Render Target User Parameter`, there is only one choice `TextureRenderTarget_Depth`. It means that you can only assign Texture Render Target as Render Target User Parameter of Render Target 2D.

<figure style="width: 1079px" class="align-center">
<img src="/assets/images/ue5_niagara_sph/set_Texture_RT_to_RT_2D.png">
<figcaption align = "center">Fig: Set Texture Render Target as Render Target User Parameter of Render Target 2D</figcaption>
</figure>

After setting, you validate whether the Render Target is associate with Niagara System immediately. To do this, create a new simulation stage, set `Data Interface` as Render Target 2D. Create a new module, enter the module, add Render Target 2D in Map Get, drag from Render Target 2D input and search `Set Render Target Value` in the popup. Create `Execution Index` block, link to `Linear To Index`, link `IndexX, IndexY` to `Set Render Target Value`. Create `Make Linear Color` and link to `Set Render Target Value`. For example, choose a purple color, open your Render Target file from Content Browser, then you will see your Render Target change corresponding to your `Make Linear Color` block.

### Overview

<figure style="width: 250px" class="align-center">
<img src="/assets/images/ue5_niagara_sph/overview_3.png">
<figcaption align = "center">Fig: Overview 3</figcaption>
</figure>

<figure style="width: 266px" class="align-center">
<img src="/assets/images/ue5_niagara_sph/overview_4.png">
<figcaption align = "center">Fig: Overview 4</figcaption>
</figure>

## Calc Depth And Write Neighbor

### Module View

First module is `CalcuateParticleDepth`.

<figure style="width: 1171px" class="align-center">
<img src="/assets/images/ue5_niagara_sph/CalcuateParticleDepth_1.png">
<figcaption align = "center">Fig: CalcuateParticleDepth Module View 1</figcaption>
</figure>

<figure style="width: 1105px" class="align-center">
<img src="/assets/images/ue5_niagara_sph/CalcuateParticleDepth_2.png">
<figcaption align = "center">Fig: CalcuateParticleDepth Module View 2</figcaption>
</figure>

The hlsl is borrow from `World Position to Screen UV` block. This is about projecting particle's position into screen space.

```hlsl
ScreenUV = float2(0,0);

#if GPU_SIMULATION

float4 SamplePosition = float4(In_SamplePos, 1);
float4 ClipPosition = mul(SamplePosition, View.WorldToClip);
float2 ScreenPosition = ClipPosition.xy / ClipPosition.w;

ScreenUV = float2(ScreenPosition.x, ScreenPosition.y*-1)/2 + (0.5, 0.5);

#endif
```

Calcuating depth is simple, in short, it is vector multiples with forward.

<figure style="width: 600px" class="align-center">
<img src="/assets/images/ue5_niagara_sph/sph_particle_depth.svg">
<figcaption align = "center">Fig: Particle Depth</figcaption>
</figure>

You should also check whether the particle is behind camera, if so, then discard.

The second module is `WriteRenderNeighbor`. This is about storing particles into Neighbor Grid 3D according to particle's projected position.


<figure style="width: 788px" class="align-center">
<img src="/assets/images/ue5_niagara_sph/WriteRenderNeighbor_1.png">
<figcaption align = "center">Fig: WriteRenderNeighbor Module View 1</figcaption>
</figure>

<figure style="width: 677px" class="align-center">
<img src="/assets/images/ue5_niagara_sph/WriteRenderNeighbor_2.png">
<figcaption align = "center">Fig: WriteRenderNeighbor Module View 2</figcaption>
</figure>

```hlsl
#if GPU_SIMULATION

int3 Index;
NeighborGrid.UnitToIndex(float3(UnitPos, 0), Index.x,Index.y,Index.z);

int3 NumCells;
NeighborGrid.GetNumCells(NumCells.x, NumCells.y, NumCells.z);

int MaxNeighborsPerCell;
NeighborGrid.MaxNeighborsPerCell(MaxNeighborsPerCell);
		
if (Index.x >= 0 && Index.x < NumCells.x && 
    Index.y >= 0 && Index.y < NumCells.y)
{
    int LinearIndex;
    NeighborGrid.IndexToLinear(Index.x, Index.y, 0, LinearIndex);

    int PreviousNeighborCount;
    NeighborGrid.SetParticleNeighborCount(LinearIndex, 1, PreviousNeighborCount);

    if (PreviousNeighborCount < MaxNeighborsPerCell)
    {
        int NeighborGridLinear;
        NeighborGrid.NeighborGridIndexToLinear(Index.x, Index.y, Index.z, PreviousNeighborCount, NeighborGridLinear);

        int IGNORE;
        NeighborGrid.SetParticleNeighbor(NeighborGridLinear, InstanceIdx, IGNORE);
    }		
}

#endif
```

### Problem: Capturing result of camera query always located at world origin

Niagara System capturing result of camera query always located at world origin. That means, only when player look at world origin can player see it, no matter where the Niagara System instance is.

For example, in this gif, Niagara System is moving in the world, but capturing result doesn’t move in the Render Target.

<figure style="width: 622px" class="align-center">
<img src="/assets/images/ue5_niagara_sph/sph_moving_but_dont_move_in_render_target.gif">
<figcaption align = "center">Fig: Capturing result of camera query always located at world origin</figcaption>
</figure>

### Solution

I don’t know it is a bug or not. Maybe it is because I enable `Local Space` in emitter property. Anyway, it is confusing.

However, simple hack is adding Niagara System position to particle’s position when you are capturing particles.

<figure style="width: 860px" class="align-center">
<img src="/assets/images/ue5_niagara_sph/simple_hack.png">
<figcaption align = "center">Fig: Simple hack to ensure camera captures right particle position</figcaption>
</figure>

I have post a discussion in UE forum, maybe someone knows why.

[Niagara System capturing result of camera query always located at world origin](https://forums.unrealengine.com/t/niagara-system-capturing-result-of-camera-query-always-located-at-world-origin/1582261)

But only add a Niagara System position is not enough. The result position reflected on the Render Target is still not correct.

So after trying, the correct position is particles' position + Niagara System position / 2.

<figure style="width: 848px" class="align-center">
<img src="/assets/images/ue5_niagara_sph/correct_pos_hack.png">
<figcaption align = "center">Fig: Correct position hack</figcaption>
</figure>

Although I haven’t said it here yet, if you have set Render Target as Material Input, and link it with opacity mask, you will see:

<figure style="width: 662px" class="align-center">
<img src="/assets/images/ue5_niagara_sph/render_mask.gif">
<figcaption align = "center">Fig: Correct Position in Render Target</figcaption>
</figure>

To observe this, scale of plane should be a litter smaller than it should be.

## Draw Depth To Grid 2D

Set `Iteration Source` as `Data Interface`, and drag `RenderGrid2D` to `Data Interface`, then in the module, if you set namespace of a value in map out as `STACKCONTEXT`, the value will be stored into `Data Interface` automatically.

Now, if you want `for loop` pixels, logically you first traverse Grid 2D, stored your desired value into cells of Grid 2D. After travering finished, you write value from Grid 2D to a render target.

<figure style="width: 600px" class="align-center">
<img src="/assets/images/ue5_niagara_sph/sph_is_particle_overlap_with_cell.svg">
<figcaption align = "center">Fig: Is particle overlap with cell?</figcaption>
</figure>

If you found your Render Target shows cube-like particle but not sphere, try to adjust `RenderSpriteSize` from small value to large one.

### Module View

<figure style="width: 987px" class="align-center">
<img src="/assets/images/ue5_niagara_sph/draw_depth_input.png">
<figcaption align = "center">Fig: Draw Depth To Grid 2D Input</figcaption>
</figure>

<figure style="width: 631px" class="align-center">
<img src="/assets/images/ue5_niagara_sph/draw_depth_output.png">
<figcaption align = "center">Fig: Draw Depth To Grid 2D Output</figcaption>
</figure>

```hlsl
const int2 IndexOffsets [ 9 ] = 
{
    int2(-1,-1),
    int2(-1, 0),
    int2(-1, 1),
    int2(0,-1),
    int2(0, 0),
    int2(0, 1),
    int2(1,-1),
    int2(1, 0),
    int2(1, 1)
};

OutDepth = 0;

#if GPU_SIMULATION

int3 Index;
NeighborGrid.UnitToIndex(float3(CellPos, 0), Index.x,Index.y,Index.z);

int3 NumCells;
NeighborGrid.GetNumCells(NumCells.x, NumCells.y, NumCells.z);

int MaxNeighborsPerCell;
NeighborGrid.MaxNeighborsPerCell(MaxNeighborsPerCell);

float minDepth = 3.402823466e+38;

for(int xxx = 0; xxx < 9; ++xxx)
{
    for(int i = 0; i < MaxNeighborsPerCell; i++)
    {
        const int3 IndexToUse = int3(Index.x + IndexOffsets[xxx].x, Index.y + IndexOffsets[xxx].y, 0);
        
        int NeighborLinearIndex;
        NeighborGrid.NeighborGridIndexToLinear(IndexToUse.x, IndexToUse.y, IndexToUse.z, i, NeighborLinearIndex);
        
        int CurrNeighborIdx;
        NeighborGrid.GetParticleNeighbor(NeighborLinearIndex, CurrNeighborIdx);

        bool myBool; 
        float2 ProjectedParticlePos;
        ParticleReader.GetVector2DByIndex<Attribute="ProjectedPos">(CurrNeighborIdx, myBool, ProjectedParticlePos);

        float ParticleDepth;
        ParticleReader.GetFloatByIndex<Attribute="Depth">(CurrNeighborIdx, myBool, ParticleDepth);
        
        float DistanceFromCellToParticle = length(ProjectedParticlePos - CellPos);
        float ProjectedRenderSpriteSize = RenderSpriteSize/ParticleDepth;

        if(IndexToUse.x >= 0 && IndexToUse.x < NumCells.x &&
           IndexToUse.y >= 0 && IndexToUse.y < NumCells.y &&
           CurrNeighborIdx != -1 &&
           DistanceFromCellToParticle < ProjectedRenderSpriteSize &&
           ParticleDepth != 0)
        {
            minDepth = minDepth < ParticleDepth ? minDepth : ParticleDepth;
        }
    }
}
if(minDepth < 1e+38)
{
    OutDepth = minDepth;
}
#endif
```

There is still concern about we shouldn't stored particle depth directly, because it is depth at particle cell. What we need is actually depth at cell center.

<figure style="width: 600px" class="align-center">
<img src="/assets/images/ue5_niagara_sph/sph_actual_depth.svg">
<figcaption align = "center">Fig: Particle actual depth</figcaption>
</figure>

```hlsl
const int2 IndexOffsets [ 9 ] = 
{
    int2(-1,-1),
    int2(-1, 0),
    int2(-1, 1),
    int2(0,-1),
    int2(0, 0),
    int2(0, 1),
    int2(1,-1),
    int2(1, 0),
    int2(1, 1)
};

OutDepth = 0;

#if GPU_SIMULATION

int3 Index;
NeighborGrid.UnitToIndex(float3(CellPos, 0), Index.x,Index.y,Index.z);

int3 NumCells;
NeighborGrid.GetNumCells(NumCells.x, NumCells.y, NumCells.z);

int MaxNeighborsPerCell;
NeighborGrid.MaxNeighborsPerCell(MaxNeighborsPerCell);

float minDepth = 3.402823466e+38;
int minDepthIndex = -1;

for(int xxx = 0; xxx < 9; ++xxx)
{
    for(int i = 0; i < MaxNeighborsPerCell; i++)
    {
        const int3 IndexToUse = int3(Index.x + IndexOffsets[xxx].x, Index.y + IndexOffsets[xxx].y, 0);
        
        int NeighborLinearIndex;
        NeighborGrid.NeighborGridIndexToLinear(IndexToUse.x, IndexToUse.y, IndexToUse.z, i, NeighborLinearIndex);
        
        int CurrNeighborIdx;
        NeighborGrid.GetParticleNeighbor(NeighborLinearIndex, CurrNeighborIdx);

        bool myBool; 
        float2 ProjectedParticlePos;
        ParticleReader.GetVector2DByIndex<Attribute="ProjectedPos">(CurrNeighborIdx, myBool, ProjectedParticlePos);

        float ParticleDepth;
        ParticleReader.GetFloatByIndex<Attribute="Depth">(CurrNeighborIdx, myBool, ParticleDepth);
        
        float DistanceFromCellToParticle = length(ProjectedParticlePos - CellPos);
        float ProjectedRenderSpriteSize = RenderSpriteSize/ParticleDepth;

        if(IndexToUse.x >= 0 && IndexToUse.x < NumCells.x &&
           IndexToUse.y >= 0 && IndexToUse.y < NumCells.y &&
           CurrNeighborIdx != -1 &&
           DistanceFromCellToParticle < ProjectedRenderSpriteSize &&
           ParticleDepth != 0)
        {
            minDepth = minDepth < ParticleDepth ? minDepth : ParticleDepth;
            minDepthIndex = minDepth < ParticleDepth ? minDepthIndex : CurrNeighborIdx;
        }
    }
}

if(minDepthIndex != -1)
{
    bool myBool; 
    float2 ProjectedParticlePos;
    ParticleReader.GetVector2DByIndex<Attribute="ProjectedPos">(minDepthIndex, myBool, ProjectedParticlePos);

    float DistanceFromCellToParticle = length(ProjectedParticlePos - CellPos);
    float ProjectedRenderSpriteSize = RenderSpriteSize/minDepth;

    OutDepth = minDepth + ProjectedRenderSpriteSize - sqrt(ProjectedRenderSpriteSize * ProjectedRenderSpriteSize - DistanceFromCellToParticle * DistanceFromCellToParticle);
}
#endif
```

But I think there is little difference between them.

### Problem: Performance is N/A

Niagara is compiled successfully, but the running time of a certain simulation stage is N/A.

<figure style="width: 887px" class="align-center">
<img src="/assets/images/ue5_niagara_sph/performance_NA.png">
<figcaption align = "center">Fig: Performance is N/A</figcaption>
</figure>

I have post a question in UE forum.

[UE5 Niagara Compilation succeeded but certain simulation stage’s performance time is N/A](https://forums.unrealengine.com/t/ue5-niagara-compilation-succeeded-but-certain-simulation-stages-performance-time-is-n-a/1575522)

### Solution

After one day of work, I found the error disappear surprisingly.

From the time I encounter the problem to the time I found the problem disappear, I have done many step, most of them are irrelevant about Grid 2D. The only relevant step is adding a built-in `Grid 2D Set Resolution` module. But if I delete the module, the performance logging time is still work well. So it is a magic and I don't want to pay attention to this issue anymore.

## Bilateral Filtering

The module does Bilateral Filtering on depth map stored in Grid 2D. Blur will make your depth map smooth, which will make your normal map look less like particles. Gauss Filterign will blur the whole image, but Bilateral Filtering will preserves sharp edges.

### Module View

<figure style="width: 736px" class="align-center">
<img src="/assets/images/ue5_niagara_sph/Bilateral_Filtering_input.png">
<figcaption align = "center">Fig: Bilateral Filtering Input</figcaption>
</figure>

<figure style="width: 752px" class="align-center">
<img src="/assets/images/ue5_niagara_sph/Bilateral_Filtering_output.png">
<figcaption align = "center">Fig: Bilateral Filtering Output</figcaption>
</figure>

```hlsl
OutDepth = 0;

#if GPU_SIMULATION

int2 NumCells;
Grid2D.GetNumCells(NumCells.x, NumCells.y);

float weightSum = 0;
float depthSum = 0;

float CurrDepth;
Grid2D.GetFloatValue<Attribute="Depth">(IndexX, IndexY, CurrDepth);

if(CurrDepth > 0)
{
    for(int x = -BlurRadius; x <= BlurRadius; x++)
    {
        for(int y = -BlurRadius; y <= BlurRadius; y++)
        {
            float spaceWeight = exp(-(x*x + y*y)*BlurScale*BlurScale);

            float NeighborDepth;
            Grid2D.GetFloatValue<Attribute="Depth">(IndexX + x, IndexY + y, NeighborDepth);
            float r2 = (NeighborDepth - CurrDepth)*BlurDepthFalloff;
            float depthWeight = exp(-r2*r2);

            if(NeighborDepth > 0)
            {
                weightSum += spaceWeight * depthWeight;
                depthSum += NeighborDepth * spaceWeight * depthWeight;
            }
        }
    }
}

if(weightSum > 0){
    OutDepth = depthSum / weightSum;
}
#endif
```

Toggle Blur module, you should see the difference.

<figure style="width: 342px" class="align-center">
<img src="/assets/images/ue5_niagara_sph/disable_blur.png">
<figcaption align = "center">Fig: Disable Blur</figcaption>
</figure>

<figure style="width: 330px" class="align-center">
<img src="/assets/images/ue5_niagara_sph/enable_blur.png">
<figcaption align = "center">Fig: Enable Blur</figcaption>
</figure>

However, blur parameter also matters. Inappropriate parameters are equivalent to you not blurring, and your normal map will end up looking like particles, such as the image above.

In the last I will show a more natural result, where parameters setting is:

|Name|Value|
|:-:|:-:|
|BlurRadius|5.0|
|RenderSpriteSize|3.0|
|BlurDepthFalloff|1.0|
|BlurScale|0.1|

## Draw Depth To Render Target

Now `Data Interface` can choose Render Target 2D or Grid 2D.

### Module View

<figure style="width: 1029px" class="align-center">
<img src="/assets/images/ue5_niagara_sph/DrawDepthRenderTarget_input.png">
<figcaption align = "center">Fig: Draw Depth Render Target Input</figcaption>
</figure>

<figure style="width: 968px" class="align-center">
<img src="/assets/images/ue5_niagara_sph/DrawDepthRenderTarget_middle.png">
<figcaption align = "center">Fig: Draw Depth Render Target Middle Part</figcaption>
</figure>

<figure style="width: 669px" class="align-center">
<img src="/assets/images/ue5_niagara_sph/DrawDepthRenderTarget_output.png">
<figcaption align = "center">Fig: Draw Depth Render Target Output</figcaption>
</figure>

### Problem: Only one line in Render Target is white

After drawing to render target, there should be a beautiful depth map in Render Target, but I only see one line in Render Target is white.

<figure style="width: 1011px" class="align-center">
<img src="/assets/images/ue5_niagara_sph/only_one_line_is_white.png">
<figcaption align = "center">Fig: Only One Line in Render Target is White</figcaption>
</figure>

Log the proejcted position stored on particles, and it seems correct, because all of them are in (0,1) and there are not strange values.

<figure style="width: 373px" class="align-center">
<img src="/assets/images/ue5_niagara_sph/projected_position_is_correct.png">
<figcaption align = "center">Fig: Projected Position is Correct</figcaption>
</figure>

Depth values stored on particles are also checked. They are located in (10, 100) roughly, and it is reasonable. My WorldGridExtent is (6,6,6), so distance of camera to particles should be of the same magnitude as grid length, if camera want to see the particles clearly.

Then another reason may be calcuation of index is wrong. Error may located in `Write Depth to Grid 2D` module or `Write Depth to Render Target` module.

Firstly I validate `Write Depth to Render Target` module quickly. I add a test module before `Write Depth to Render Target` module, add output `x*y` as `Depth`.

<figure style="width: 249px" class="align-center">
<img src="/assets/images/ue5_niagara_sph/quick_validate_write_depth_to_render_target_1.png">
<figcaption align = "center">Fig: Quickly Validate Write Depth to Render Target 1</figcaption>
</figure>

<figure style="width: 1240px" class="align-center">
<img src="/assets/images/ue5_niagara_sph/quick_validate_write_depth_to_render_target_2.png">
<figcaption align = "center">Fig: Quickly Validate Write Depth to Render Target 2</figcaption>
</figure>

<figure style="width: 511px" class="align-center">
<img src="/assets/images/ue5_niagara_sph/quick_validate_write_depth_to_render_target_3.png">
<figcaption align = "center">Fig: Quickly Validate Write Depth to Render Target 3</figcaption>
</figure>

You can see Render Target perfectly displayed `x*y` result, so `Write Depth to Render Target` module has no error. So problem is in `Write Depth to Grid 2D` module.

In `Write Depth to Grid 2D` module, type same testing code in the end of custom hlsl, and you will get same result on Render Target.

```hlsl
OutDepth = (float)Index.x/(float)NumCells.x * (float)Index.y/(float)NumCells.y;
```

Use `CellPos` to get same result.

```hlsl
OutDepth = CellPos.x * CellPos.y;
```

At least it shows that grid index and `CellPos` is right.

After I add condition statement between `IndexToUse` and `NumCells`, the flashing white line disappear, only pure black left.

```hlsl
if(IndexToUse.x >= 0 && IndexToUse.x < NumCells.x &&
    IndexToUse.y >= 0 && IndexToUse.y < NumCells.y &&
    CurrNeighborIdx != -1 &&
    DistanceFromCellToParticle < ProjectedRenderSpriteSize &&
    ParticleDepth != 0)
{
    minDepth = minDepth < ParticleDepth ? minDepth : ParticleDepth;
}
```

Then I realize that problem may not lies in Grid 2D but in Neighbor Grid 3D. 

In fact, if you set `-CurrNeighborIdx` to depth then you will see pure white in Render Target, it means you never find a valid neighbor.

```hlsl
minDepth = -CurrNeighborIdx;
```

I guess particles may not be stored into Neighbor Grid 3D, so I go back to `Write Render Neighbor` module.

```hlsl
#if GPU_SIMULATION

int3 Index;
NeighborGrid.UnitToIndex(float3(UnitPos, 0), Index.x,Index.y,Index.z);

...

OutIndexX = Index.x;
OutIndexY = Index.y;
OutIndexZ = Index.z;
#endif
```

<figure style="width: 496px" class="align-center">
<img src="/assets/images/ue5_niagara_sph/unit_to_index_is_right.png">
<figcaption align = "center">Fig: Unit to Index is right</figcaption>
</figure>

So next I have tried other debug method, but still don't know what is wrong.

### Solution

The finally solution is pretty simple: I should drag Neighbor Grid 3D to Emitter Spawn Group to initialize it.

<figure style="width: 267px" class="align-center">
<img src="/assets/images/ue5_niagara_sph/have_to_drag_neighbor_grid_3d_to_emitter_spawn.png">
<figcaption align = "center">Fig: Have to drag Neighbor Grid 3D to Emitter Spawn Group</figcaption>
</figure>

It is confusing me bacause if I use the Neighbor Grid 3D in later module, there is no error.

Anyway, leave it behind.

### Problem: Unexpected Render Target Flickering

Right after fix the problem of initializing Neighbor Grid 3D, my Render Target finally have right depth map but unexpected flickering occurs.

The flickering in the recorded gif is not due to image compression.

<figure style="width: 688px" class="align-center">
<img src="/assets/images/ue5_niagara_sph/flashing_render_target.gif">
<figcaption align = "center">Fig: Unexpected Flickering</figcaption>
</figure>

<figure style="width: 688px" class="align-center">
<img src="/assets/images/ue5_niagara_sph/flashing_render_target_stop_at_one_frame.png">
<figcaption align = "center">Fig: Render Target Stop at One Frame</figcaption>
</figure>

I have searched UE forum:

[Niagara particles flickering in latest .26 release](https://forums.unrealengine.com/t/niagara-particles-flickering-in-latest-26-release/153007/1)

Their solutions: 

1. Reduce spawn rate 

2. Reduce particle emitter sphere radius 

3. Enlarge `Fixed Bounds` in `Emitter Properties`

4. Change to `CPU Simualtion`

5. Set the `Sort Mode` to `View Depth` in Sprite Renderer

    It mey be problem about translucent sorting

6. Enable `Camera Distance Culling` in Sprite Renderer, adjust max camera distance

7. Change the mesh size much smaller

However, in this case, it is not about translucent particles, because it is not about capturing particles actually. The camera capture only capture particles' depth. Then simulation stage writes the depth into Grid 2D. So my situation may be totally different from other.

### Solution

Finally I don't want to waste my time on the engine bug, so I give up trying.

But just after an hour I come back to work, the problem disapper amazingly.

Nothing to say with the kind of situtation.

Updated: After a few days, I happen to find the flickering reason: Render Target is switching between scene capturing result and Niagara System capturing result. You can see it clearly in my record.

<figure style="width: 955px" class="align-center">
<img src="/assets/images/ue5_niagara_sph/flickering_reason.gif">
<figcaption align = "center">Fig: Flickering Reason</figcaption>
</figure>

So it is engine bug, not my fault.

### Problem: Render Target has content in Niagara System Preview but no in the level

After I found Render Target doesn't flicker, I put the Niagara System into level. Becuase my WorldGridExtent is very small, which is (6, 6, 6), so I had to scale the Niagara System to (100, 100, 100) size. I had talked before sprite size should also be large, but now we don't need sprite to show where the particles are, we only need a plane to show fluid renderring result, so we can disable sprite renderer in the SPH emitter. But as I continued to make the fluid material, I found that Render Target has content in Niagara System Preview but no in the level.

<figure style="width: 955px" class="align-center">
<img src="/assets/images/ue5_niagara_sph/no_content_in_the_level.png">
<figcaption align = "center">Fig: Render Target has content in Niagara System Preview but no in the level</figcaption>
</figure>

### Solution

It recalls me that small particles is invisible, so I guess that even though we scale Niagara System to make particles visible, in the camera view, it doesn't apply scale transform to particles in the level. Camera is always capturing particles of original size, so particles move in small space. While small size particles is invisible, or I guess it is also related with small spacing. Combining these two factors, which makes camera can't capture particles in the level.

To solve it, the only way is to change WorldGridExtent and adjust SPH parameters to adapt with new extent. This is a torturous thing, so after I obtained a seemingly acceptable result in the small extent, I did not adjust the parameters in the large extent. Now, it seems like an unavoidable task.

If only want to validate the camera problem, we can leave the quality of visual effect behind. Practice shows that the assumption is right. Make WorldGridExtent larger, then particles move in a larger space, then camera can capture them effectively.

<figure style="width: 716px" class="align-center">
<img src="/assets/images/ue5_niagara_sph/have_to_make_niagara_system_large.png">
<figcaption align = "center">Fig: Have to make WorldGridExtent Large</figcaption>
</figure>

New parameter:

|Module|Parameter Name|Value|
|:-:|:-:|:-:|
|Set (SYSTEM) WorldGridExtent|WorldGridExtent|(200,200,200)|
|Emitter Spawn|RestDensity|-50|
|Emitter Spawn|PointMass|10|
|Emitter Spawn|GasConstantK|100|
|Emitter Spawn|Viscosity|25|
|Advent|MaxAccel|200|
|Advent|MaxVelocity|100|
|Advent|WallDamping|-0.8|

After this parameter adjustment, I found a way to quickly obtain the appropriate parameters: first increase the pressure, such as increasing the gas parameters, and then increase the viscosity. Finally, you will get an effect that looks a bit like a fluid. Although this cannot avoid the so-called jelly water, at least you must first be able to obtain a stable effect before you can start optimizing it.

### Problem: Move Niagara System but particles are blocked at border

<figure style="width: 221px" class="align-center">
<img src="/assets/images/ue5_niagara_sph/sph_blocked_at_border.gif">
<figcaption align = "center">Fig: Move Niagara System but particles are blocked at border</figcaption>
</figure>

### Solution

Enable `Local Space` in Emitter property, then all particles' position are calcuated in local space. In other word, the particles' coordinate follows actor but not world coordinate.

## New An Emitter to Emit Plane for Renderring

You should create a plane to show the renderring result. It can be created in Niagara System or in world level. The former looks nice because it hides details into Niagara System itself.

To do this, new an emitter to emit a plane sprite. The sprite lifetime should be long enough, so it can be set as `LoopDuration`

<figure style="width: 368px" class="align-center">
<img src="/assets/images/ue5_niagara_sph/lifetime_is_loop_duration.png">
<figcaption align = "center">Fig: Lifetime is LoopDuration</figcaption>
</figure>

The plane sprite should be located between camera and Niagara System, and should be large enough to cover the whold Niagara System.

<figure style="width: 354px" class="align-center">
<img src="/assets/images/ue5_niagara_sph/calc_plane_pos.png">
<figcaption align = "center">Fig: Calculated Plane Position</figcaption>
</figure>

<figure style="width: 254px" class="align-center">
<img src="/assets/images/ue5_niagara_sph/large_scale_of_plane.png">
<figcaption align = "center">Fig: Large scale of Plane</figcaption>
</figure>

There is room to decide specific distance from camera to plane. Here I choose 100.

<figure style="width: 1132px" class="align-center">
<img src="/assets/images/ue5_niagara_sph/distance_100.png">
<figcaption align = "center">Fig: Distance is 100</figcaption>
</figure>

### Overview

<figure style="width: 793px" class="align-center">
<img src="/assets/images/ue5_niagara_sph/overview_5.png">
<figcaption align = "center">Fig: Overview 5</figcaption>
</figure>

### Problem: Emitter turns into balck in the Niagara System Overview and Sprite doesn't show up

Sometimes, my plane emitter turns into balck in the Niagara System Overview and plane sprite doesn't show up.

<figure style="width: 212px" class="align-center">
<img src="/assets/images/ue5_niagara_sph/emitter_turns_balck.png">
<figcaption align = "center">Fig: turns into balck in the Niagara System Overview and Sprite doesn't show up</figcaption>
</figure>

### Solution

It may be UE bug, restart UE project will solve the problem.

## Create Fluid Material

Next you should add a material to your plane.

<figure style="width: 351px" class="align-center">
<img src="/assets/images/ue5_niagara_sph/set_material_to_plane.png">
<figcaption align = "center">Fig: Set Material to Plane</figcaption>
</figure>

This material should have a `TextureSampleParameter2D` paramter, to get Render Target.

Create an instance of the material, then you can assign Render Target to the instance parameter.

<figure style="width: 426px" class="align-center">
<img src="/assets/images/ue5_niagara_sph/assign_render_target_to_texture_input.png">
<figcaption align = "center">Fig: Assgin Render Target to Texture Input</figcaption>
</figure>

### Get Normal from Depth Map

There is method to get normal from depth map, but I haven't pay time to understand it.

[Screen Space Fluid Rendering for Games - Nvidia](https://developer.download.nvidia.com/presentations/2010/gdc/Direct3D_Effects.pdf)

Unreal has built-in `Normal From Height Map` block, but it only receives `Texture 2D` object. So I copy the content from it and replace the `Texture 2D` part with `Input` parameter.

<figure style="width: 639px" class="align-center">
<img src="/assets/images/ue5_niagara_sph/built_in_normal_from_height_map.png">
<figcaption align = "center">Fig: Built-in Normal From Height Map Block</figcaption>
</figure>

<p align="center">
<iframe src="https://blueprintue.com/render/y2jbegzh/" height="600" width="900" scrolling="no" allowfullscreen></iframe>
</p>

Becuase I store depth to RGB of Render Target, so when you take depth from Render Target, you should choose the same channel.

### Get Opacity Mask from Depth Map

To mask space without fluid, sample from Render Target and link it directly to Opacity Mask.

<p align="center">
<iframe src="https://blueprintue.com/render/_aluyoo5/" height="600" width="900" scrolling="no" allowfullscreen></iframe>
</p>

But if you want other objects to cover fluid, you should compare `SceneDepthWithoutWater` with fluid depth. If `SceneDepthWithoutWater` is smaller than fluid depth, then other objects can cover fluid.

### Single Layer Water Material

To use `SceneDepthWithoutWater` block, you should set your `Shading Model` to `Single Layer Water`.

<figure style="width: 320px" class="align-center">
<img src="/assets/images/ue5_niagara_sph/switch_to_single_layer_water.png">
<figcaption align = "center">Fig: Switch to Single Layer Water</figcaption>
</figure>

To compile the material, you should have `Single Layer Water Material` block.

<figure style="width: 778px" class="align-center">
<img src="/assets/images/ue5_niagara_sph/single_layer_water_material.png">
<figcaption align = "center">Fig: Single Layer Water Material</figcaption>
</figure>

#### Problem: ScatteringCoefficients Node Link in Single Layer Water Material breaks alpha blending

I have fed an occlusion-tested depth into the alpha. The specific method of the occlusion test is, water depth is compared with `Single Layer Water Material` block, output depth is `0  * water depth = 0` when water is covered by other objects, `1 * water depth = water depth` when nothing to block.

If I link a input value into `ScatteringCoefficients` node in `Single Layer Water Material` block, then alpha blending will be break.

<figure style="width: 528px" class="align-center">
<img src="/assets/images/ue5_niagara_sph/break_alpha_blending.png">
<figcaption align = "center">Fig: ScatteringCoefficients Node Link in Single Layer Water Material breaks alpha blending</figcaption>
</figure>

If I break `ScatteringCoefficients` node link, then alpha blending will be correct.

<figure style="width: 497px" class="align-center">
<img src="/assets/images/ue5_niagara_sph/correct_alpha_blending.png">
<figcaption align = "center">Fig: Correct alpha blending</figcaption>
</figure>

I have post a thread in UE forum waiting the answer.

[ScatteringCoefficients Node Link in Single Layer Water Material breaks alpha blending](https://forums.unrealengine.com/t/scatteringcoefficients-node-link-in-single-layer-water-material-breaks-alpha-blending/1586830)

#### Solution(?)

Don't use `ScatteringCoefficients` node.

#### Problem: Water border shows up ignoring occlusion

In the image above, you will see water border shows up ignoring occlusion，it is because depth map in Render Target appears weird blur margin.

<figure style="width: 315px" class="align-center">
<img src="/assets/images/ue5_niagara_sph/blur_marigin_of_render_target.png">
<figcaption align = "center">Fig: Blur Margin of Render Target</figcaption>
</figure>

Current stage I don't have a solution to deal with this. Maybe a material hack can takes effect.

## Final Result

Although it doesn’t look very good, at least it has normals that look reasonable. How to make it look better after that is a matter for the shader.

<figure style="width: 213px" class="align-center">
<img src="/assets/images/ue5_niagara_sph/final_sph_effect.gif">
<figcaption align = "center">Fig: Final Result</figcaption>
</figure>

<script src="https://utteranc.es/client.js"
        repo="CheapMeow/cheapmeow.github.io"
        issue-term="pathname"
        theme="github-light"
        crossorigin="anonymous"
        async>
</script>

