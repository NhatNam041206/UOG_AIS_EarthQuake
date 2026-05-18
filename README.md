# Low-Frequency Seismic Pipeline (LFSP)

LFSP is a modular Python pipeline for preparing low-frequency (5 Hz) tri-axial acceleration records for earthquake detection with SeisBench/EQTransformer models that expect 100 Hz waveforms.

## Why this project exists

Most field loggers in structural monitoring provide low-rate acceleration streams (e.g., 5 Hz). EQTransformer-style models are typically trained for higher sample rates (100 Hz). LFSP bridges this mismatch with reproducible DSP preprocessing and SeisBench-ready export formats.

## Clean Architecture (Controller-Service-Repository)

- **Repository (`RawRepository`)**: reads `Times(s)` and physical channels `AZ1(m/s2)`, `AX1(m/s2)`, `AY1(m/s2)` from CSV/Excel.
- **Signal Service (`DSPService`)**: handles gap detection, cubic-spline upsampling (5 Hz → 100 Hz), detrending, bandpass filtering (1.0–2.4 Hz), and normalization.
- **Export Service (`ExportService`)**: writes SeisBench-compliant HDF5 (`/data/<trace_name>`, shape `(6000, 3)`, `float32`) and metadata CSV.
- **Inference Controller (`PipelineController`)**: orchestrates the full workflow and model inference (`instance` / `ethz` weights).

See [ARCHITECTURE.md](ARCHITECTURE.md) for the detailed design.

## Project layout

```text
/src
  /data_access
    raw_repository.py
  /services
    dsp_service.py
    export_service.py
    model_service.py
  /core
    pipeline_controller.py
/config
  pipeline_config.yaml
main.py
README.md
```

## Installation

```bash
python -m venv .venv
source .venv/bin/activate
pip install --upgrade pip
pip install obspy seisbench h5py pandas scipy
```

## Quick start (main entry point)

```bash
python main.py \
  --input ./data/raw_station.csv \
  --station-id ST001 \
  --weights ethz \
  --config ./config/pipeline_config.yaml \
  --output-dir ./outputs
```

Expected outputs:
- `outputs/traces.hdf5`
- `outputs/metadata.csv`

## Data and model assumptions

1. Input channels are prioritized in this order: `AZ1(m/s2)`, `AX1(m/s2)`, `AY1(m/s2)`.
2. Data gaps are detected when `Times(s)` increments exceed **0.2 s**.
3. Each exported trace is a 60-second window at 100 Hz: `6000` samples × `3` channels.
4. Model weights are selected from SeisBench presets (`instance` or `ethz`).

## Future extensions

- Real-time streaming ingestion (Kafka/MQTT + rolling windows).
- Replace file-based repository with SQL/TimeSeries database repositories.
- Transfer learning on regional labeled events using SeisBench finetuning workflows.
- Automated quality-control dashboards for drift, clipping, and gap statistics.

## Additional documentation

- [ARCHITECTURE.md](ARCHITECTURE.md)
- [SIGNAL_PROCESSING_SPEC.md](SIGNAL_PROCESSING_SPEC.md)
- [API_REFERENCE.md](API_REFERENCE.md)
