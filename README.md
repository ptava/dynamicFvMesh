# dynamicFvMesh

Working on adaptive mesh refinement in OpenFOAM.

- [x] Sort candidate cells for refinement based `cellError` *scalarField*.
- [x] Support both single-processor and parallel runs by reconstructing `cellError` field.
- [x] Stop refine cells once user-defined threshold is reached.
- [x] Add synchronization across processors to ensure consistency across boundaries.
- [x] Make available for processing additional information on adaptive mesh refinement
- [x] Introduce a user-defined time-based smoothing to gradually ramp in cell refinement at startup.


---

## OpenFOAM Class for Dynamic Refinement and Un-refinement

The `dynamicRefineFvMesh` class uses several parameters to handle refinement and un-refinement within the computational mesh; they need to be specified in the `dynamicMeshDict` in the `constant` folder along with the associated field name:

- **lowerRefineLevel** and **upperRefineLevel** → Cells with `fieldValue` in between these limits are selected for refinement.
- **unrefineLevel** → Cells, already split, with `fieldValue` lower than the specified `unrefineLevel` are selected for unrefinement.
- **maxCells** → Maximum number of cells allowed within the computational domain.
- **maxRefinement** and **nBufferLayers** → Control parameters associated with cell refinement level.

The adaptive refinement in OpenFOAM needs a `volScalarField` to be evaluated each timestep, or at least each `refineInterval`. This field will be manipulated in order to obtain the actual `fieldValue` considered in the refinement criteria definition. From now on the input field defined in the dictionary will be named **`field−`**, while the actual field used for cell selection will be named **`field+`**. This highlights the effects of the smoothing operation defined in the OpenFOAM class for dynamic refinement and un-refinement.

---

## Improvement of General Validity to the OpenFOAM Class

The `field+` is a reconstruction of the smoothed field considered to mark cells for refinement.  
Before smoothing, the code evaluates the **`cellError` scalarField**:

- `cellError` is the minimum distance between the range limits and the cell value.
- It approaches 0 for interpolated values near the limits, and its maximum in the mid-region of the specified range.
- It can be computed before and after smoothing operation (`field-` and `field+`).

In the latest OpenFOAM version, any cell with `cellError > 0` is considered a candidate for refinement.  
**Problem**: All candidates are treated equally, ignoring the magnitude of `cellError`.  
**Solution**: Sort cells by `cellError` (highest to lowest) and refine until reaching `maxCells`.

---

## Sorting Candidate Cells

**Parallel computing issue**: Sorting across processors.

**Approach**:
1. Gather all `cellError` lists from processors with `Pstream::gatherList`.
2. Distribute results with `Pstream::scatterList`.
3. Combine into `SortableList` and sort (`reverseSort`).
4. Select cells in sorted order until reaching `maxCells`.

---

## Additional info stored and written

Expose a new flag in the `dynamicMeshDict` called `dumpRefinementInfo`. If `dumpRefinementInfo` is set to `true` in the `dynamicMeshDict`, the following information will be stored in the registry each refinement step:
- `volScalarField` `field-` -> `expectedCellError`
- `volScalarField` `field+` -> `cellError`
- number of cells selectable for refinement with `cellError > 0` -> `nTotToRefine`
- number of cells selected for refinement -> `nTotRefined`
- maximum `cellError` value among selected cells -> `upperLimit`
- minimum `cellError` value among selected cells -> `lowerLimit`

    mydynamicRefineFvMeshCoeffs
    {
        // ...
        dumpRefinementInfo true;
    }

---

## Startup Smoothing

Expose a time-dependent parameter $γ(t)∈[0,1]γ(t)∈[0,1]$ in the `dynamicMeshDict` called `refineScale` (*Funtion1*). At each step, scale the target number of cells to refine by $γ(t)$. Early in the run $γ≈0 ⇒$ very few (or none) are added; later $γ→1 ⇒$ full refinement kicks in.

The new parameter prevents sudden mesh explosions at the beginning of a run; new cells are introduced smoothly as its value rises.

    mydynamicRefineFvMeshCoeffs
    {
        // ...
        refineScale
        {
            type        table;      // any Function1
            outOfBounds clamp;
            values      ((0 0) (0.1 0.3) (0.25 1));
        }
    }
