---
title: "Google Summer of Code 2024 - Multigrid for FEniCSx"
date: 2024-08-30T12:00:00+02:00
draft: false
toc: true
math: true
aliases:
  - /en/posts/2024/gsoc/
---

## Intro

This is the report on the Google Summer of Code 2024 project, entitled ['Multigrid for FEniCSx'](https://summerofcode.withgoogle.com/programs/2024/projects/YhEwl8iB), under the auspices of the [NumFOCUS](https://numfocus.org/) organization, on the [FEniCS Project](https://fenicsproject.org/), which is (since 2016) fiscally supported by NumFOCUS.

Under the supervision of [Jack S. Hale](https://www.jackhale.co.uk/) (Research Scientist, University of Luxembourg), [Chris Richardson](https://www.ieef.cam.ac.uk/user/cnr12) (Research Software Engineer, University of Cambridge) and [Joseph Dean](https://www.eng.cam.ac.uk/profiles/jpd62) (Research Scientist, University of Cambridge) the Google Summer of Code provided an opportunity for me to contribute to the FEniCS project and get to know open source software development efforts for high-performance computing applications.

The FEniCS project's primary component is the finite element solving environment, [DOLFINx](https://doi.org/10.5281/zenodo.10447666), on which the majority of the project's work was conducted.
The "x" indicates that this is a new version, which supersedes the former (now legacy) [DOLFIN](https://doi.org/10.11588/ans.2015.100.20553).
Although the objective of both the legacy and current projects is identical, the necessity to adapt to new requirements, incorporate new technological advances and, integrate insights gained from previous experiences has prompted the decision to rewrite and redesign the project in a new version.

## Geometric Multigrid and an Application to GSOC

Let us for this section reference the proposal of the project, which explains the project's goals.

{{< embed-pdf url="/gsoc-proposal.pdf" renderPageNum="1" hidePaginator="true" >}}

## Implementation decisions

As an initial step in the research and familiarization phase, it was essential to figure assess the objectives of the FEniCS project with regard to a geometric multigrid implementation.
The interaction between this novel feature and existing ones, as well as the design principles of the DOLFINx code base played a major role during this phase.
Let us briefly revisit the findings and their implications for the project.

One of the fundamental design decisions of DOLFINx (also in contrast to its predecessor) is the incorporation of a data-driven design principle.
Functionality should be provided on standard data containers holding the necessary information, without further encapsulation and object based data wrapping, while implicit expectations on data or objects are to be kept to a minimum.
The primary rationale for this approach is to facilitate the extension of usability and functionality beyond the initially conceived scope of a given feature.

For the multigrid implementation this resulted, after careful consideration, in the decision to make the transfer operation itself an explicit function that needs to be provided explicitly for a specific use case.
This represents a different approach than that taken by other software, such as Firedrake (see [here](https://www.firedrakeproject.org/demos/geometric_multigrid.py.html)) and PETSc (see [here](https://petsc.org/release/manualpages/PC/PCGAMG/)), which employ an implicit construction of transfer operations.
Moreover, the mesh hierarchies to be considered in this project should be generated by the available refinement routines (based on the work of [Plaza](https://www.oden.utexas.edu/media/reports/1996/9654.pdf)), and not be applicable to general meshes without clear containment.

In addition to providing greater clarity for the user, the design should also, in principle, facilitate enhanced performance, or at the very least, more precise control over it.
This is because features are not utilized unless they have been employed by the user.

Nevertheless these specializations will not allow us to adress the discussed shared properties of geometric multigrid with $p$-multigrid and domain decomposition methods.
This should be a relatively straightforward process to implement once the geometric multigrid method has been implemented.

## Interpolation Operators and Dual Operators

A first idea to approach the implementation of a geometric multigrid with the hope of an easy to attain proof of concept, came with the availability of the non-matching mesh interpolation routine of DOLFINx.
This routine implements a general interpolation operation between non matching meshes, and thus also non-matching, function spaces in general.
So we have already the implementation at hand to produce for two different finite element approximation spaces $V_\text{coarse}$ and $V_\text{fine}$ mappings
So a first approach might be to make use of these mappings for a first geometric multigrid implementation and refine it after.
Nevertheless, a more detailed examination reveals an inherent and irremediable defect in this setup.

Let us consider an one dimensional example.

$$
V_\text{coarse} = P^1 ( \left\lbrace 0, 1 \right\rbrace )
\quad \text{and} \quad
V_\text{fine} = P^1 \left( \left\lbrace 0, \frac{1}{2}, 1 \right\rbrace \right)
$$

with the non matching interpolations as restriction $R : V_\text{fine} \to V_\text{coarse}$ and prolongation $P : V_\text{coarse} \to V_\text{fine}$ operators, i.e.

$$
P = \begin{bmatrix}
  1 & 0 \newline
  \frac{1}{2} & \frac{1}{2} \newline
  0 & 1
\end{bmatrix}
\quad \text{and} \quad
R = \begin{bmatrix}
  1 & 0  & 0 \newline
  0 & 0 & 1
\end{bmatrix}
$$

Especially the map $P$ is injective and $R$ is surjective, which fits our expectations.
But we loose one very important property that we require for transfer operators.
The restriction should be the adjoint of the prolongation, i.e. (for matrices)

$$
R = P^T  \iff P = R^T.
$$

For the example this is not the case.
This breaks the properties of the multigrid operator - for example without duality we loose every mathematical ground of proving any properties of the multigrid operator matrix, such as for example the for convergence critical spectral radius.

### Testing Discrete Adjoint Operators

The subtleties around the adjoint property do need end with this.
The requirement of the adjointness on the transfer operators and the implications on coefficients of the transferred functions preclude the possibility of recovering function scaling
Consequently, it is not feasible to expect that a combination of interpolations between the two spaces will yield a suitable transfer operator combination..

Furthermore, this suggests that meticulous attention must be paid to the criteria employed during the assessment of transfer operators, particularly with regard to the verification of implementations.
To illustrate, the constant one function is not recovered after it has been restricted and prolonged with suitable operators.
This is a consequence of our focus on the transfer of the residual, that is, an element of the dual space, not a function.

## An Interval Refinement Routine for DOLFINx

The logical step to start with a geometric multigrid in the context of mesh hierarchies generated by refinement routines is, of course, the one-dimensional case, i.e. interval meshes.
However DOLFINx did not provide any such routine at that time.
So as a first major step, the implementation of a one-dimensional refinement routine was addressed, which resulted in a (merged) [pull request](https://github.com/FEniCS/dolfinx/pull/3314).

Let us consider a small example to illustrate what we need to do.
Given a mesh consisting of three vertices (bold) and two cells (or equivalently edges)

**0** --- 0 --- **1** --- 1 --- **2** --- 2 --- **3**,

we want to perform a refinement on a set of marked edges $M$.
Let us first consider the case of uniform refinement, so we mark all edges for refinement, i.e. $M=\\{ 0, 1, 2 \\}$.
While in higher dimensions mesh refinement offers a great number of applicable refinement schemes to refine a marked edge, in one dimension the necessary operation is unique.
Each marked edge is to be split, i.e. a new vertex is to be introduced in the center and the former edge is to be split into two new finer/shorter edges.
For the former mesh we thus expect the fine mesh

**0** --- 0 --- **1** --- 1 --- **2** --- 2 --- **3** --- 3 --- **4** --- 4 --- **5** --- 5 --- **6**.

At first glance, this seems like a very straightforward operation to implement, but a few caveats arising from parallel data structures with relabeling of indexes for locality and the for parallel use cases inevitable repartitioning make it surprisingly sophisticated to implement.

For the debugging of the code, especially in parallel, looking at visualizations of the meshes before and after refinement was a crucial step.
This was easy to do thanks to the handy [febug](https://github.com/nate-sime/febug) tool, which makes visualizing different mesh entities straightforward and easy.

_Coming soon_: detailed exaplanation of the algorithm introduced in [PR 3314](https://github.com/FEniCS/dolfinx/pull/3314) including data stored in parallel, algorithm employed and output (parent cells)

### A Story on Assertions in Compiled Python Modules

After discovering an initial bug in the merged code, resulting in an obscure crash of a Python test, we discovered a problem in the interaction with assertions and the Python runtime.
The bug was caused by the code hitting an assertion in the C++ part of the software, correctly due to a poorly designed test case.
However the `pytest` framework, which is used for unit and integration testing in the FEniCS project, did not catch this and only showed an inconclusive core dump.
This turned out to be a major and not trivially fixable problem.
If the C++ part, i.e. in a call of the compiled Python module, encounters an C-style assertion the runtime itself is killed, as this triggers a call to ´abort´.
In particular, the handling of the assertion is not passed back to the outer Python context which has systems in place to recover from this.
Since `abort` produces a `SIGABRT` signal that forcibly exits the program there is no easy way to fix this interaction.
However, if the Python module (i.e. the C++ code) is built in a `Debug` mode we would like to be able to debug from the Python side as well, which is then no longer possible.
This needs to be investigated further and remains an [open issue](https://github.com/FEniCS/dolfinx/issues/3333).

### Unifying the Refinement Interface

After merging the one-dimensional refinement routine, we wanted to merge its interface with the previously available two- and three-dimensional Plaza refinement routines.
This resulted in many changes and simplifications to the interface.
It became more explicit and easier to use, resulting in an (open) [pull request](https://github.com/FEniCS/dolfinx/pull/3322).

One technical change that was introduced to facilitate this was the use of `std::optional`'s for input and output parameters.
Previously these were not used and a 'no-set' case was always handled separately leading to multiple different code paths.
In addition, this dispatching had to be present in the python module as well.
However, Nanobind (the Python exporter used in the DOLFINx project) however supports the `std::optionals` matching from the python optional `None` or value style for this.
So after the change, we no longer need to handle different parameter combinations in the Python module, this is all done in the C++ module.
These changes are part of another (open) [pull request](https://github.com/FEniCS/dolfinx/pull/3328).

## The Transfer Matrix

_Coming soon_: parallelization and ghost nodes, graph partitioning

### Debugging a Parallel CSR Matrix

Compressed Sparse Row Major (CSR) matrices are a great data format for efficient computations and FEM matrix assemblies.
However, the data format is not easy to debug or visualize.
For debugging purposes the DOLFINx CSR matrix implementation has a `to_dense` functionality that translates the CSR format into a dense representation of the matrix for debugging purposes.
However, this functionality only really supported sequential calls, and dropped quite a bit of information in parallel runs.
Let us recap what the functionality used to provide and present its new functionality.

Given two parallel partitions $I = (i_0, \dots, i_n)$ and $J = (j_0, \dots, j_n)$ associated with the row and column distribution across the ranks, i.e. the process $p$, for some $0 \leq p < n$, owns rows $[i_p, i_{p+1})$ and columns $[j_p, j_{p+1})$.
The `to_dense` function produced the dense subblock of the global matrix $A = (a_{ij})_{0\leq i < i_n,\ 0 \leq j \leq j_n}$ which was strictly global, i.e. the submatrix

$$
\begin{bmatrix}
  a_{i_p, j_p} & \dots & a_{i_p, j_{p+1}} \newline
  \vdots & \ddots & \vdots \newline
  a_{i_{p+1}, j_p} & \dots & a_{i_{p+1}, j_{p+1}}
\end{bmatrix}.
$$

However, this functionality drops from the dense representation all entries that are part of the local entries, and thus available on the given process rank $p$, that are in columns not owned by the current process $p$.
After the proposed changes `to_dense` now returns a dense representation of all locally owned rows (over all global columns), i.e. the submatrix we get on process $p$ is then

$$
\begin{bmatrix}
  a_{i_p, 0} & \dots & a_{i_p, j_n} \newline
  \vdots & \ddots & \vdots \newline
  a_{i_{p+1}, 0} & \dots & a_{i_{p+1}, j_n}
\end{bmatrix}.
$$

These changes are part of a (merged) [pull request](https://github.com/FEniCS/dolfinx/pull/3354) and are required for debugging and testing the transfer matrix implementation alike.

## Geometric Twogrid Example

_Coming soon..._

## General Code Contributions

While the primary objective was the implementation of a geometric multigrid, a number of additional changes were made to different repositories. These changes are outlined below in chronological order.

### DOLFINx - Introduce Aliasing for `mdspan` [(Closed)](https://github.com/FEniCS/dolfinx/pull/3116)

The `mdspan` library utilized by the FEniCS Project is based on the Kokkos [reference implementation](https://github.com/kokkos/mdspan).
To streamline the namespace switching process, depending on compiler support for either `std::experimental::mdspan` or `std::mdspan`, or no support at all the implementation introduces the macro `MDSPAN_IMPL_STANDARD_NAMESPACE`.
This results in unnecessarily lengthy calls, as it requires the repeated writing out of `MDSPAN_IMPL_STANDARD_NAMESPACE::mdspan` each time `mdspan` is used
In the pull request, the suggestion was to wrap this into a custom hiding layer, allowing for access via `dolfinx::common::mdspan`, regardless of what this dispatches to.
This approach would also enable the code to remain unchanged in the event of an STL version update.
Following a discussion, the proposed changes were rejected on the grounds that they constituted an approximation of a third-party library.

### Basix - Update mdspan [(Merged)](https://github.com/FEniCS/basix/pull/840)

Update to new Kokkos single header `mdspan` changes.

### DOLFINx - Make compile time options compile time constants [(Merged)](https://github.com/FEniCS/dolfinx/pull/3246)

Previously, the transformation of compiler flags in translation units was moved to header files in combination with the usage of `consteval`, which effectively made them true compile time constants.
The following example illustrates the idiom of the changes for a given CMake flag.

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

While researching mesh-related data structures in DOLFINx for the project, I discovered several suboptimal code paths in the mesh generation code.
This PR addresses these issues, primarily through modernization, simplification, and additional testing.
The modernizations also made use of the previously (in the FEniCS Project) unused [ranges library](https://en.cppreference.com/w/cpp/ranges), this was the trigger for looking into further use cases of it, resulting in the following 12 pull requests.

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

During the process of updating from `std::sort` to `std::ranges::sort`, it was observed that the custom DOLFINx implementation of radix sort `radix_sort` (which is utilized for performance-critical sorting of degree of freedom indices primarily) is not compatible with the ranges interface.
This pull request introduces a range-based radix_sort, which offers the same functionality as `std::ranges::sort`.
To achieve this, the operator-based approach used in the ranges library was followed, and a projection routine for the elements can be added.
This allowed for the removal of the previously hand-coded `argsort_radix` in DOLFINx, leaving only one instance of the `radix_sort` functionality.
Given its performance-critical nature, the changes were benchmarked [here](https://github.com/schnellerhase/dolfinx/pull/24).
The results demonstrated that the higher-quality code is also the faster one.

### Favor `exterior_facet_indices` over `locate_entities_boundary` for retrieving complete boundary [(Merged)](https://github.com/FEniCS/dolfinx/pull/3283)

This pull request addresses the observation that when all boundary entities need to be selected in FEniCSx to construct a Dirichlet boundary condition on them, it is not necessary to filter these with a specific lambda as the default interface of `locate_entities_boundary` requires.
Instead, the `exterior_facet_indices` can be used, which simply returns all boundary facets.
This resulted in a (still ongoing) discussion on optional arguments in the DOLFINx code base and their implications for the exported Python module.

### Strictly Maintenance PR's

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
