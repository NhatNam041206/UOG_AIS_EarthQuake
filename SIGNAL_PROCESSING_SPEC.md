# Signal Processing Specification (LFSP)

This specification formalizes how LFSP transforms low-rate acceleration records into model-ready 100 Hz segments.

## 1. Input channels and physical meaning

Required channels (priority):
1. `AZ1(m/s2)`
2. `AX1(m/s2)`
3. `AY1(m/s2)`

Time axis: `Times(s)` sampled at nominal `Δt = 0.2 s` (5 Hz).
Gap threshold constant: `gap_threshold_s` (default `0.2`).

## 2. Transduction logic (mV → m/s²)

If sensors provide voltage instead of calibrated acceleration, convert before interpolation:

\[
a(t)=\frac{V(t)-V_0}{S\cdot G}
\]

Where:
- \(V(t)\): measured voltage (mV or V)
- \(V_0\): sensor zero-offset
- \(S\): sensitivity (V/g or mV/g)
- \(G = 9.80665\,m/s^2\)

If `AZ1/AX1/AY1` already carry SI units (`m/s²`), conversion is skipped.
To prevent a `×1000` scaling error, input units must be explicit in metadata/config (`voltage_unit: mV|V`) and sensitivity must use matching units.

## 3. Gap detection and handling

Given time vector \(t_i\), compute:

\[
\Delta t_i = t_i - t_{i-1}
\]

A **Data Gap** is flagged when:

\[
\Delta t_i > \texttt{gap\_threshold\_s}
\]

Policies:
- **Pad mode:** insert zero-valued samples to preserve absolute time continuity.
- **Split mode:** terminate current segment and start a new trace after the gap.

Each gap event is logged with start index, duration, and policy outcome.

## 4. Recommended processing order

Source grid:
- \(f_s^{in}=5\,Hz\), \(\Delta t^{in}=0.2\,s\)

Target grid:
- \(f_s^{out}=100\,Hz\), \(\Delta t^{out}=0.01\,s\)

Preferred order per trace/channel group:
1. gap handling
2. detrend at native rate
3. bandpass at native rate (`1.0–2.4 Hz`)
4. cubic-spline upsampling to `100 Hz`
5. z-score normalization

Filtering at native 5 Hz minimizes interpolation-induced high-frequency artifacts and reduces unnecessary compute.

## 5. Upsampling with cubic spline (5 Hz → 100 Hz)

For each channel independently, construct a cubic spline \(s(t)\) from valid samples and evaluate on the dense grid.
Implementation requirement: use `scipy.interpolate.CubicSpline` to keep interpolation behavior deterministic across runs/environments.

## 6. Detrending

Apply linear detrend per channel:

\[
x_d(t)=x(t)-\hat{x}_{trend}(t)
\]

This suppresses baseline drift and improves filter/model stability.

## 7. Bandpass filtering constraints

At the original 5 Hz rate, Nyquist is:

\[
f_N = \frac{f_s}{2}=2.5\,Hz
\]

The chosen passband `1.0–2.4 Hz` remains below Nyquist and emphasizes teleseismic-relevant low-frequency content. Filtering should be applied with a stable zero-phase implementation (e.g., Butterworth + `filtfilt`) to avoid phase distortion.

## 8. Normalization (z-score)

For each channel/window:

\[
z_i=\frac{x_i-\mu}{\sigma+\epsilon}
\]

Where:
- \(\mu\): channel mean
- \(\sigma\): channel standard deviation
- \(\epsilon\): small constant (e.g., `1e-8`) for numerical safety

## 9. Segmentation and export shape

Windows are fixed to 60 s at 100 Hz:
- `6000` samples per channel
- `3` channels total
- final tensor shape `(6000, 3)` and dtype `float32`

These windows are written to HDF5 under `/data/<trace_name>` for SeisBench compatibility.

## 10. Required dependency stack

- `obspy`
- `seisbench`
- `h5py`
- `pandas`
- `scipy`
