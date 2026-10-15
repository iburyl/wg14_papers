# Standard support for float-float and double-double arithmetic in C

| Field | Value |
| --- | --- |
| Document | Working draft; WG14 document number not assigned |
| Date | 2026-10-15 |
| Authors | Nikita Astafev — NVIDIA |
| | Ilya Burylov — NVIDIA — [burylov@gmail.com](mailto:burylov@gmail.com) |
| | Andrey Kolesov — NVIDIA |
| | Tue Ly — Google — [lntue@google.com](mailto:lntue@google.com) |
| Audience | ISO/IEC JTC 1/SC 22/WG 14 |
| Purpose | Motivation and direction for discussion |

## Abstract

Float-float and double-double arithmetic represent one number as the unevaluated
sum of two floating-point components. Implementations exist in compiler ABIs,
mathematical libraries, geometry software, rendering systems, and scientific
applications. Their users repeatedly specify local representations, arithmetic
interfaces, conversions, and evaluation requirements.

This paper asks WG14 to consider standard support for these representations and
their arithmetic without defining specific implementation details.
The first objective is agreement on the numerical and programming contracts that
such support should provide - the next step would be to extend with specific
implementation proposal.

A reusable two-component arithmetic facility needs to specify how results are
represented, rounded, combined, classified, converted, and passed between
independently written C components. The difference between rounding in common
implementations and in IEEE 754 augmented arithmetic raises questions of
compatibility and implementation cost that a future design must resolve.

## 1. Request to WG14

WG14 is asked to consider the following direction:

> Develop a specification for optional float-float and double-double arithmetic
> in C, with explicit rounding semantics and consideration of existing practice,
> with explicit representation, accuracy, conversion, and evaluation contracts.

An initial specification could be limited to binary32-based float-float and
binary64-based double-double. These component formats should be identified
explicitly: C does not require every implementation's `float` and `double` to
have those formats. Generalization to other component formats can be considered
separately.

## 2. Representation and scope

For finite components, the represented mathematical value is

```text
x = hi + lo
```

In this paper, **float-float** means a pair of binary32 components and
**double-double** means a pair of binary64 components. **Double-word arithmetic**
is the common name for the underlying technique. A normalized pair is commonly
described by requiring the high component to be the round-to-nearest,
ties-to-even result of the pair's exact sum. This describes normalization of the
represented pair, not a promise that each composite operation is correctly
rounded. A specification must make admissible components, boundary cases, and
normalization guarantees precise. This describes common practice without choosing
the standard's rounding policy; that question is developed in §5.3.

In suitable ranges, the arithmetic can deliver accuracy on the scale of twice
the component precision: approximately 48 bits for float-float and 106 bits for
double-double. These are not claims of uniform precision at every magnitude or
of correctly rounded operations on a conventional fixed-width significand.
The exponent range remains tied to the component format, and low-component
underflow limits the extra precision near zero.

The target of this paper is a reusable two-component arithmetic facility.
Variable-length expansions, compensated reductions, two-part dates, and
origin-relative coordinates can inform requirements, but do not by themselves
establish use of this facility. Integer-based floating-point emulation and
ordinary mixed-precision computation are also distinct.

The algorithms and their error bounds have an established literature. Proven
bounds depend on the particular algorithm and its assumptions; they should not
be inferred solely from the number of stored components. [R1]

## 3. Existing practice

The following examples establish different forms of existing practice. Compiler
types, library interfaces, internal helpers, and published applications are
identified separately. A source implementation does not establish execution in
every downstream program.

| Example | Evidence | Relevance to C |
| --- | --- | --- |
| PowerPC IBM extended double | Published ABI representation and compiler support through `__ibm128`; LLVM distinguishes `ppc_fp128` from IEEE `fp128`. | A compiler-supported two-double format already exists, with documented semantic qualifications. [R2] |
| QD | Explicit `dd_real` library arithmetic, with C and Fortran interfaces as well as C++ support; Berkeley Lab provenance. | Demonstrates a software implementation and demand for access across language boundaries. [R3] |
| XBLAS | Its extra internal precision is implemented using double-double, while inputs and outputs retain the BLAS interface formats. | Direct precedent for DD inside a C numerical-library interface. [R4] |
| NVIDIA CCCL / libcu++ | `<cuda/fpmp>` provides production FF/DD types and arithmetic in the current main branch. | A C++ library implementation for host and device code whose experience can inform a C facility. [R9] |
| GEOS / JTS | Explicit DD arithmetic used by geometry algorithms, including robust orientation and intersection calculations. | A concrete application requiring more reliable intermediate arithmetic. These libraries are C++ and Java; they provide requirements evidence rather than a standard C API. [R5] |
| Epic Unreal Engine | Documented High/Low binary32 components in the DoubleFloat rendering library, with references to double-word arithmetic literature. | Establishes a separate need for float-float on graphics-oriented targets. The interface itself is C++/HLSL. [R6] |
| Google Highway | Float32/float64 high-low SIMD helpers with addition, multiplication, division, and mathematical-function callers. | Shows both component precisions in a portable SIMD library; the API is C++. [R7] |
| AMD ROCm OCML | `float2`/`double2` high-low arithmetic and power-function callers. | An additional independent implementation example. [R8] |
| QUDA | DD arithmetic and documented DD reductions for lattice QCD. | A scientific use case beyond ordinary mixed-precision solver updates. [R19] |
| NVIDIA cuEST | Documented float-float scalar kernels for quantum chemistry. | Application use exposed through a library with a C API; distinct from its GEMM emulation. [R20] |
| Scientific applications | Reports of DD tallies in a Los Alamos MCNP alpha solver, DD intermediates in QR research, and a custom pySecDec amplitude calculation presented by a CERN researcher. | Application evidence extending beyond library construction; public-release coverage is not established by these reports. [R10–R12] |

These examples support standardizing a common contract. They do not establish
agreement on one existing implementation's special values, rounding semantics,
layout, or interface. Nor do they establish a universal performance advantage.

### 3.1 Scientific computing and library integration

QD and XBLAS illustrate two useful interfaces: a reusable scalar abstraction and
extra precision internal to an otherwise conventional numerical operation.
Neither interface subsumes the other. [R3, R4]

CCCL/libcu++ provides the `fpmp2` class template and `fp32mp2` / `fp64mp2`
aliases through `<cuda/fpmp>`. Its interface includes arithmetic, conversions,
comparisons, mathematical functions, and numeric limits, with selectable
arithmetic methods. This production C++ implementation illustrates the scope of
a reusable FF/DD facility without prescribing the names or design of a C
interface. [R9]

Sandia-origin Trilinos documents configurable integration of QD's `dd_real`,
including an Ifpack2 solver option and serialization support. In contrast,
PETSc's FAQ reports that its attempt to integrate QD C++ classes failed because
they could not serve as drop-in native numeric types. These are evidence of
integration work, not proof that a library-only C design is inadequate or that
standardization alone would resolve PETSc's difficulties. [R13, R14]

### 3.2 Application requirements

Geometry algorithms use DD intermediates to improve robustness; rendering
systems use FF to retain coordinate precision on binary32-oriented targets.
GEOS/JTS and Unreal DoubleFloat demonstrate these different requirements. [R5, R6]

QUDA documents DD reductions for lattice QCD, backed by its `dbldbl.h`
implementation and Kate Clark's Lattice 2023 presentation. These reductions
are distinct from ordinary mixed-precision reliable updates and do not
guarantee order-independent results. [R19]

cuEST uses float-float in Polarizable Continuum Model solvent kernels and the
VV10 nonlocal correlation functional. These scalar kernels use pairs of FP32
values, separately from its Ozaki-based GEMM emulation. Their documented
precision and range limits, and per-function controls, illustrate the need for
an explicit arithmetic contract. [R20]

### 3.3 Why float-float belongs in the same work

Float-float serves targets where binary32 arithmetic is the practical execution
format. Double-double serves computations seeking more precision than binary64.
Their basic representation and many algorithms are shared, but neither the
precision nor the available hardware operations are interchangeable. Rendering
and quantum-chemistry kernels motivate float-float; lattice QCD reductions
motivate double-double. Libraries supporting both demonstrate the opportunity
for a common design. [R6, R7, R9, R19, R20]

A C specification would not, by itself, govern shader-language evaluation.
Graphics examples motivate the numerical facility and possible interoperability;
they do not imply that WG14 can fix non-C compiler semantics.

## 4. Relation to existing standards work

IEEE 754-2019 includes recommended augmented arithmetic operations. Their results
provide building blocks for higher precision and reproducible computation, as
described in the revision background. A primitive producing a head and tail is
not, by itself, a specification of arithmetic on persistent two-component
values. [R15]

WG14 N3180 is the November 2023 working draft of a separate supplementary
floating-point Technical Specification. Its explanatory DD-addition example
following §7.3 states: "There is no standard specification for double-double."
The example uses `aug_add`; it does not define a general DD type. [R16]

ISO/IEC TS 18661-4:2025 was subsequently published as a separate specification
of supplementary functions absent from C23. This paper relies on the public
N3180 draft for the example and its wording, and on ISO's publication record
for the status of the 2025 TS. It does not assume that every detail of the draft
is unchanged in the published text. [R17]

These documents provide relevant background. Whether a future FF/DD facility
should adopt their helper semantics, permit them as an implementation technique,
or specify its operations independently remains open. Their special tie-breaking
rule differs from common FF/DD practice, raising compatibility and performance
questions (§5.3).

Adoption as a TS would permit explicitly identified supplementary support;
it would not make the facility a requirement of base C conformance or promise
later incorporation into core C.

## 5. Problems that a shared specification could address

### 5.1 Independently defined representations and interfaces

A program may have to translate between a library class, a two-element vector,
two scalar arguments, and a high/low structure. Identical component widths do
not imply identical contracts. Operand normalization, exceptional values, and
the precision of intermediate operations can differ.

A shared C interface would allow a producer, consumer, and numerical kernel to
name the same facility. It could reduce adapter code without requiring existing
implementations to change their internal representations.

### 5.2 Evaluation requirements

Error-free transformations rely on specified rounding points. Reassociation,
unintended contraction, excess evaluation precision, or loss of subnormal
residuals can invalidate an algorithm's assumptions.

This is not a claim that conforming C implementations may freely discard
required semantics. The issue is to define which semantics the facility
requires, how it interacts with C evaluation controls, and what implementation
modes support it. A function-level arithmetic contract can allow an
implementation to select instructions while preserving the specified result.

Explicit FMA is often part of a correct algorithm; forbidding all contraction
or fused operations is not an adequate general solution. The specification
should protect required rounding points and permit transformations that
preserve its contract.

There is practical evidence for this concern. The Los Alamos report describes
compiler sensitivity in its DD implementation. Epic's UE-231741 identifies
reassociation as the apparent cause of a DoubleFloat rendering defect, with a
target fix in UE 5.6. These are specific reports, not blanket findings about
current C compilers. [R10, R18]

### 5.3 Rounding mode and compatibility with ordinary arithmetic

Common FF/DD implementations use ordinary round-to-nearest, ties-to-even
component arithmetic. For example, the head produced by a TwoSum or
FMA-based TwoProduct is the usual rounded scalar sum or product; additional
operations recover its residual under the algorithm's preconditions. The
rounding of such component operations, normalization of a pair, and the error
of a composite FF/DD operation are distinct parts of the contract. [R1, R9, R19]

IEEE augmented arithmetic uses round-to-nearest with ties toward zero. N3180
specifies that rule independently of the dynamic and constant rounding
directions. This is a different tie-breaking rule, not ordinary rounding toward
zero for every result. At halfway cases it can select a different head, with
a correspondingly different tail. [R16]

Making that helper contract a prerequisite could introduce implementation cost.
On targets without direct support, the special tie rule can require additional
software steps; it does not require changing the hardware rounding mode in every
implementation. N3180 itself notes that faster DD-addition methods exist when
augmented-addition hardware is unavailable. Whether the helper semantics offer
benefits that justify this cost for a standard FF/DD facility needs evaluation;
no universal performance penalty is asserted. [R16, R21]

The design question is therefore which rounding contract best serves the
facility: ordinary ties-to-even component operations, augmented-arithmetic
semantics, or another explicitly specified policy. The consequences for existing
algorithms, halfway-case representations, accuracy bounds, and implementation
cost need comparison before a choice is made.

The floating-point-environment policy also remains open. A design must determine
which caller-selected modes are supported, whether any mode is an execution
precondition or a fixed semantic guarantee, and how calls interact with the
caller's environment. Any mode changes and restoration costs must be considered
alongside the chosen policy.

### 5.4 Precision, range, and classification

Treating a pair as an ordinary format with exactly twice the component's
significand width can give misleading limits and spacing assumptions. A design
must distinguish the representable value set from the accuracy guaranteed by
its operations. The PowerPC ABI documents variable effective precision and
limitations near underflow. [R2]

NaNs, infinities, and signed zeros also need a composite interpretation. It is
insufficient to specify only how finite normalized operands are processed and
leave callers to infer the rest.

### 5.5 Conversion and constants

Constructing a pair from an existing `double` preserves that input value; it
cannot recover information already lost when a decimal token was converted to
`double`. Reading a decimal character sequence directly into a pair is a
different operation.

Conversions from integers, scalar floating types, component pairs, and text
therefore need distinct contracts. Narrowing back to one component needs a
specified rounding rule. If language types are considered, constant expressions
and literals require explicit treatment as well.

### 5.6 Interoperability

An interface needs well-defined construction, extraction, and passage of values
between translation units. Byte layout and external serialization are separate
choices: a common C arithmetic API need not prescribe one cross-platform wire
format. Teuchos's explicit DD serialization support illustrates work currently
performed by individual libraries. MPI datatype design is outside the immediate
scope of this request. [R13]

## 6. Possible forms of support

Three forms merit consideration; combinations are possible.

| Form | Benefit | Questions it leaves to address |
| --- | --- | --- |
| Library types and named functions | Can establish a common C interface with comparatively limited language changes. | Arithmetic uses function calls; type-generic dispatch, initialization, and interaction with existing numerical APIs need design. |
| Optional language arithmetic types | Could integrate operators, conversions, expressions, and generic selection. | Requires decisions on rank, usual arithmetic conversions, literals, constant evaluation, complex types, and the floating-point model. |
| Supplementary specification | Can provide an optional, separately identified facility and implementation experience. | Still needs precise semantics and a conformance mechanism; publication as a supplement does not settle library versus language form. |

A productive first step would be a library-shaped semantic design, assessed
alongside implementer feedback on optional language types. This is a proposed
development strategy, not a conclusion that the library form is sufficient.

Existing IBM double-double should be studied as compatibility evidence rather
than assumed to define the portable facility. Any differences in values,
exceptions, accuracy, or normalization would need explicit mapping. [R2]

## 7. Candidate minimum scope

A follow-up design should at least address the following items. They are a
requirements checklist, not proposed normative wording.

1. **Availability and component formats.** Explicit feature identification;
   distinguish binary32/binary64 components from implementation-dependent C
   type names. Define any rounding, subnormal, or execution-mode prerequisites.
2. **Value model and construction.** Admissible finite pairs, normalization,
   zero, infinities, NaNs, and construction from arbitrary components. State
   whether noncanonical inputs are accepted, normalized, or rejected.
3. **Basic operations.** Addition, subtraction, multiplication, division,
   negation, comparison, and classification. State accuracy bounds, valid
   domains, overflow/underflow behavior, and exceptional cases per operation.
4. **Conversions.** Integer and scalar floating conversions, component access,
   narrowing, and decimal character conversion sufficient to preserve the
   facility's extra precision. Specify applicable rounding rules.
5. **Rounding policy.** Choose and specify component rounding, pair
   normalization, operation accuracy, and behavior under caller-selected modes.
   Evaluate compatibility and cost when deciding the relationship to
   augmented-arithmetic helpers and their special tie rule (§5.3).
6. **Evaluation and environment.** Exception flags, excess precision,
   contraction, implementation options, and preservation of the caller's
   environment. Define guarantees across separately compiled components.
7. **Limits and queries.** Meaningful accuracy and range information. Explain
   how this relates to, or differs from, the single-significand model underlying
   familiar floating-point limits and adjacent-value operations.
8. **C integration.** Object initialization, assignment, parameter passing,
   headers, names, feature-test mechanisms, and type-generic facilities as
   appropriate to the selected form.

A first facility need not provide a complete transcendental library, arbitrary
expansion lengths, complex arithmetic, vector types, an MPI binding, or a
portable serialized encoding. It should leave those extensions possible.
Nor need it require hardware acceleration or require every operation to be
correctly rounded, provided its stated accuracy contract is useful and precise.

## 8. Alternatives and limits of the proposal

**Use a wider scalar type.** `long double` or an IEC 60559 extended type may be
the best choice on a given implementation. Their precision, availability, and
cost do not universally match the two-component use cases. Supporting FF/DD
would add a choice rather than replace those types.

**Use a third-party library.** This is already viable and is the basis for much
of the evidence. Standardization could supply a common contract and interface;
it cannot eliminate all migration work or guarantee faster implementations.

**Use augmented arithmetic alone.** These primitives do not specify a
persistent FF/DD facility. Their tie rule also differs from common practice.
Whether to build on their semantics, use them as an implementation technique,
or define independent operations remains a question for the design (§5.3).

**Use compensation, exact accumulators, or algorithm reformulation.** These may
be better solutions for particular reductions or stability problems. DD alone
does not guarantee order-independent sums, exact predicates, or numerical
stability for arbitrary algorithms. Its usefulness does not depend on replacing
those techniques.

The strongest case for WG14 action is the recurring need for a portable
numerical interface, supported by multiple independent implementations and
concrete applications. Download counts, hardware emulation trends, and broad
claims of universal deployment are unnecessary to that case.

## 9. Questions for committee discussion

1. Is there interest in developing standard C support for persistent
   two-component floating-point values and arithmetic?
2. Should the initial scope include both binary32-based float-float and
   binary64-based double-double?
3. Should the next paper develop library types and functions, optional language
   types, or both alternatives for comparison?
4. Should that work target a supplementary specification initially, or be
   developed directly as a proposal for a future revision of C?
5. Which numerical contracts and C integration issues in §7 must be resolved
   before WG14 can choose the form?
6. Which rounding policy best balances existing practice, numerical guarantees,
   and implementation cost, and what relationship should it have to IEEE
   augmented arithmetic (§5.3)?

## References

The references below are primary specifications, project documentation/source,
or reports by the relevant authors. Development-branch links establish inspected
source, not inclusion in a particular released binary. Evidence was assembled
on 2026-10-13–15; status qualifications are intentional.

- **R1. Numerical foundations.** Joldes, Muller and Popescu, *Tight and rigorous
  error bounds for basic building blocks of double-word arithmetic*, ACM TOMS
  44(2), 2018, [DOI 10.1145/3121432](https://doi.org/10.1145/3121432).
  Muller and Rideau, *Formalization of double-word arithmetic, and comments on
  “Tight and rigorous error bounds for basic building blocks of double-word
  arithmetic”*, ACM TOMS 48(1), 2022.
- **R2. Compiler and ABI precedent.** [PowerPC64 ELF ABI v1.7, §3.1.5](https://refspecs.linuxfoundation.org/ELF/ppc64/PPC-elf64abi-1.7.html);
  [GCC floating types](https://gcc.gnu.org/onlinedocs/gcc/Floating-Types.html);
  [LLVM floating-point types](https://llvm.org/docs/LangRef.html#floating-point-types).
- **R3. QD.** [Project README and author affiliations](https://raw.githubusercontent.com/BL-highprecision/QD/master/README);
  [Berkeley Lab software directory](https://amcr.lbl.gov/software-portal/amcr-software-directory/).
- **R4. XBLAS.** [Extra Precise Basic Linear Algebra Subroutines](https://www.netlib.org/xblas/),
  including its explicit DD implementation statement and C implementation description.
- **R5. Geometry.** [GEOS DD representation](https://github.com/libgeos/geos/blob/main/include/geos/math/DD.h);
  [GEOS DD algorithm callers](https://github.com/libgeos/geos/blob/main/src/algorithm/CGAlgorithmsDD.cpp);
  [JTS DD implementation](https://github.com/locationtech/jts/blob/master/modules/core/src/main/java/org/locationtech/jts/math/DD.java).
- **R6. Float-float rendering.** Epic Games, [Large World Coordinates Rendering](https://dev.epicgames.com/documentation/en-us/unreal-engine/large-world-coordinates-rendering-in-unreal-engine-5).
- **R7. Google Highway.** [FF/DD helpers](https://github.com/google/highway/blob/master/hwy/contrib/math/fp_arith-inl.h);
  [math-function callers](https://github.com/google/highway/blob/master/hwy/contrib/math/math-inl.h).
- **R8. AMD ROCm OCML.** [Two-component arithmetic](https://github.com/ROCm/llvm-project/blob/amd-staging/amd/device-libs/ocml/src/ep.h);
  [double power-function callers](https://github.com/ROCm/llvm-project/blob/amd-staging/amd/device-libs/ocml/src/powD_base.h);
  [float callers, including unsafe-math alternatives](https://github.com/ROCm/llvm-project/blob/amd-staging/amd/device-libs/ocml/src/powF_base.h).
- **R9. NVIDIA CCCL / libcu++.** [Public `<cuda/fpmp>` header](https://github.com/NVIDIA/cccl/blob/main/libcudacxx/include/cuda/fpmp);
  [FF/DD types and core operations](https://github.com/NVIDIA/cccl/blob/main/libcudacxx/include/cuda/__fp/fpmp.h).
  Production implementation in the current main branch.
- **R10. Los Alamos application report.** C. J. Josey, *General Improvements to
  the MCNP Alpha-Eigenvalue Solver*, LA-UR-18-22738, 2018-03-29,
  [DOI 10.2172/1431059](https://www.osti.gov/servlets/purl/1431059).
  DD tallying and compiler sensitivity were checked in indexed report text;
  direct full-document retrieval and current-release source verification were unavailable.
- **R11. QR research.** Yamazaki, Tomov and Dongarra, *Mixed-precision Cholesky
  QR factorization and its case studies on multicore CPU with multiple GPUs*,
  SIAM J. Sci. Comput. 37(3):C307–C330, 2015,
  [ORNL publication record](https://impact.ornl.gov/en/publications/mixed-precision-cholesky-qr-factorization-and-its-case-studies-on/),
  DOI 10.1137/14M0973773. Historical research evidence.
- **R12. CERN-associated amplitude calculation.** *Two-loop amplitudes for ttH
  production: the quark-initiated Nf-part*, 2024,
  [arXiv:2402.03301, §2.6](https://arxiv.org/html/2402.03301v3#S2.SS6);
  Vitaly Magerya (CERN), *Double-double for virtual amplitude evaluation*,
  [CERN NGT/Openlab workshop programme, 2 July 2025](https://indico.cern.ch/event/1538409/timetable/?print=1&view=standard).
  The published calculation and programme establish application and affiliation;
  public-release inclusion of the DD extension was not verified.
- **R13. Trilinos integration.** [Project origin and governance](https://trilinos.github.io/about.html);
  [Ifpack2 `dd_real` option](https://trilinos.github.io/ifpack2.html);
  [Teuchos serialization](https://trilinos.github.io/docs/teuchos/class_teuchos_1_1_serialization_traits.html).
- **R14. PETSc integration experience.** [FAQ: QD and extended precision](https://petsc.org/release/faq/#why-doesn-t-petsc-use-qd-to-implement-support-for-extended-precision).
- **R15. Augmented arithmetic background.** IEEE 754-2019 revision group,
  [background and recommended operations](https://grouper.ieee.org/groups/msc/ANSI_IEEE-Std-754-2019/background/).
- **R16. WG14 N3180.** [CFP working draft, 13 November 2023](https://www.open-std.org/jtc1/sc22/wg14/www/docs/n3180.pdf),
  §7 and the explanatory example following §7.3.
- **R17. Published supplementary specification.** ISO,
  [ISO/IEC TS 18661-4:2025 publication record](https://committee.iso.org/cms/live/live/en/sites/isoorg/contents/data/standard/08/36/83622.html).
- **R18. Reported optimizer sensitivity.** Epic Games,
  [UE-231741](https://issues.unrealengine.com/issue/UE-231741),
  DoubleFloat reassociation report, target fix 5.6.
- **R19. QUDA DD application.** [QUDA project](https://github.com/lattice/quda);
  [NVIDIA-authored `dbldbl.h`](https://github.com/lattice/quda/blob/develop/include/dbldbl.h);
  Kate Clark, *Restoring Reproducibility to Lattice QCD*, Lattice 2023,
  [DD reductions, slides 8–11](https://indico.fnal.gov/event/57249/contributions/270632/attachments/169825/228082/reproduciblity_lattice2023.pdf).
- **R20. NVIDIA cuEST scalar float-float.** [Scalar Emulation in cuEST](https://docs.nvidia.com/cuda/cuest/scalar_emulation.html),
  including kernel use cases, precision/range limitations, and math-mode controls.
- **R21. Augmented-operation emulation.** Boldo, Lauter and Muller,
  *Emulating Round-to-Nearest Ties-to-Zero “Augmented” Floating-Point Operations
  Using Round-to-Nearest Ties-to-Even Arithmetic*, IEEE Transactions on
  Computers 70(7):1046–1058, 2021,
  [DOI 10.1109/TC.2020.3002702](https://doi.org/10.1109/TC.2020.3002702).
  Also cited in N3180, reference [16].
