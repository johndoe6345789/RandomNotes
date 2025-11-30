# 3D CAD Software Programming Libraries

This document is a high‑level catalog of 3D CAD‑related programming libraries and SDKs, grouped by their primary role in a CAD stack.

> Scope: libraries intended for programmatic creation, manipulation, analysis, or exchange of 3D CAD data (geometry kernels, CAD scripting APIs, file I/O SDKs, etc.), rather than end‑user GUI applications.

---

## 1. Full‑scale 3D geometry kernels / CAD kernels

Libraries whose primary role is to provide B‑rep / solid / surface modeling, boolean operations, filleting, meshing, and often CAD data exchange.

* **Open Cascade Technology (OCCT)**

  * Language: C++ (bindings for Python, JS, others)
  * Type: Open‑source full‑scale 3D geometry and topology kernel
  * Notes: B‑rep solid modeling, NURBS, STEP/IGES/XT import/export, meshing, visualization. Used by FreeCAD, Salome, and many in‑house CAD/CAM/CAE tools.

* **CGAL (Computational Geometry Algorithms Library)**

  * Language: C++
  * Type: General computational geometry library
  * Notes: Robust geometric primitives, polyhedral booleans, triangulations, arrangements; more generic than a traditional CAD kernel, but often used inside custom CAD/CAE tools.

* **Parasolid**

  * Language: C API (C/C++ wrappers common)
  * Type: Commercial solid modeling kernel (Siemens)
  * Notes: Powers SolidWorks, Solid Edge, NX, Onshape, etc. Not open source; licensed as an SDK.

* **ACIS**

  * Language: C++
  * Type: Commercial solid modeling kernel (Spatial / Dassault Systèmes)
  * Notes: Used in many mid‑range CAD and CAM systems. STEP/IGES etc. via add‑ons.

* **ShapeManager**

  * Language: C++
  * Type: Autodesk’s proprietary kernel (fork of ACIS)
  * Notes: Internal to Autodesk products (Inventor, AutoCAD, Fusion). Exposed via product‑specific APIs, not as a generic third‑party kernel.

* **Granite / Granite One**

  * Language: C++
  * Type: PTC’s geometric modeling kernel
  * Notes: Used in Creo; SDK offered to partners.

* **C3D Modeler**

  * Language: C++
  * Type: Commercial modeling kernel (C3D Labs)
  * Notes: Offers modeling, constraints, parametrics, and data exchange components.

* **SMLib**

  * Language: C++
  * Type: Commercial modeling kernel (Solid Modeling Solutions)
  * Notes: NURBS‑centric geometric kernel.

* **sgCore**

  * Language: C++/.NET
  * Type: Proprietary modeling kernel distributed as an SDK
  * Notes: Solid modeling and booleans for technical / engineering CAD.

* **OpenNURBS**

  * Language: C++
  * Type: NURBS and 3DM file toolkit (McNeel)
  * Notes: Used for Rhino’s 3DM format; not a full solid kernel but often part of CAD stacks.

---

## 2. CAD scripting / parametric CAD libraries (geometry‑first)

Libraries designed for “code‑driven CAD”: using a programming language to define parametric models.

* **CadQuery**

  * Language: Python
  * Back‑end: Open Cascade (via OCP)
  * Notes: High‑level, fluent API for parametric modeling. Good for automation, batch CAD generation, and headless CAD.

* **pythonOCC / OCP**

  * Language: Python
  * Back‑end: Open Cascade
  * Notes: Direct Python bindings to OCCT. Low‑level access to the full modeling kernel, plus higher‑level helpers.

* **FreeCAD Python API**

  * Language: Python
  * Back‑end: OCCT and FreeCAD’s document model
  * Notes: Scriptable parametric CAD via FreeCAD “Documents,” features, and workbenches. Good for automation, custom tools, and batch export pipelines.

* **OpenSCAD language**

  * Language: Custom functional DSL (with wrappers from other languages)
  * Back‑ends: CGAL and others
  * Notes: CSG‑style parametric modeling via scripts. Commonly used through the OpenSCAD GUI, but models are defined as code.

* **SolidPython / Cadquery‑like wrappers**

  * Language: Python
  * Back‑end: OpenSCAD (generates .scad)
  * Notes: Pythonic wrapper around OpenSCAD, enabling CAD generation from Python that is then rendered in OpenSCAD.

* **SolveSpace kernel & scripting**

  * Language: C++ core with scripting / automation options
  * Notes: Parametric constraint‑based geometry engine; can be driven programmatically via command line / scripting for some workflows.

* **CascadeStudio / OpenCascade.js**

  * Language: TypeScript / JavaScript
  * Back‑end: Open Cascade compiled to WebAssembly
  * Notes: “Code CAD in the browser.” Good for web‑based parametric modeling, custom web tools, and educational CAD sandboxes.

* **Onshape FeatureScript**

  * Language: Onshape’s own functional language
  * Back‑end: Onshape (Parasolid)
  * Notes: Used to define custom features in Onshape documents (e.g., custom holes, patterns) entirely via code.

---

## 3. CAD data exchange, viewing, and interoperability SDKs

Libraries and SDKs specializing in reading/writing many CAD formats (STEP, IGES, Parasolid, CATIA, NX, SOLIDWORKS, etc.) and sometimes visualization.

* **CAD Exchanger SDK**

  * Language: C++, C#, Java, Python, JavaScript bindings
  * Notes: Commercial SDK for reading/writing many MCAD formats and visualizing them. Used to embed CAD import/export into custom apps.

* **Open Cascade XDE (Extended Data Exchange)**

  * Language: C++ (OCCT module)
  * Notes: Open source; handles STEP, IGES, and other neutral formats including colors, assemblies, PMI.

* **HOOPS Exchange**

  * Language: C++ / .NET
  * Notes: Commercial; high‑quality CAD data exchange SDK for downstream applications.

* **IFCOpenShell**

  * Language: C++ core with Python bindings
  * Notes: Focused on IFC/BIM models, but often part of CAD/BIM workflows for architecture and construction.

* **Open Asset Import Library (Assimp)**

  * Language: C++
  * Notes: Focused more on polygonal / DCC formats (OBJ, FBX, etc.) than solid CAD, but commonly used where CAD is converted to meshes for visualization, games, or XR.

---

## 4. CAD‑oriented visualization and scenegraph libraries

Rendering and interaction toolkits that are commonly used with CAD kernels.

* **VTK (Visualization Toolkit)**

  * Language: C++ with Python/Java bindings
  * Notes: Often used as the visualization layer on top of CAD kernels; supports 3D interaction, picking, clipping, and many filters.

* **Qt 3D / Qt Quick 3D**

  * Language: C++ / QML
  * Notes: Not CAD‑specific, but frequently used to build CAD/CAE GUIs with embedded 3D views.

* **three.js**

  * Language: JavaScript
  * Notes: Common destination for CAD‑to‑web pipelines; many CAD‑in‑browser tools convert B‑rep to meshes and render via three.js.

* **OpenSceneGraph**

  * Language: C++
  * Notes: Scene graph engine used in simulation and visualization software, including some CAD/CAE front ends.

* **Filament, bgfx, and other modern renderers**

  * Language: C++
  * Notes: General‑purpose physically based renderers often used as modern CAD visualization back‑ends.

---

## 5. Geometry, mesh, and CSG libraries used inside CAD workflows

These are not full CAD kernels but are often used in CAD‑adjacent tooling: mesh repair, remeshing, CSG, SDF modeling, etc.

* **libigl**

  * Language: C++
  * Notes: Geometry processing for meshes: remeshing, boolean operations, curvature, deformation.

* **Geometry3Sharp (g3)**

  * Language: C#
  * Notes: Mesh processing and basic solid modeling, used in Unity and .NET environments.

* **Carve CSG**

  * Language: C++
  * Notes: CSG boolean library, used historically in some CAD/visualization tools.

* **OpenVDB / NanoVDB**

  * Language: C++
  * Notes: Sparse volumetric data structures; used in voxel / SDF workflows that may bridge to/from CAD.

* **Recast/Detour etc.**

  * Language: C++
  * Notes: Mostly for navigation meshes and games, but sometimes appear in CAD‑like simulation tools.

---

## 6. Product‑specific programmatic APIs (embedded CAD automation)

Most major CAD systems expose APIs/SDKs that allow programmatic control, model creation, parametric edits, and custom add‑ins.

> These are not stand‑alone libraries in the same sense as OCCT or CGAL, but they matter for CAD programming.

* **Autodesk Inventor API**

  * Languages: C++, .NET, VBA
  * Notes: Access to Inventor’s parametric model, features, drawings, and assemblies.

* **Autodesk Fusion 360 API**

  * Languages: Python, JavaScript
  * Notes: Cloud‑backed parametric modeling, custom features, automation, and CAM workflows.

* **AutoCAD .NET / ObjectARX APIs**

  * Languages: C++, .NET
  * Notes: Lowest‑level access to AutoCAD’s drawing database; widely used for CAD automation.

* **SOLIDWORKS API**

  * Languages: C#, VB.NET, C++ (via COM)
  * Notes: Feature‑level automation, model interrogation, add‑ins, and custom UI.

* **Siemens NX Open**

  * Languages: C/C++, C#, Java, Python
  * Notes: Full access to NX modeling, drafting, and CAM.

* **Creo Toolkit / J‑Link**

  * Languages: C/C++ (Toolkit), Java (J‑Link)
  * Notes: Access to PTC Creo’s parametric models and features.

* **RhinoCommon / rhino3dm / Grasshopper scripting**

  * Languages: .NET, Python, JS, others
  * Notes: NURBS modeling APIs in Rhino, plus headless geometry via rhino3dm. Grasshopper exposes visual programming but can be scripted.

* **FreeCAD Workbench APIs**

  * Language: Python
  * Notes: Creation of custom workbenches and tools integrated into the FreeCAD GUI.

* **Onshape REST API + FeatureScript**

  * Languages: HTTP/JSON, plus Onshape’s FeatureScript language
  * Notes: Cloud CAD automation, integrations, custom features.

---

## 7. Web‑first CAD and 3D sketching libraries

Libraries designed specifically for browser‑based parametric CAD or lightweight CAD‑like experiences.

* **CascadeStudio / OpenCascade.js**

  * JS/WASM port of OCCT with a browser editor and an embedded DSL for CAD.

* **verb (verbnurbs)**

  * Language: JavaScript
  * Focus: NURBS curves and surfaces, intersection and trimming; useful for CAD‑like NURBS modeling in the browser.

* **Replicad / Web‑CAD wrappers**

  * Language: TypeScript / JavaScript
  * Focus: High‑level APIs on top of CAD kernels compiled to WebAssembly.

* **three‑CSG and similar JS CSG libraries**

  * Language: JavaScript
  * Focus: CSG operations on polygonal meshes, often used for simple “CAD‑ish” modeling directly on top of three.js.

---

## 8. Constraint solvers and parametric engines

These systems manage dimensions, constraints, and parametric relationships, which are central to many CAD tools.

* **D‑CM / LGS 3D (Bricsys / LEDAS heritage)**

  * Language: C++
  * Notes: Commercial geometric constraint solvers for 2D and 3D.

* **SolveSpace constraint solver**

  * Language: C++
  * Notes: Open source, embedded in the SolveSpace CAD program, often referenced as a lightweight parametric kernel.

* **SymPy / generic math libraries**

  * Language: Python, C++, etc.
  * Notes: Occasionally used in custom CAD tools to solve parametric equations, though not CAD‑specific.

---

## 9. How to choose libraries for a custom CAD toolchain

Key decision points:

* **Kernel vs mesh‑only:** Do you need exact B‑rep/NURBS (OCCT, Parasolid, ACIS, C3D) or are polygonal meshes sufficient (libigl, Assimp, OpenVDB)?
* **License:** Open source (OCCT, CGAL, libigl, FreeCAD, IFCOpenShell) vs commercial SDK (Parasolid, ACIS, Granite, CAD Exchanger, HOOPS Exchange).
* **Language ecosystem:** Python‑centric stacks (CadQuery, pythonOCC, FreeCAD) vs C++ heavy stacks (OCCT, CGAL, kernels) vs browser/JS stacks (OpenCascade.js, verb, three.js‑based tools).
* **Deployment target:** Desktop engineering app, headless server, plug‑in for existing CAD, or web app.

This document is intentionally broad rather than exhaustive. For any given project, you typically combine:

1. A **geometry kernel** (or mesh library),
2. A **data‑exchange layer**,
3. A **visualization stack**, and
4. Optional **scripting / automation APIs** and **constraint solvers**.
