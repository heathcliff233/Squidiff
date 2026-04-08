# AGENTS.md

Architecture index for the vendor PerturbDiff baseline (repository path: `vendor/Squidiff`, package name: `Squidiff`).

## System Map

PerturbDiff in this vendor snapshot is a conditional diffusion model over gene-expression vectors.

### Global Topology

`control -> data(h5ad) -> conditioning encoder -> diffusion denoiser -> runtime(train/sample) -> artifacts`

## Pipeline Inventory

1. **Control Pipeline**
- Training entrypoint: `train_squidiff.py`.
- Sampling entrypoint: `sample_squidiff.py`.
- Arg defaults and model/diffusion construction live in `Squidiff/script_util.py`.

2. **Data Pipeline**
- Reads `.h5ad` via `scanpy`.
- Converts `adata.X` to dense float tensor batches.
- Required metadata includes `obs['Group']`.
- Optional drug-structure path requires `obs['SMILES']`, `obs['dose']`, and a control `.h5ad`.
- Drug conditions are encoded as Morgan fingerprints scaled by `log10(dose + 1)`.

3. **Conditioning Pipeline**
- `MLPModel` can compute latent condition `z_sem` via `EncoderMLPModel` from `x_start` and optional condition fields.
- Sampling APIs can bypass encoder and inject latent directly with `z_mod`.

4. **Model Pipeline**
- Backbone: timestep-conditioned MLP denoiser (`MLPModel`).
- Blocks: stacked `MLPBlock` layers with timestep embedding and optional latent injection.
- Encoder: `EncoderMLPModel` outputs latent condition used by denoiser blocks.

5. **Diffusion Pipeline**
- Gaussian diffusion objective with configurable beta schedule and timestep respacing.
- Default objective: epsilon prediction MSE.
- Supports DDPM and DDIM sampling loops.

6. **Runtime Pipeline**
- `TrainLoop` handles optimization (`AdamW`), EMA tracking, schedule sampling, logging, and checkpoint writes.
- Training loop is step-based (`lr_anneal_steps`) and does not define a validation pipeline in this repo.

7. **Artifact Contract Pipeline**
- Model checkpoints: `<resume_checkpoint>/model.pt` and EMA variants.
- Optimizer checkpoints: `<logger_dir>/optXXXXXX.pt`.
- Training metrics/logs: logger outputs under `logger_path`.

## Wiring Graphs

### Train (`train_squidiff.py`)
`control(args) -> create_model_and_diffusion -> prepared_data(h5ad loader) -> TrainLoop -> diffusion.training_losses -> optimizer/ema -> checkpoints/logs`

### Sample (`sample_squidiff.py`)
`control(args + model_path) -> create_model_and_diffusion -> load checkpoint -> sample_fn(ddim/ddpm) with z_mod -> predicted expression vectors`

## Source Index

1. `train_squidiff.py`
2. `sample_squidiff.py`
3. `Squidiff/scrna_datasets.py`
4. `Squidiff/script_util.py`
5. `Squidiff/MLPModel.py`
6. `Squidiff/diffusion.py`
7. `Squidiff/train_util.py`
8. `Squidiff/resample.py`
9. `Squidiff/respace.py`

## Operational Notes

- The repository contains both top-level and package-local `train_squidiff.py`; behavior differs slightly between copies.
- Resume/load helpers exist but parameter-resume wiring in `TrainLoop` is partially disabled in current code.
- `use_fp16=True` paths call `model.convert_to_fp16()`, but `MLPModel` does not currently define that method.
- This vendor baseline is not yet wired into ecocell registry/task/data/model plugins.
