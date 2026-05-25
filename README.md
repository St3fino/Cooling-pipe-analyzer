# Hydronet

A 1-D incompressible hydraulic pressure-drop tool for **tree-shaped** pipe networks, with a Streamlit wizard UI.

## Features

- **Domain model** — tree network of pipes, fittings, curve components, and pumps.
- **Physics** — Darcy–Weisbach with **Churchill** explicit friction factor (laminar → turbulent).
- **Solver** — bottom-up tree reduction with `HydraulicCurve` (dp(Q) and its inverse via Brent).
- **Three solve modes**:
  - **A**: total `Δp` given `Q`
  - **B**: total `Q` given target `Δp`
  - **C**: pump operating point — `Δp_pump(Q) = Δp_system(Q) + (p_leaf − p_root)`
- **Fluid properties** via CoolProp (optional) or built-in fallback (Water, WaterGlycol50).
- **Project save/load** as JSON and **CSV export** of element / branch diagnostics.
- **Streamlit wizard** that calls only `app/use_cases.py` (no physics in the UI layer).

## MVP scope: trees, not loops

The network **must be a directed tree**: each node except the root has exactly one parent element.
A real closed-loop circuit (where flow returns to the pump) is **not** supported in this MVP.
Use an *equivalent open tree* — replace the closing segment with a leaf node whose pressure is fixed
(typically `p_leaf = 0` for atmospheric, or any positive value for a pressurised vessel).

If the user tries to add an element whose target node already has a parent (which would create a
cycle or a multi-parent topology), the validator rejects it with a clear error.

## Sign conventions

- Flow `Q` is non-negative and oriented from `from_node` to `to_node` of each element.
- `Element.dp(Q, fluid)` returns the pressure **drop** (positive for passive elements).
- `PumpElement.dp(Q, fluid)` returns `-dp_rise(Q, fluid)` — pumps **subtract** from total drop.
- Mode C operating point equation:

  ```
  dp_pump(Q) = dp_network_passive(Q) + (p_leaf_fixed - p_root_fixed)
  ```

  With the defaults `p_root_fixed = 0` and `p_leaf_fixed = 0` this reduces to
  `dp_pump(Q) = dp_network(Q)`. A positive `p_leaf_fixed` represents a pressurised
  destination, so the pump must overcome both network drop **and** back-pressure.

## Installation

```bash
pip install -e .
# Optional CoolProp support
pip install -e ".[coolprop]"
# Dev/test
pip install -e ".[dev]"
```

## Run the wizard

```bash
streamlit run src/hydronet/ui/streamlit_app.py
```

## Run the test suite

```bash
pytest
```

## Demo network (preset in the wizard)

A chain inspired by a typical pump-circuit, **flattened to a tree** (one leaf at atmospheric):

```
[ROOT] → Pump → Pipe → Reducer → Pipe → ThrottleValve → Elbow90 → Pipe → Elbow90 → Pipe → Elbow90 → Elbow90 → [LEAF, p_fixed = 0]
```

In the wizard, click **Load preset example** on step 2 to build it automatically and then run Mode C
to find the pump operating point.

## Repository layout

```
src/hydronet/
  config.py             — module-wide constants (gravity etc.)
  domain/               — fluids, network, elements, libraries
  solver/               — numerics, curve fitting, tree solver
  app/                  — use-cases (Builder, Runner, Exporter) and validation
  io/                   — project JSON, CSV export
  ui/                   — Streamlit wizard
  utils/                — logging, unit helpers
tests/                  — pytest test suite
```

## Notes & limitations

- Tree only (no loops).
- All leaves should share `p_fixed` for the parallel-split case; mixed leaf pressures are accepted but
  the solver treats each branch independently using its own leaf pressure (see code for details).
- Mode C requires the pump element be located at the root (`from_node = root`).
- The fallback fluid table covers `Water` and `WaterGlycol50` with simple parametric/constant models —
  see `domain/fluid.py` for exact formulas. Install CoolProp for higher fidelity.

## License

MIT
