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
---

It is an implementation note of SPH using UE5 Niagara System, related about usage of Simulation Stage, Grid 3D, renderring using depth buffer and so on.

## SPH Introduction

Basic knowledge about SPH can see other tutorial.

## Implementation using Engine 5 Niagara System

### Niagara System Setup

Basic knowledge about Niagara System can see other tutorial.

To setup, change `Sim Target` to `GPUCompute Sim`, because our Niagara System may calculate millions of particles.

Change `Calculate Bounds Mode` to `Fixed`, because particles are upload to GPU so it is hard to calculate bounds every frame.

Change `Life Cycle Mode` to `Self`, in order to set `Loop Behaviour` as `Infinite`.

### Neighbor Grid3D

#### Add Neighbor Grid3D from empty

One data structure we should use is `Neighbor Grid3D`, we will store our particle indice into grid, for convience of searching neighbor particles in 8 grids around certain particle.

In Niagara System window, find `Parameter` window (not `Details` windows), find `Emitter Attributes`, click "+" button, search `neighbor` in popup, then you will find `Neighbor Grid3D` item, confirm.

You know the adding parameters of Niagara System is similar to actor blueprint.

<figure>
<p align="center">
<img src="/assets/images/ue5_niagara_sph/neighbor_grid.png">
</p>
<figcaption align = "center">Fig: Neighbor Grid</figcaption>
</figure>

After creating a `Neighbor Grid3D` parameter, you should drag it into you emitter. Then `Neighbor Grid3D` will take effect in this emitter.

<figure>
<p align="center">
<img src="/assets/images/ue5_niagara_sph/after_drag_neighbor_grid_to_emitter.png">
</p>
<figcaption align = "center">Fig: After Drag Neighbor Grid to Emitter</figcaption>
</figure>

#### Using module script from content example

After you add `Neighbor Grid3D` variable, you should also define some variable about extent, transform matrix, etc. To skip these details at the first time, I copy two modules `Initialize Neighbor Grid` and `Populate Neighbor Grid` from offical Content Example.

<figure>
<p align="center">
<img src="/assets/images/ue5_niagara_sph/copy_modules.png">
</p>
<figcaption align = "center">Fig: Copy Modules</figcaption>
</figure>

However, after doing this, I add `Spawn Rate` and then click `Fix issue`, try to add `Emitter State`. But UE log "fail to add emitter state" becuase "could not find location".

I don't know why, so I create a emitter from existing copy, and delete redundant module from it. So now I get no error.

<figure>
<p align="center">
<img src="/assets/images/ue5_niagara_sph/copy_other_emitter_to_get_emitter_state.png">
</p>
<figcaption align = "center">Fig: Copy Other Emitter to Get Emitter State</figcaption>
</figure>

### Solve Density and Pressure

#### Setup variable

You can setup your emitter according to this figure. You know what variable to add.

<figure>
<p align="center">
<img src="/assets/images/ue5_niagara_sph/solving_density_setup.png">
</p>
<figcaption align = "center">Fig: Solve Density and Pressure Setup</figcaption>
</figure>

#### HLSL

<figure>
<p align="center">
<img src="/assets/images/ue5_niagara_sph/solving_density_input.png">
</p>
<figcaption align = "center">Fig: Solve Density and Pressure Input</figcaption>
</figure>

<figure>
<p align="center">
<img src="/assets/images/ue5_niagara_sph/solving_density_output.png">
</p>
<figcaption align = "center">Fig: Solve Density and Pressure Output</figcaption>
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

float3 CellSize;
NeighborGrid.GetCellSize(CellSize);

float DesitySum = 0;
float SmoothRadius = CellSize.x / 100;
float h2 = SmoothRadius * SmoothRadius;

float KernelPoly6 = 315.0 / (64.0 * 3.141592 * pow(SmoothRadius, 9));;

for(int xxx = 0; xxx < 27; ++xxx)
{
    for(int i = 0; i < MaxNeighborsPerCell; i++)
    {
        const int3 IndexToUse = Index + IndexOffsets[xxx];
        
        int NeighborLinearIndex;
        NeighborGrid.NeighborGridIndexToLinear(IndexToUse.x, IndexToUse.y, IndexToUse.z, i, NeighborLinearIndex);
        
        int CurrNeighborIdx;
        NeighborGrid.GetParticleNeighbor(NeighborLinearIndex, CurrNeighborIdx);

        if(IndexToUse.x >= 0 && IndexToUse.x < NumCells.x &&
           IndexToUse.y >= 0 && IndexToUse.y < NumCells.y &&
           IndexToUse.z >= 0 && IndexToUse.z < NumCells.z &&
           CurrNeighborIdx != -1)
        {
            if(SelfParticleIndex == CurrNeighborIdx)
            {
                DesitySum += pow(h2, 3); // self
            }
            else
            {
                // temp bool used to catch valid/invalid results for direct reads
                bool myBool; 
                float3 OtherPos;
                DirectReads.GetVectorByIndex<Attribute="Position">(CurrNeighborIdx, myBool, OtherPos);

                const float3 vectorFromOtherToSelf = Position - OtherPos;
                const float r = length(vectorFromOtherToSelf) / 100;
                const float h2_r2 = h2 - r * r;

                DesitySum += pow(h2_r2, 3); // (h^2-r^2)^3
            }
        }
    }
}
// kg/m^3
Density = KernelPoly6 * PointMass * DesitySum;
Pressure = (Density - RestDensity) * GasConstantK;
#endif
```

I use the method firstly, but I found my 

#### HLSL v2

<figure>
<p align="center">
<img src="/assets/images/ue5_niagara_sph/solving_density_input_2.png">
</p>
<figcaption align = "center">Fig: Solve Density and Pressure Input 2</figcaption>
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
float SmoothRadius = CellSize.x / 100;
float h2 = SmoothRadius * SmoothRadius;

float KernelPoly6 = 315.0 / (64.0 * 3.141592 * pow(SmoothRadius, 9));;

for(int xxx = 0; xxx < 27; ++xxx)
{
    for(int i = 0; i < MaxNeighborsPerCell; i++)
    {
        const int3 IndexToUse = Index + IndexOffsets[xxx];
        
        int NeighborLinearIndex;
        NeighborGrid.NeighborGridIndexToLinear(IndexToUse.x, IndexToUse.y, IndexToUse.z, i, NeighborLinearIndex);
        
        int CurrNeighborIdx;
        NeighborGrid.GetParticleNeighbor(NeighborLinearIndex, CurrNeighborIdx);

        if(IndexToUse.x >= 0 && IndexToUse.x < NumCells.x &&
           IndexToUse.y >= 0 && IndexToUse.y < NumCells.y &&
           IndexToUse.z >= 0 && IndexToUse.z < NumCells.z &&
           CurrNeighborIdx != -1)
        {
            if(SelfParticleIndex == CurrNeighborIdx)
            {
                DesitySum += pow(h2, 3); // self
            }
            else
            {
                // temp bool used to catch valid/invalid results for direct reads
                bool myBool; 
                float3 OtherPos;
                DirectReads.GetVectorByIndex<Attribute="Position">(CurrNeighborIdx, myBool, OtherPos);

                const float3 vectorFromOtherToSelf = Position - OtherPos;
                const float r = length(vectorFromOtherToSelf) / 100;
                const float h2_r2 = h2 - r * r;

                DesitySum += pow(h2_r2, 3); // (h^2-r^2)^3
            }
        }
    }
}
// kg/m^3
Density = KernelPoly6 * PointMass * DesitySum;
Pressure = (Density - RestDensity) * GasConstantK;
#endif
```

### Solve Force

#### HLSL

<figure>
<p align="center">
<img src="/assets/images/ue5_niagara_sph/solve_force_input.png">
</p>
<figcaption align = "center">Fig: Solve Force Input</figcaption>
</figure>

<figure>
<p align="center">
<img src="/assets/images/ue5_niagara_sph/solve_force_output.png">
</p>
<figcaption align = "center">Fig: Solve Force Output</figcaption>
</figure>

```hlsl
Accel = float3(0,0,0);

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

float SmoothRadius = CellSize.x / 100;
float h2 = SmoothRadius * SmoothRadius;

float KernelSpiky = -45.0 / (3.141592 * pow(SmoothRadius, 6));
float KernelViscosity = 45.0 / (3.141592 * pow(SmoothRadius, 6));

for(int xxx = 0; xxx < 27; ++xxx)
{
    for(int i = 0; i < MaxNeighborsPerCell; i++)
    {
        const int3 IndexToUse = Index + IndexOffsets[xxx];
        
        int NeighborLinearIndex;
        NeighborGrid.NeighborGridIndexToLinear(IndexToUse.x, IndexToUse.y, IndexToUse.z, i, NeighborLinearIndex);
        
        int CurrNeighborIdx;
        NeighborGrid.GetParticleNeighbor(NeighborLinearIndex, CurrNeighborIdx);
        
        if(IndexToUse.x >= 0 && IndexToUse.x < NumCells.x &&
           IndexToUse.y >= 0 && IndexToUse.y < NumCells.y &&
           IndexToUse.z >= 0 && IndexToUse.z < NumCells.z &&
           CurrNeighborIdx != -1)
        {
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
            const float r = length(vectorFromOtherToSelf) / 100;
            const float h_r = SmoothRadius - r; // h-r
            const float h2_r2 = h2 - r * r; // h^2-r^2

            // F_Pressure
            const float3 pterm = -PointMass * KernelSpiky * h_r * h_r * (Pressure + OtherPressure) / (2.f * Density * OtherDensity) * vectorFromOtherToSelf / r;

            // F_Viscosity
            const float3 vterm = PointMass * KernelViscosity * Viscosity * h_r * (OtherVel - Velocity) / (Density * OtherDensity);

            Accel += pterm + vterm;
        }
    }
}
#endif
```