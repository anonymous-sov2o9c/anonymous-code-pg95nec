# OmniChainBench

This repository provides the evaluation framework for preparing benchmark inputs and evaluating model outputs consistently across OmniChainBench tasks and experiments.

The framework uses a prepared-data workflow:

1. Convert raw benchmark annotations and videos into reusable prepared sample bundles.
2. Reuse the prepared bundles across model adapters and evaluation runs.
3. Write predictions, structured outputs, scoring records, and aggregate summaries under `artifacts/`.

## What Is Included

- Raw dataset validation
- Chain manifest generation for OracleTrack
- Prepared-data generation for the built-in `main` protocol and importable custom protocols
- Adapter-based model evaluation
- Framework-owned prompt construction, chain-history injection, structuring, judging, scoring, and summaries
- LLM-as-a-judge support through an OpenAI-compatible API
- OracleTrack reruns

The repository does not ship the concrete adapters for the baseline models. New models are connected through the adapter interface described in [docs/ADAPTER_AGENT_GUIDE.md](docs/ADAPTER_AGENT_GUIDE.md).

## Project Layout

Key files and directories:

- [pyproject.toml](pyproject.toml): package metadata and dependencies
- [src/omnichain_eval/cli.py](src/omnichain_eval/cli.py): CLI entrypoint
- [src/omnichain_eval/config.py](src/omnichain_eval/config.py): TOML config loading
- [src/omnichain_eval/dataset.py](src/omnichain_eval/dataset.py): raw data loading and validation
- [src/omnichain_eval/protocols.py](src/omnichain_eval/protocols.py): sampling protocols
- [src/omnichain_eval/prepare.py](src/omnichain_eval/prepare.py): prepared-data cache builder and loader
- [src/omnichain_eval/experiments.py](src/omnichain_eval/experiments.py): experiment orchestration and summaries
- [src/omnichain_eval/adapters/base.py](src/omnichain_eval/adapters/base.py): model adapter interface
- [prompts/benchmark_v1](prompts/benchmark_v1): inference prompt pack
- [prompts/structurer_v1](prompts/structurer_v1): structurer prompt pack
- [prompts/judge_v1](prompts/judge_v1): judge prompt pack
- [configs/examples](configs/examples): example TOML configs for common workflows
- [docs/ADAPTER_AGENT_GUIDE.md](docs/ADAPTER_AGENT_GUIDE.md): adapter implementation guide

Generated directories:

- configured `prepared_root`, for example `/path/to/omnichainbench_prepared/`: prepared sample bundles
- `artifacts/`: runtime predictions, structured outputs, results, and summaries

## Installation

The project uses `uv`.

Basic setup:

```bash
UV_CACHE_DIR=/tmp/uv-cache uv sync
```

For development and tests:

```bash
UV_CACHE_DIR=/tmp/uv-cache uv sync --extra dev
```

Notes:

- `UV_CACHE_DIR=/tmp/uv-cache` is recommended in environments where the default uv cache path may be unwritable.
- Video decoding uses PyAV. If PyAV is unavailable, `prepare-data` fails loudly.

## Dataset Assumptions

The dataset is available on Hugging Face:
[anonymous-1nd83bu/anonymous-dataset-c73l5hv](https://huggingface.co/datasets/anonymous-1nd83bu/anonymous-dataset-c73l5hv).

The framework expects a raw dataset root configured through TOML. In the server examples used in this repository, that root is:

```text
/path/to/omnichainbench
```

At minimum, the raw dataset should contain:

- annotation files like `<data_root>/<sport>/<event>/<video_id>.json`
- videos alongside them as `<data_root>/<sport>/<event>/<video_id>.mp4`
- tracking files in `mot/*.txt`

Each annotation becomes a stable sample id:

```text
<sport>/<event>/<video_id>#<annotation_id>
```

Example:

```text
3x3_Basketball/Men/1#4
```

## CLI

The package installs one CLI:

```bash
uv run omnichain-eval <command> --config <path/to/config.toml>
```

Supported commands:

- `validate-data`
- `build-chain-manifest`
- `prepare-data`
- `run-eval`

All operational parameters are configured through TOML. Each command reads the section it needs from the TOML file passed with `--config`.

## Configuration

Example configs are provided under [configs/examples](configs/examples):

- [workflow.toml](configs/examples/workflow.toml): raw-data validation and chain-manifest generation
- [prepare_main.toml](configs/examples/prepare_main.toml): prepared-data generation for the built-in `main` protocol
- [prepare_custom_protocol.toml](configs/examples/prepare_custom_protocol.toml): prepared-data generation for an importable custom protocol
- [run_eval_adapter.toml](configs/examples/run_eval_adapter.toml): mock smoke-test evaluation
- [run_eval_main.toml](configs/examples/run_eval_main.toml): live `run-eval` example for `main`
- [run_eval_custom_protocol.toml](configs/examples/run_eval_custom_protocol.toml): live `run-eval` example for an importable custom protocol

Supported top-level TOML sections:

- `[validate_data]`
- `[build_chain_manifest]`
- `[prepare_data]`
- `[run_eval]`
- `[structurer]`
- `[judge]`

Minimal `run-eval` shape:

```toml
[run_eval]
prepared_root = "/path/to/omnichainbench_prepared"
protocol = "main"
artifacts_root = "artifacts/runs"
prompt_root = "prompts/benchmark_v1"
adapter = "your_package.adapters.video:YourVideoAdapter"
chain_manifest = "artifacts/chain_pairs.jsonl"

[structurer]
backend = "openai"
prompt_root = "prompts/structurer_v1"
base_url = "https://dashscope.aliyuncs.com/compatible-mode/v1"
api_key_env = "DASHSCOPE_API_KEY"
model = "qwen3.5-397b-a17b"
temperature = 0
invalid_json_retries = 2

[structurer.extra_body]
enable_thinking = false

[judge]
backend = "openai"
prompt_root = "prompts/judge_v1"
base_url = "https://dashscope.aliyuncs.com/compatible-mode/v1"
api_key_env = "DASHSCOPE_API_KEY"
model = "qwen3.5-397b-a17b"
temperature = 0
invalid_json_retries = 2

[judge.extra_body]
enable_thinking = false
```

Path rules:

- relative paths are resolved relative to the TOML file location
- absolute paths are supported
- `protocol` and `protocols` accept either a built-in id such as `main` or an importable Python spec such as `your_package.protocols:EightFrameUniformProtocol`
- secrets can stay out of TOML by using `api_key_env`

For local smoke tests, use the mock adapter and static backends shown in [run_eval_adapter.toml](configs/examples/run_eval_adapter.toml).

## Typical Workflow

Validate the raw dataset:

```bash
UV_CACHE_DIR=/tmp/uv-cache uv run omnichain-eval \
  validate-data \
  --config configs/examples/workflow.toml
```

Build the OracleTrack chain manifest:

```bash
UV_CACHE_DIR=/tmp/uv-cache uv run omnichain-eval \
  build-chain-manifest \
  --config configs/examples/workflow.toml
```

Prepare reusable test data for the built-in `main` protocol:

```bash
UV_CACHE_DIR=/tmp/uv-cache uv run omnichain-eval \
  prepare-data \
  --config configs/examples/prepare_main.toml
```

Run a smoke evaluation with the built-in mock adapter:

```bash
UV_CACHE_DIR=/tmp/uv-cache uv run omnichain-eval \
  run-eval \
  --config configs/examples/run_eval_adapter.toml
```

For a real model, implement an adapter and point `[run_eval].adapter` at `module.path:ClassName`. See [docs/ADAPTER_AGENT_GUIDE.md](docs/ADAPTER_AGENT_GUIDE.md) for the adapter contract.

## Prepared Data And Protocols

Prepared data is stored by protocol under `prepared_root`:

```text
<prepared_root>/
  <protocol_id>/
    build_manifest.json
    index.jsonl
    stats.json
    samples/
      ...
```

The built-in `main` protocol is the default OmniChainBench evaluation protocol. If a model needs a native sampling rule, implement a `BaseProtocol` subclass and reference it from TOML:

```toml
[prepare_data]
protocols = ["your_package.protocols:EightFrameUniformProtocol"]

[run_eval]
protocol = "your_package.protocols:EightFrameUniformProtocol"
```

`prepare-data` and `run-eval` must use the same protocol. Mismatches fail and require rebuilding prepared data for the desired protocol.

## Model Adapters

Evaluation is adapter-based. `[run_eval].adapter` accepts:

- `mock`
- `module.path:ClassName`

Adapters receive a framework-built `ModelInput` containing rendered prompt messages and prepared media paths. The adapter should call the model and return the raw model answer as a string. Prompt rendering, chain history, structuring, judging, scoring, resume, and summary generation remain framework-owned.

To quickly connect a target model with a Coding Agent, give the agent [docs/ADAPTER_AGENT_GUIDE.md](docs/ADAPTER_AGENT_GUIDE.md) as the implementation handoff. The guide tells the agent which files to inspect, what adapter contract to implement, how to consume prepared frames or sampled videos, and how to configure `run-eval` for the new adapter. In practice, the agent should only add an importable `BaseModelAdapter` subclass and a matching TOML config; the framework handles the rest.

## Outputs

`run-eval` writes a run directory under:

```text
artifacts/runs/<timestamp>_<model_name>_<protocol_id>/
```

Important files include:

- `run.log`
- `predictions.jsonl`
- `structured_predictions.jsonl`
- `results.jsonl`
- `chain_predictions.jsonl`
- `chain_structured_predictions.jsonl`
- `chain_results.jsonl`
- `summary.json`

`run-eval` is resumable from these artifact files. Rerunning the same config skips completed samples and retries unfinished prediction, structuring, judging, or scoring work.

## Main Experiment And OracleTrack

The main experiment runs on prepared data for the built-in `main` protocol:

```bash
UV_CACHE_DIR=/tmp/uv-cache uv run omnichain-eval \
  run-eval \
  --config configs/examples/run_eval_main.toml
```

OracleTrack requires a chain manifest:

```toml
[run_eval]
chain_manifest = "artifacts/chain_pairs.jsonl"
```

OracleTrack is framework-owned. When enabled, the runner performs language, visual, and language-visual Oracle reruns and reports the corresponding Oracle metrics in `summary.json`.

For OracleTrack, set:

```toml
[run_eval]
enable_oracle_track = true
oracle_prompt_root = "prompts/benchmark_oracle_v1"

[structurer]
oracle_prompt_root = "prompts/structurer_oracle_v1"
```

## Running Tests

```bash
UV_CACHE_DIR=/tmp/uv-cache uv run ruff check .
UV_CACHE_DIR=/tmp/uv-cache uv run pytest
```

The test suite covers protocol sampling, chain manifest generation, prepared-data building, mock evaluation, structurer validation, resumability, and main experiment plus OracleTrack summaries.
