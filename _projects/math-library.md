---
hasThumbnail: true

visit:
  - anchorText: "View on GitHub"
    link: "https://github.com/YashCK/Math_Library"
---

## Overview

This a Java library built from scratch that was aimed at higher-level mathetmatics. I wanted to combine practical numerical routines with algebraic structure, so the same linear algebra tools can work over different number systems (and even user-defined types) when they satisfy the right algebraic properties.

When I built this library, I was trying to optimize for a few things:

- **Build core math tools from scratch** (rather than relying on existing libraries), to understand the algorithms and data structures deeply. 
- **Make the API flexible and reusable**, so the same vector/matrix ideas apply to different underlying scalar types.
- **Use abstract algebra as the organizing principle**: if a type behaves like a Field (or Group/Ring), then linear algebra operations can be defined generically on top of it.
- **Keep the implementation modular and readable**, so it’s easy to extend with new objects and algorithms over time.

It supported many facets of linear algebra, centered around generic vectors and matrices that generalize the idea of working over \( \mathbb{F}^n \) (a vector space over a field). 

**Vectors**
- Vectors are designed to be **parameterized by type**, so you can construct vectors over different element types
- The library is structured so you can define your own vector types 
- Core operations include common vector computations like **addition/subtraction and inner products**.

**Matrices**
- Matrices are also generic and built as “collections of vectors,” supporting multiple underlying element types.
- The library supports matrix operations like **addition/subtraction, matrix multiplication, and column/row operations**.
- Matrices can be instantiated from either **row** or **column** representations.

There are some general implemented methods such as:
- **Row reduction**
- **Gaussian elimination**
- **Gauss–Jordan**
- **LU-based solving**

Mnay other features were planned, such as determinants/inverses, condition numbers, eigen-stuff, decompositions like QR/SVD, etc.

The abstract algebra foundations in the library included abstractions for:
- **Set**
- **Additive and multiplicative groups**
- **Ring**
- **Field**

It also includes predefined fields:
- **Real**
- **Complex**
- **Integers modulo p**

I also initally wanted to support multivariable calculus, differential equations, and linear, programming/optimization but didn't end up getting to it.