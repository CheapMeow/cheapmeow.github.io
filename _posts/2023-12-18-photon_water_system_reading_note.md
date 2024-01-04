---
title: 'GDC2023 Photon Water System Reading Note'
show_date: true
permalink: /posts/2023/12/photon_water_system_reading_note/
tags:
  - UE5
  - Water Simulation
  - Game Effect
  - GDC
toc: true
toc_sticky: true
author_profile: false
---

It is an reading note of Photon Water System in GDC2023, related about usage of flow map baking, shallow water equation solving for water surface propagation.

The article can't cover all details, it just a reading note. Implementation details are so complex that it is recommended to have a look on original source.

[GDCVault Photon Water System](https://www.gdcvault.com/search.php#&category=free&firstfocus=&keyword=photon+water%2Bsystem)

Photon Water System:

1. Procedural tools for river, lake, and ocean

2. Offline flow map baking

3. Runtime fluid simulation

4. Buoyancy and boat physics

5. Underwater volumetric lighting

6. Adaptive water mesh tessellation

## Fluid simulation

### Offline Flow map baking

Flowmap is pre-computed texture using physical-based simulation tool in existing third party software, such as Houdini.

Details about usage see:

[Water Flow in Portal 2](https://advances.realtimerendering.com/s2010/index.html)

The velocity field is stored as a 2d vector map and fetch at runtime.

However, the existing tool is still inconvenience for artists to use and cannot support turbulent flow.

LightSpeed use lattice Boltzmann method for the solution of shallow water equations (LBMSWE) to bake the flow map. The method has severval advantages:

1. Support turbulence flow

2. Simple to implement

3. Highly parallelizable

4. Conservative

Details about LBM see other tutorial.

#### Solving

The basic LBMSWE algorithm has four steps:

1. Update the equilibrium distribution $f_{\alpha}^{eq}$

2. Streaming and collision

3. Boundary handling

4. Compute macroscopic variables

In fact, after reading, I just review the theroy of LBM but have no idea about baking.

#### Summarize

To summarize, LBMSWE:

Pros:

1. Support turbulence flow turbulent flow

2. Simple to implement

3. Highly parallelizable

4. Conservative

Cons:

1. Large memory usage

2. Not good for real-time waterfront propagation

### Runtime fluid simulation

LightSpeed uses Shallow Water Equation (SWE) to implement height-field based 2.5D fluid simulation.

To solve classic NS equation based on the shallow water assumption:

$$
\dfrac{Dh}{Dt}+h\nabla \cdot u = 0
$$

$$
\dfrac{Du}{Dt} = -g \nabla (h+H)
$$

<figure class="align-center">
<img src="/assets/images/photon_water_system_reading_note/height_field.png">
<figcaption align = "center">Fig: Height Field</figcaption>
</figure>

They split world into grid. Water depth $h$ are stored at cell centers, velocities are stored at edge centers.

<figure class="align-center">
<img src="/assets/images/photon_water_system_reading_note/SWE_grid.png">
<figcaption align = "center">Fig: SWE Grid</figcaption>
</figure>

#### Solving

The solving is split into 3 steps:

1. Height integration $\frac{\partial{h}}{\partial{t}} = -h\nabla \cdot u$

2. Velocity advection $\frac{\partial{u}}{\partial{t}} = -u \cdot \nabla u$

3. Apply water pressure $\frac{\partial{u}}{\partial{t}} = -g \nabla (h+H)$

step 1 and step 3 are using discrete format of first order differential.

step 2 uses classic idea of Semi-Lagrangian Method, which see center of grid as particle, backtrace this particle position $p_{last}$ at last time, sample at $p_{last}$ and set it as field value at next time. 

The way to get $p_{last}$ is simply regarding velocity doesn't change over time, so $p_{last} = p_{curr} - v_{curr} \Delta t$

To summarize: $A_{new}(p_{curr}) = A_{curr}(p_{last})$

It is just my personal conclusion, you can see famous paper `Stable Fluid` to get more details.

[Stable Fluid](https://pages.cs.wisc.edu/~chaol/data/cs777/stam-stable_fluids.pdf)

And the paper author also gave details about implementation of Stable Fluid:

[Real-Time Fluid Dynamics for Games](http://graphics.cs.cmu.edu/nsp/course/15-464/Fall09/papers/StamFluidforGames.pdf)

#### SWE + Particles System

### Grid-based foam simulation

## Open world water rendering

### Rendering pipeline

### Surface wave

### Tessellation

<script src="https://utteranc.es/client.js"
        repo="CheapMeow/cheapmeow.github.io"
        issue-term="pathname"
        theme="github-light"
        crossorigin="anonymous"
        async>
</script>