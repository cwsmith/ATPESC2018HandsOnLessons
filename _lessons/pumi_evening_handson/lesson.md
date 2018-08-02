---
layout: page-fullwidth
title: "Adaptive MFEM+PUMI Workflow"
subheadline: "Conformal Unstructured Mesh Adaptation for Advancing Shock"
teaser: "MFEM with PUMI Conformal Unstructured Mesh Adaptation"
permalink: "lessons/pumi_evening_handson/"
use_math: true
lesson: true
header:
 image_fullwidth: "xsdk_logo_wide.png"
---

## My Questions

- how do i create links from text to sections or figures?

## At A Glance

|Questions|Objectives|Key Points|
| How can I define a simulation on a complex geometric model?        | Demonstrate model defeaturing and mesh generation pre-processing tools           | Complex models require robust pre-processing tools!                                                                                                                                   | 
|                                                                    | Demonstrate the power of geometric model classification                          | Automated simulation tools designed for scientists and engineers working with real CAD models problem definition information must be specified on the geometric model - not the mesh. | 
| How can I efficiently execute simulations with transient behavior? | Demonstrate an adaptive MFEM+PUMI linear elastic analysis with large deformation | Adaptation is critical to automated, robust, and efficient simulation of simulations with transient behavior in which a static mesh defined a-priori will fail.                       | 

Before you begin, first [Open the Answers Form](https://goo.gl/forms/HmuX6HrT0Yfoz7ny2){:target="\_blank"}
in a separate browser tab/window.
*Answer Question 0* once you have it open.
We will be entering responses to questions here that are placed throughout the
lesson.

The MFEM+PUMI examples are prepared for execution on Cooley at ALCF.
Except where noted, all commands shown below should be executed in a terminal
logged into Cooley.
In each new terminal first prepare your environment by running the commands
listed in the following section.

Note, each step can be performed independently; we encourage you to go
through them in order.

## To begin this lesson

- Ensure that your environment is only using the `@default` packages by running the following commands:
```
cp ~/.soft ~/.soft_bk
echo '@default' > ~/.soft
resoft
mv ~/.soft_bk ~/.soft
```

- Add mvapich2 to your environment
```
soft add +mvapich2-2.3rc1
```

- Copy the example directory to your home directory
```
rsync -aP {{site.handson_install_root}}/mfem-pumi-lesson ~/.
```

- Enter the lesson directory
```
cd ~/mfem-pumi-lesson
```


## Geometric Model Defeaturing

Geometric models are often provided by design engineers and include many
features that are not required for simulation.
Figure 1 depicts the geometric model of a tokamak fusion reactor that has
multiple bolts, nuts, and brackets that have no influence on the frequency
analysis (FIXME).  These features are removed from the Parasolid CAD model
by the Simmetrix SimModSuite tools by calling Parasolid kernel APIs to delete
the faces and 'heal' the remaining hole via extensions of the bounding edges and
faces.  Figure 2 depicts the defeatured tokamak model. In a model with this
level of complexity that engineers and scientists running the simulation often
must coordinate to create the `as-simulated' model.

![Figure 1](mfem-superlu0000.png){:width="500"}
*Figure 1. Initial tokamak geometric model*

![Figure 2](mfem-superlu0000.png){:width="500"}
*Figure 2. Defeatured tokamak geometric model*

For this hands-on we will perform a few simple defeaturing operations on the RPI
Formula Hybrid suspension upright shown in Figure 3.
The ids of the geometric model faces of interest are labelled.
Faces ## ## ## are small faces created during the design that are not relevant
to the simulation and can be removed.
To remove those faces run the following commands.

```
qsub -I -n 1 -t 30 -A ATPESC2018 -q training
cd ~/mfem-pumi-lesson/defeature
./defeature upright.x_t ## ## ## upright_defeatured.x_t
```

![labelled upright](mfem-superlu0000.png){:width="500"}
*Figure 3. RPI Formula Hybrid suspension upright with faces labelled*

In the Mesh Generation step we will generate and compare meshes of the initial
and defeatured geometric models.

## Problem Definition

We will define a tensile loading on the upright by applying a uniform force on
one end (min Z face) and fixing the displacements on the other end (max Z face).
All other geometric model faces will be unconstrained.  No body forces are
applied.

Using geometric classification of the mesh we can define the boundary conditions
on the geometric model without having any knowledge/consideration of the mesh.
A very simple text based interface was defined for this demonstration example to
define the two boundary conditions.  To specify the geometric model faces with
the load applied we add one line with `Load` followed by another line with the
number of faces with that load applied.  The geometric model face ids, one per
line, are then listed. Using the same convention we define the fixed faces using
the string `Dirichlet`, the number of listed faces, then the list of face ids.
The boundary condition specification for this example is listed below.
Using the geometric model classification mechanism supports creation of more
feature-rich interfaces (command line, file, GUI).

```
$ cat upright.def
Dirichlet
1
43

Load
1
344
```

*Answer Question 1*

## Mesh Generation

Now that we have prepared the model and defined the boundary conditions we can
proceed with mesh generation using the Simmetrix SimModSuite library APIs.
The mesh generation procedures are driven by size controls (e.g., the absolute
length of mesh edges) defined by the user on geometric model entities or defined
within geometric primatives that intersect the model (e.g., cubes, spheres,
cylinders, etc.).
The mesh generator is automated, which in the domain of mesh generation means
that it will always produce a valid mesh while doing its best to respect the
user specified controls.
Note, the features of the geometric model highly dictate what mesh sizes are
possible.
For example, in a thin cross-section isotropic  mesh elements (those with equal
lengthed edges) cannot be larger than the cross-section thickness.
If the user requests anisotropic, and/or higher order (curved), mesh elements
than that thickness constraint would change.

Once the mesh is generated we convert it, in-memory (i.e., using PUMI and
SimModSuite APIs), to a PUMI mesh and then write the PUMI mesh to file.

*Answer Question 2*

```
qsub -I -n 1 -t 30 -A ATPESC2018 -q training
cd ~/mfem-pumi-lesson/meshGeneration
# generate the mesh on the initial geometric model and create paraview vtu files
mpirun -np 4 ./generate --native-model=upright.x_t upright.smd 5kg1
mpirun -np 4 ./render upright.x_t 5kg1 5kg1_initial/
# generate the mesh on the defeatured model and create paraview vtu files
mpirun -np 4 ./generate --native-model=upright_defeatured.x_t upright_defeatured.smd 5kg1
mpirun -np 4 ./render upright_defeatured.x_t 5kg1 5kg1_defeatured/
```

![initial model](mfem-superlu0000.png){:width="500"}
*Figure 4. Mesh of initial upright model*

![defeatured model](mfem-superlu0000.png){:width="500"}
*Figure 5. Mesh of defeatured upright model*

### Optional - Visualize the Initial Meshes
Download the pvtu files from [Here - FIXME](https://goo.gl/forms/HmuX6HrT0Yfoz7ny2), or
use a file transfer utility (e.g., `scp`, `rsync`, `putty`, etc.) to copy them to your local machine.

## Partitioning

Efficient parallel execution requires equally distributing the units of work among the
processing resources while keeping communication costs low.
The multi-level graph and recursive sectioning geometric methods are among the
most commonly available and used methods for partitioning unstructured meshes.
In this lesson we will exercise the multi-level graph partitioner provided by
the Zoltan interface and implemented by ParMETIS.  We will then run recursive
coordinate bisection (RCB) and recursive inertial bisection (RIB) to produce
meshes of with the same part count (number of sub-domains), and compare the
results.  All three partitioners are targeting an imbalance of 5%; defined as
the max element count on any part divided by the average element count across
all parts.

```
qsub -I -n 1 -t 30 -A ATPESC2018 -q training
cd ~/mfem-pumi-lesson/partition
mpirun -np 4 ./ptnParma upright_defeatured.x_t 5k1g_.smb 5k1g_p4_parmetis/ 4 parmetis kway 0  # rib
mpirun -np 4 ./ptnParma upright_defeatured.x_t 5k1g_.smb 5k1g_p4_rcb/ 4 rcb ptn 0  # rcb
mpirun -np 4 ./ptnParma upright_defeatured.x_t 5k1g_.smb 5k1g_p4_rib/ 4 rib ptn 0  # rib
```

Examine the output and Answer Question 3.

Examine Figures 6-8 below and Answer Question 4.

![rib](mfem-superlu0000.png){:width="500"}
*Figure 6. RIB partition*
![rcb](mfem-superlu0000.png){:width="500"}
*Figure 7. RCB partition*
![multi-level](mfem-superlu0000.png){:width="500"}
*Figure 8. ParMETIS multi-level graph partition*

### Optional - Visualize the Partitioned Meshes
Download the pvtu files from [Here]( https://goo.gl/forms/HmuX6HrT0Yfoz7ny2), or
use a file transfer utility (e.g., `scp`, `rsync`, `putty`, etc.) to copy them to your local machine.

## Adaptive Simulation



## Out-Brief

Here, re-emphasize the lesson objectives and key points.

Its fine to go into greater detail about questions or objectives this lesson
did not fully cover.

### Further Reading

Include links to other online sources you might want to include.
