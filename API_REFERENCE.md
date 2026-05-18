# API Reference (`/src`)

> This reference follows an auto-generated style and documents the intended public interfaces for LFSP modules.

## `src/data_access/raw_repository.py`

### `class RawRepository`

Repository (DAO) for seismic tabular sources.

#### `read_file(path: str) -> pandas.DataFrame`
Read CSV/Excel input and return a DataFrame.

#### `extract_channels(df: pandas.DataFrame) -> pandas.DataFrame`
Return standardized columns: `Times(s)`, `AZ1(m/s2)`, `AX1(m/s2)`, `AY1(m/s2)`.
Raises a schema error if required fields are missing.

---

## `src/services/dsp_service.py`

### `class DSPService`

Signal processing service for low-frequency acceleration traces.

#### `detect_gaps(time_s: pandas.Series, threshold: float = 0.2) -> list[dict]`
Identify and report Data Gaps where time increment exceeds threshold.

#### `handle_gaps(df: pandas.DataFrame, mode: str = "pad") -> list[pandas.DataFrame]`
Apply `pad` (zero-fill) or `split` strategy before interpolation.
- `pad`: returns a one-item list with continuity-preserved DataFrame.
- `split`: returns multiple DataFrames, one contiguous segment per gap-delimited block.

#### `resample_cubic(time_s, xyz, fs_out: int = 100) -> tuple[numpy.ndarray, numpy.ndarray]`
Upsample 5 Hz records to 100 Hz using cubic spline interpolation.
Return tuple elements in order: `(time_resampled, xyz_resampled)`.

#### `detrend(xyz) -> numpy.ndarray`
Remove linear trend from each channel.

#### `bandpass(xyz, fs: int = 100, fmin: float = 1.0, fmax: float = 2.4) -> numpy.ndarray`
Apply bandpass filter constrained to low-frequency teleseismic band.

#### `zscore(xyz, eps: float = 1e-8) -> numpy.ndarray`
Apply channel-wise z-score normalization.

#### `segment(xyz, window_s: int = 60, fs: int = 100) -> list[numpy.ndarray]`
Split into fixed windows, each shaped `(6000, 3)`.

---

## `src/services/export_service.py`

### `class ExportService`

Writers for SeisBench-ready datasets.

#### `write_hdf5(segments, out_path: str) -> None`
Write segments to `/data/<trace_name>` as `float32` arrays.

#### `write_metadata(rows: list[dict], out_path: str) -> None`
Create metadata CSV with `trace_name`, `start_time`, `station_id`.

---

## `src/services/model_service.py`

### `class ModelService`

Thin SeisBench model wrapper.

#### `load(weights: str = "instance") -> None`
Load EQTransformer-compatible model weights (`instance` or `ethz`).

#### `predict(batch) -> list[dict]`
Run inference and return detections/picks.

---

## `src/core/pipeline_controller.py`

### `class PipelineController`

Orchestrates repository, DSP, export, and inference services.

#### `run(input_path: str, station_id: str, weights: str, config_path: str, output_dir: str) -> dict`
Execute end-to-end preprocessing, export, and inference workflow.

#### `_build_metadata(segments, station_id: str, start_times) -> list[dict]`
Construct metadata rows corresponding to exported HDF5 keys.

#### `_log_gap_events(gaps: list[dict]) -> None`
Record gap diagnostics for audit and QC.
