# 2026-07-08 · Transient 1D PINN — Design, Convergence, and Gotchas

**Tags:** `#pinn` `#transient` `#heat-equation` `#causality` `#pytorch` `#jupyter`

**Related repo:** [pinn-gan-selfheating](https://github.com/siddhanta-roy/pinn-gan-selfheating)
**Notebook:** [`notebooks/02_transient_heat.ipynb`](https://github.com/siddhanta-roy/pinn-gan-selfheating)
**View on nbviewer** (for the animation): https://nbviewer.org/github/siddhanta-roy/pinn-gan-selfheating/blob/main/notebooks/02_transient_heat.ipynb

---

## Problem

Extend a validated 1D steady-state PINN skeleton to the transient heat equation:

$$\frac{\partial T}{\partial t} = \alpha \frac{\partial^2 T}{\partial x^2}, \quad x \in [0, 1], \ t \in [0, t_{\text{final}}]$$

with Dirichlet BCs $T(0, t) = T(1, t) = 0$ and initial condition $T(x, 0) = \sin(\pi x)$.

**Analytical benchmark:** $T(x, t) = \sin(\pi x) \, e^{-\alpha \pi^2 t}$ — the fundamental mode of the heat equation, exact for MMS-style validation.

**Setup:** $\alpha = 1.0$, $t_{\text{final}} = 0.2$, 2-input MLP (32 hidden, 4 layers, tanh), Adam @ lr = 1e-3, 5000 epochs.

---

## Key design decisions

### IC as a soft loss, not a hard-coded ansatz

- The output surrogate spans the whole $(x, t)$ strip — there's no time-stepping, so the IC has to be enforced globally like the BCs.
- Could use an ansatz like $T(x, t) = T_0(x) + t \cdot \mathcal{N}(x, t)$ to hard-encode the IC (eliminates one loss term, often converges faster).
- **Chose soft IC** because the code stays generic for real GaN problems where the IC/BC geometry gets ugly. One rung at a time.

### Loss weights: `w_pde = 1, w_bc = 10, w_ic = 10`

- Empirically balances the three components. IC and BC are ~10× overweighted vs. PDE residual.
- The IC overweighting is a **causality mitigation**: without it, the network can fit the interior residual before it fits $t = 0$, giving low aggregate loss but drift from the IC.
- No adaptive weighting (Wang et al. NTK-based) yet — manual first for intuition.

### Random spatio-temporal collocation, re-sampled every epoch

- PDE / BC / IC points drawn from `torch.rand` each step; 4000 / 200 / 200 per epoch.
- Acts as implicit data augmentation; avoids overfitting to a fixed grid.
- Cost is trivial (`torch.rand` is essentially free).

### Causality anchoring (the transient-PINN trap)

- Standard PINN training is acausal — residual minimized everywhere at once.
- Symptom to watch: residual low mid-domain, high near $t = 0$; L2 error grows with $t$.
- **For $t_{\text{final}} = 0.2$, IC overweighting alone was enough.** For $t_{\text{final}} \geq 1.0$, would need curriculum learning or explicit causal weighting (Wang et al. 2022).

---

## What worked — convergence trajectory

Ran 5000 epochs on CPU (fc8engele01 @ GF), ~3–5 min wall time.

| Epoch | Total | PDE      | BC       | IC       |
|-------|-------|----------|----------|----------|
| 0     | 4.470e+00 | 8.01e-02 | 6.81e-02 | 3.71e-01 |
| 500   | 8.309e-01 | 6.48e-02 | 4.92e-02 | 2.74e-02 |
| 1000  | 9.348e-02 | 4.07e-02 | 3.40e-03 | 1.88e-03 |
| 2000  | 1.116e-02 | 6.63e-03 | 3.67e-04 | 8.59e-05 |
| 3000  | 2.403e-03 | 2.15e-03 | 1.43e-05 | 1.07e-05 |
| 4999  | 1.271e-02 | 1.12e-03 | 5.67e-04 | 5.92e-04 |

**Final L2 error over the $(x, t)$ strip: `1.807e-02`.**

Observations:
- Total loss dropped **~350×** over training.
- **IC dropped fastest** (0.37 → 5.9e-4, 630×), confirming the anchoring strategy worked.
- Final composite loss is balanced across the three terms — no pathological imbalance.

## What didn't (would explore later)

- Didn't try `w_ic = 20` or `w_ic = 50` — visual overlay at $t = 0$ shows a ~2% amplitude gap that a bigger IC weight would probably close.
- Didn't try longer training (10k epochs); returns diminishing given the fixed collocation budget.
- Didn't try 64-neuron / 5-layer MLP — the 32/4 was enough for L2 ~2e-2, but for L2 ~1e-3 would probably need more capacity.

---

## Reference implementation

- Repo: [siddhanta-roy/pinn-gan-selfheating](https://github.com/siddhanta-roy/pinn-gan-selfheating)
- Key files:
  - `src/model.py` — 2-input MLP, `in_dim` parameterized so day-1 (steady) still works
  - `src/pde_loss.py` — `transient_residual`, `bc_loss_transient`, `ic_loss`
  - `src/train.py` — `train_transient`, `sample_collocation`, `analytical_solution`, `l2_error`
  - `notebooks/02_transient_heat.ipynb` — driver + animated $T(x, t)$
  - `tests/test_transient.py` — 6 regression tests including a 300-epoch smoke run

---

## When to reuse this

Applicable when:
- **1D parabolic PDE** with Dirichlet BCs and a smooth IC.
- IC/BC geometry is simple enough that soft constraints are fine (no ansatz needed).
- $t_{\text{final}}$ is small enough that IC anchoring alone handles causality — for this problem, $\alpha \pi^2 t_{\text{final}} \sim 2$, so the solution decays to $\sim 14\%$ within the window.

For extensions:
- **Neumann BCs** → replace `bc_loss_transient` with a flux form (autograd $T_x$ at the boundary, compare to prescribed flux).
- **Non-zero IC** → change `torch.sin(math.pi * x)` in `ic_loss` to whatever $T_0(x)$ is.
- **$t_{\text{final}} \geq 1.0$** → add curriculum learning (train on $[0, 0.02]$, extend to $[0, 0.05]$, etc.) or Wang-style causal weighting.
- **Higher-order PDE** (e.g., Kuramoto-Sivashinsky) → extend the autograd chain in `transient_residual` accordingly.

---

## Gotchas discovered today

### 1. Notebook committed as 0 bytes

- **Symptom:** GitHub showed `0 Bytes`, notebook wouldn't render, but VS Code opened it fine locally.
- **Cause:** `git add` ran before VS Code had flushed the notebook to disk. The commit landed successfully with an empty payload.
- **Why CI didn't catch it:** pytest doesn't load notebooks; the test surface is only `src/` and `tests/`. Green CI ≠ working notebook.
- **Fix:** Recommit after explicit save. Confirm with `git ls-tree -l HEAD notebooks/02_transient_heat.ipynb` — the size in bytes is what got committed, not what's on disk.
- **Prevention:** Enable VS Code `"files.autoSave": "onFocusChange"`. Or run `git diff --cached --stat` before every commit to sanity-check staged bytes.

### 2. GitHub strips notebook animation JavaScript

- `matplotlib.animation.FuncAnimation.to_jshtml()` outputs a JavaScript player. GitHub's static renderer strips scripts for security.
- **Fix:** Link to https://nbviewer.org in the README. nbviewer executes the JavaScript and renders the animation correctly.
- **General principle:** For any interactive notebook content (widgets, plotly, animations), always add an nbviewer or Colab link.

### 3. Copy-paste corruption from chat → editor

Pattern observed multiple times in one session: character sequences mixing punctuation with alphanumerics get silently mangled when pasting from chat output into VS Code.

- Examples: `t_np.3f` → `t_np.3f`; URLs like `nbviewer.org/github/user/repo` → `nbviewer.org/.../02_transient_heatuser/repo`.
- **Workarounds:**
  - Paste through a plain-text intermediary (e.g., `nano`, Notepad, or a `.txt` scratch file) and visually verify.
  - Prefer old-style `%`-formatting over f-strings for animation title text (`"t = %.3f" % t_val` beats `f"t = {t_np.3f}"`).
  - For URLs, click the resulting link before committing.

### 4. "Editor view ≠ disk ≠ commit ≠ remote"

Four independent states, four independent save gates. Bit me three times today:
- Editor shows correct content, but Ctrl+S wasn't pressed → disk is stale.
- Disk is correct, but `git add` + `git commit` weren't run → repo is stale.
- Repo is committed, but `git push` wasn't run → GitHub is stale.
- GitHub has the latest push, but browser cached the previous version → what visitors see is stale.

**Habit:** After every "fix," verify at the *destination* level you care about. `git diff` (disk → commit), `git log origin/main..main` (commit → remote), hard-refresh browser (remote → visitor).

---

## Next milestone

**Temperature-dependent thermal conductivity: $k(T) = k_0 (300/T)^{1.4}$** — introduces the first genuine nonlinearity into the PDE. Expected gotchas:
- Log-transform tricks to keep gradients stable when $T$ is small.
- Loss landscape becomes non-convex — Adam may need warm-up or lower initial LR.
- Analytical benchmark no longer available for validation — will need MMS (method of manufactured solutions) with a hand-picked $T(x, t)$ and its induced source term.

Target: 1D nonlinear transient, MMS-validated, L2 < 5e-3.

---

*Written 2026-07-08 by Siddhanta Roy, GlobalFoundries Bangalore.*
