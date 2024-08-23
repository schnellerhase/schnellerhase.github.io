---
title: "Google Summer of Code 2024 - Multigrid for FEniCSx"
date: 2024-08-30T12:00:00+02:00
draft: false
toc: true
mathjax: true
---

## Intro

This is the report on the Google Summer of Code 2024 project, entitled ['Multigrid for FEniCSx'](https://summerofcode.withgoogle.com/programs/2024/projects/YhEwl8iB), under the auspices of the [NumFOCUS](https://numfocus.org/) organization, on the [FEniCS Project](https://fenicsproject.org/), which is (since 2016) fiscally supported by NumFOCUS.

Under the supervision of [Jack S. Hale](https://www.jackhale.co.uk/) (Research Scientist, University of Luxembourg), [Chris Richardson](https://www.ieef.cam.ac.uk/user/cnr12) (Research Software Engineer, University of Cambridge) and [Joseph Dean](https://www.eng.cam.ac.uk/profiles/jpd62) (Research Scientist, University of Cambridge) the Google Summer of Code provided an opportunity for me to contribute to the FEniCS project and get to know open source software development efforts for high-performance computing applications.

The FEniCS project's primary component is the finite element solving environment, [DOLFINx](https://doi.org/10.5281/zenodo.10447666), on which the majority of the project's work was conducted.
The "x" indicates that this is a new version, which supersedes the former (now legacy) [DOLFIN](https://doi.org/10.11588/ans.2015.100.20553).
Although the objective of both the legacy and current projects is identical, the necessity to adapt to new requirements, incorporate new technological advances and, integrate insights gained from previous experiences has prompted the decision to rewrite and redesign the project in a new version.

## Geometric Multigrid and an Application to GSOC

Coming soon...

TODO: link proposal?

## Implementation decisions

As a first step in the research and getting up to speed phase, it was necessary to figure out what the FEniCS project wants to achieve with a geometric multigrid implementation.
Its interaction with other, already present and future, features and the overall design principles of the DOLFINx code base played a major role during this phase.
Let us shortly recap some of the findings and their consequences for the project.

One of the fundamental design choices of DOLFINx (also in contrast to its predecessor) is the data driven design.
Functionality should be provided on standard data containers holding the necessary information, without further encapsulation and object based data wrapping, while implicit expectations on data or objects are to be kept to a minimum.
As a main motivation for this the extensibility and usability outside the initial thought out scope of a given functionality, are to be named.

For the multigrid implementation this resulted, after careful consideration, in the decision to make the transfer operation itself an explicit function that needs to be provided explicitly for a certain use case.
This is a different path of choice, then what for example Firedrake (see [here](https://www.firedrakeproject.org/demos/geometric_multigrid.py.html)) and PETSc (see [here](https://petsc.org/release/manualpages/PC/PCGAMG/)) support with the very implicit construction of transfer operations under the hood, without the user needing to deal with any of it explicitly.
In addition the mesh hierarchies considered in this project should be produced by the available refinement routines (based on the work of [Plaza](https://www.oden.utexas.edu/media/reports/1996/9654.pdf)), and not be applicable to general meshes without clear containment.

Besides more explicitly for the user, the design should also, in principle, allow for higher performance, or at least greater control over it, as features are not employed if not chosen by the user.

TODO: big picture, after intro (loss of big picture: p-multigrid, domain decomposition)

## Interpolation Operators and Dual Operators

A first idea to approach the implementation of a geometric multigrid with the hope of an easy to attain proof of concept, came with the availability of the non-matching mesh interpolation routine of DOLFINx.
This routine implements a general interpolation operation between non matching meshes, and thus also non-matching, function spaces in general.
So we have already the implementation at hand to produce for two different finite element approximation spaces \( V*\text{coarse} \) and \( V*\text{fine} \) mappings

$$
P : V_\text{coarse} \to V_\text{fine}
\quad \text{and} \quad
R : V_\text{fine} \to V_\text{coarse}.
$$

Especially the map \( P \) is injective and \( R \) is surjective, which we expect from general restriction and prolongation operators.

So a first approach might be to make use of these mappings for a first geometric multigrid implementation and refine it after.
However a closer look shows an unfixable flaw in this setup.

Let us consider an example.

TODO: A nice example

The problem with the non matching interpolation routines as transfer operators arises from the fact that they are not adjoint maps of one another.
This breaks the properties of the multigrid operator - for example without duality we loose every mathematical ground of proving any properties of the multigrid operator matrix, such as for example the for convergence critical spectral radius.

TODO: story about adjointness of the operators vs adjointsnes of the transfer matrices.

## Interval Refinement in DOLFINx

So, the logical step for getting started with a geometric multigrid in the setting of mesh hierarchies produced by refinement routines is naturally the one dimensional case, i.e. interval meshes.
However DOLFINx at the time did not provide any such routine.
So as a first major step the implementation of a one dimensional refinement routine was addressed, and resulted in a (merged) [pull request](https://github.com/FEniCS/dolfinx/pull/3314).

Let us consider a small example to make clear what we need to do.
Given a mesh of three vertices (bold) and two cells (or equivalently edges)

**0** --- 0 --- **1** --- 1 --- **2** --- 2 --- **3**

we would like to perform a refinement on a set of marked edges \(M\).
Let us consider the uniform refinement case first, so we mark all edges for refinement, i.e. \( M=\{ 0, 1, 2 \} \).
While in higher dimensions mesh refinement offers a great number of applicable refinement schemes for refining an edge marked mesh, in one dimension the necessary operation is unique.
Every marked edge should be split, i.e. a new vertex in the center is to be introduced and the former edge split into two new finer/shorter ones.
For the former mesh we thus expect the fine mesh

**0** --- 0 --- **1** --- 1 --- **2** --- 2 --- **3** --- 3 --- **4** --- 4 --- **5** --- 5 --- **6**.

At a first sight, this seems like a very straight forward operation to implement, however a few caveats arising from parallel data structures with relabeling of indices for locality and the for parallel use cases inevitable repartitioning make this surprisingly evolved to implement.

TODO: mesh stored data and aspects of parallelization
TODO: algorithm of choice
TODO: output: parent cells

For the debugging of the code, especially in parallel, looking at visualizations of the pre- and post-refinement meshes was a crucial step.
This was easy to do thanks to the handy [febug](https://github.com/nate-sime/febug) tool, making visualization of different mesh entities straight forward and simple.

### A Story on Assertions in Compiled Python Modules

After discovering an initial bug in the merged code, resulting in an obscure crash of a python test, we discovered a problem of the interaction with assertions and the python runtime.
The error was caused by the code hitting, correctly due to a wrongly designed test case, an assertion in the C++ part of the software.
However the `pytest` framework, which is used for unit and integration tests in the FEniCS project, did not pick up on this and only showed a inconclusive core dump.
This turned out to be a bigger and not trivially fixable problem.
If the C++ part, i.e. in a call of the compiled python module, hits an C-style assertion the runtime itself is killed, as this triggers a call to ´abort´.
Especially the handling of the assertion is not returned to the outer python context which has systems in place to recover from this.
As `abort` produces a `SIGABRT` signal that forcefully ends the program there is no easy way of fixing this interaction.
However if the python module (so the C++ code) is build in a `Debug` mode we would like to be able to debug from the python side as well, which is then no longer possible.
This needs to be investigated further and remains an [open issue](https://github.com/FEniCS/dolfinx/issues/3333).

### Unifying the Refinement Interface

After the one dimensional refinement routine was merged, in a second endeavor we wanted to merge its interface with the previously available two and three dimensional Plaza refinement routines.
This resulted in a lot of changes and simplifications to the interface.
It became more explicit and easier to use.

One technical change that was introduced to facilitate this was the use of `std::optional`'s for input and output parameters.
Previously these were not used and a 'no-set' case was always handled separately leading to multiple different code paths.
Additionally this dispatching needed to be present in the python module as well.
Nanobind (the python exporter used in the DOLFINx project) however supports the `std::optionals` matching from the python optional `None` or value style for this.
So after changing we no longer need to handle different parameter combinations in the python module, but that all happens in C++ module.

## The Transfer Matrix

Coming soon...

- parallelization and ghost nodes
- graph partitioning

## Geometric Twogrid Example

Coming soon...

## General Code Contributions

While the main project focused on the implementation of a geometric multigrid a lot of incidental and supportive changes where contributed to different repositories as well.
Split across multiple repositories, we want to shortly outline the highlights of these in chronological order.

### DOLFINx - Introduce aliasing for mdspan [(Closed)](https://github.com/FEniCS/dolfinx/pull/3116)

The `mdspan` library in use with the FEniCS Project is based on the Kokkos [refrence implementation](https://github.com/kokkos/mdspan).
To facilitate the namespace switching depending on compiler support of either `std::experimental::mdspan` or `std::mdspan` or none of the above the implementation introduces the macro `MDSPAN_IMPL_STANDARD_NAMESPACE`.
This causes bloated calls, i.e. every time it is necessary to write out `MDSPAN_IMPL_STANDARD_NAMESPACE::mdspan`.
In the pull request this was suggested to be wrapped into a custom hiding layer, i.e. allowing for access via `dolfinx::common::mdspan`, no matter to what this dispatches to.
This would also allow the code to remain untouched in the future if one updates to the STL versions.
After a discussion the changes were rejected, due to it being considered an approximation of a third-party library.

### Basix - Update mdspan [(Merged)](https://github.com/FEniCS/basix/pull/840)

Update to new Kokkos single header `mdspan` changes.

### DOLFINx - Make compile time options compile time constants [(Merged)](https://github.com/FEniCS/dolfinx/pull/3246)

Previously in translation units wrapped transformation of compiler flag was moved to header files in combination with the usage of `consteval` to make them truly compile time constants.
The idiom of the changes was for a given (`cmake`) flag `FLAG` is demonstrated below.

```cpp
consteval bool has_flag()
{
#ifdef FLAG
  return true;
#else
  return false;
#endif
}
```

### DOLFINx - Some updates for mesh generation [(Open)](https://github.com/FEniCS/dolfinx/pull/3275)

Whilst reading up the mesh related data structures in DOLFINx for the project, I notices quite a few sub-optimal code paths in the generation code of meshes.
In this PR these were addressed, mostly consisting of modernizations, simplifications and some additional testing.
The modernizations also made use of the previously (in the FEniCS Project) unused [ranges library](https://en.cppreference.com/w/cpp/ranges), this was the trigger for looking into the following 12 pull requests.

### Basix - Update to std::ranges usage [(Merged)](https://github.com/FEniCS/basix/pull/837)

Made use of all `C++20` available ranges algorithms in the Basix code base.

### Basix - C++23 Support [(Reverted)](https://github.com/FEniCS/basix/pull/838)

Some important parts of the ranges library were missed in `C++20`, and are only available from `C++23` onward.
So to check general compatibility of the FEniCS packages with `C++23` we looked into the root dependency Basix and confirmed that we it is `C++23` ready in this PR.
This move to the `C++23` standard was later on reverted, and it was decided on to (not yet) switch to `C++23` but remain with `C++20` for now mainly due to lacking compiler support.

### DOLFINx - Update to `std::ranges` usage

Modernization to range based algorithms across all of DOLFINx, split up into several PR's:

- Replace `std::sort` with `std::ranges::sort` [(Merged)](https://github.com/FEniCS/dolfinx/pull/3293)
- Replace `std::fill` with `std::ranges::fill` [(Merged)](https://github.com/FEniCS/dolfinx/pull/3294)
- Replace `std::lower/uper_bound` with `std::ranges::lower/uper_bound` [(Merged)](https://github.com/FEniCS/dolfinx/pull/3295)
- Replace `std::for_each` with `std::ranges::for_each` [(Merged)](https://github.com/FEniCS/dolfinx/pull/3296)
- Replace `std::transform` with `std::ranges::transform` [(Merged)](https://github.com/FEniCS/dolfinx/pull/3297)
- Replace `std::set_*` with `std::ranges::set_*` [(Merged)](https://github.com/FEniCS/dolfinx/pull/3298)
- Replace `std::copy` with `std::ranges::copy` [(Merged)](https://github.com/FEniCS/dolfinx/pull/3299)
- Replace `std::min/max` with `std::ranges::min/max` [(Merged)](https://github.com/FEniCS/dolfinx/pull/3300)
- Replace `std::unique` with `std::ranges::unique` [(Merged)](https://github.com/FEniCS/dolfinx/pull/3315)

### Rework `radix_sort` [(Merged)](https://github.com/FEniCS/dolfinx/pull/3313)

While updating from `std::sort` to `std::ranges::sort`, I noted, that the custom DOLFINx implementation of radix sort `radix_sort` (which is used for performance critical sorting of degree of freedom indices mostly), is not compatible with the ranges interface.
In this pull request a range based `radix_sort` was introduced, which allows for 'equal' usage as `std::ranges::sort`.
For this the operator based approach as in the ranges library was followed and a projection routine for the elements can be injected.
This enabled for the removal of the previously also hand coded `argsort_radix` in DOLFINx, leaving only one `radix_sort` functionality left.
Due to its performance relevance the changes where benchmarked [here](https://github.com/schnellerhase/dolfinx/pull/24) and showed that the higher quality code is also the faster one.

### Favor `exterior_facet_indices` over `locate_entities_boundary` for retrieving complete boundary [(Merged)](https://github.com/FEniCS/dolfinx/pull/3283)

This pull request addressed the observation that when all boundary entities need to be selected in FEniCSx to construct a Dirichlet boundary condition on it, we must not filter these with a specific lambda as the default interface of `locate_entities_boundary` requires, but rather use the `exterior_facet_indices` which just gets all boundary facets.
This triggered a (still ongoing) discussion on optional arguments in the DOLFINx code base and its implications to the exported python module.

### Some other strictly maintenance PR's

- DOLFINx - Fix ruff check [(Merged)](https://github.com/FEniCS/dolfinx/pull/3349)
- febug - Fix installation with `dolfinx` main [(Merged)](https://github.com/nate-sime/febug/pull/13)
- FEniCS/web - Fix CI [(Merged)](https://github.com/FEniCS/web/pull/187)

## Acknowledgements

I would like to express my gratitude to the following individuals and institutions for their support and contributions:

Firstly I would like to the (previously mentioned) supervisors of the GSOC project, for their kind welcome to the community and receptivity to new ideas, as well as for fostering a generally constructive and engaging atmosphere for discussion.

I am indepted to [Henrik N. Finsberg](https://finsberg.github.io/) (Senior Research Engineer, Simula Research Laboratory) and [Jørgen S. Dokken](https://jsdokken.com/) (Senior Research Engineer, Simula Research Laboratory) for acting as my initial point of contact with the FEniCS project and for making me aware of the possibilty of a GSOC project.

Furthermore, I would be remiss if I did not acknowledge the contributions of all community members who provided assistance on the Slack channel or through comments and feedback on various pull requests.

During the GSOC period, I was fortunate to attend the [FEniCS conference 2024](https://fenicsproject.org/fenics-2024/), which I was able to do with the assistance of a NumFOCUS travel award.
Otherwise, it would not have been feasible to interact with all of the mentors and community members who were present in person.

In conclusion, I would like to express my gratitude to the Google Summer of Code program for providing me with the opportunity and financial support, which enabled me to pursue my academic interests and enhance my technical abilities with minimal constraints.