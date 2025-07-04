# 🌬️ anemoi-sandbox

A sandbox repository containing instructions and notebooks to train, evaluate, and use ML models using [Anemoi](https://www.ecmwf.int/en/about/media-centre/news/2024/anemoi-new-framework-weather-forecasting-based-machine-learning).

---

## 📚 Table of Contents

1. [Setup Environment](#setup-environment)  
2. [Config Archive](#config-archive)  
3. [Explore Anemoi Documentation](#explore-anemoi-documentation)  
4. [Example Training Run: GraphCast Global Model](#example-training-run-graphcast-global-model)  
5. [Run Lightweight Verification](#run-lightweight-verification)

---

## Setup Environment

Download and install [miniforge](https://github.com/conda-forge/miniforge), a minimal install for Conda and Mamba specific to conda-forge.

### Training environment

Create the environment:

```bash
conda create -n anemoi python=3.10 -y
conda activate anemoi
pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu121
```

**Developers:**

```bash
pip install anemoi-utils

git clone git@github.com:ecmwf/anemoi-datasets.git
cd anemoi-datasets 
pip install -e .
cd ..

git clone git@github.com:ecmwf/anemoi-core.git
cd anemoi-core

cd graphs && pip install -e . && cd ..
cd models && pip install -e . && cd ..
cd training && pip install -e . && cd ..
cd ..

git clone git@github.com:ecmwf/anemoi-inference.git
cd anemoi-inference
pip install -e .
cd ..
```

### Legacy environment

The following environment has to be created in order to run training and inference with older versions of anemoi.

Create the environment:

```bash
conda create -n anemoi-legacy python=3.10 -y
conda activate anemoi-legacy

pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu121

pip install anemoi-datasets==0.5.23
pip install anemoi-graphs==0.4.4
pip install anemoi-inference==0.5.4
pip install anemoi-models==0.4.2
pip install anemoi-training==0.3.2
pip install anemoi-transform==0.1.10
pip install anemoi-utils==0.4.22

pip install eccodes-cosmo-resources-python
conda env config vars set ECCODES_DEFINITION_PATH=$CONDA_PREFIX/share/eccodes-cosmo-resources/definitions
```

Alternatively, refer to [anemoi-config](https://github.com/ecmwf/anemoi-configs).

---

## Config Archive

Anemoi relies on YAML configuration files to define pipelines.

You can find a full archive of MeteoSwiss use cases:

```bash
git clone git@github.com:MeteoSwiss/ml-pilot-anemoi.git
```
Please refer to [Hydra Introduction](hydra_intro.md) for more details about hydra configurations.

---

## Explore Anemoi Documentation

You can have a look at all available CLI command within anemoi in each documentation section:

- 📦 [anemoi-datasets](https://anemoi.readthedocs.io/projects/datasets/en/latest/cli/introduction.html)  
- 🔧 [anemoi-graphs](https://anemoi.readthedocs.io/projects/graphs/en/latest/cli/introduction.html)  
- 🏋️ [anemoi-training](https://anemoi.readthedocs.io/projects/training/en/latest/user-guide/training.html)  
- 📈 [anemoi-inference](https://anemoi.readthedocs.io/projects/inference/en/latest/cli/introduction.html)

---

## Example Training Run: GraphCast Global Model

Here is an example on how to setup a training run on a HPC using anemoi-training.

---

### 📥 Download Dataset
Download the ERA5 096 dataset from the [anemoi-catalogue](https://anemoi.ecmwf.int/).

```bash
anemoi-datasets copy --resume s3://ml-datasets/aifs-ea-an-oper-0001-mars-o96-1979-2022-6h-v6.zarr .
```

### ✏️ Adjust Config File

Update paths in config file: `configs/stage_A-metno.yaml`.

### 📝 Create SBATCH Script
Create the bash script called `stage_A-metno.sh` to submit the job on the HPC machine.

`stage_A-metno.sh`:

```bash
#!/bin/bash

#SBATCH --job-name="stage_A"
#SBATCH --partition=normal
#SBATCH --nodes=1
#SBATCH --ntasks-per-node=4
#SBATCH --gpus-per-node=4
#SBATCH --cpus-per-task=16
#SBATCH --mem=256G
#SBATCH --time=24:00:00
#SBATCH --no-requeue
#SBATCH --exclusive
#SBATCH --output=stage_A-metno.log
#SBATCH --error=stage_A-metno.err

echo "Running training job on $(hostname) at $(pwd)"

# Activate environment
source /users/apennino/.bashrc
conda activate anemoi

srun -u anemoi-training train \
    --config-dir="configs" \
    --config-name="stage_A-metno"
```

### 🚀 Launch Training
You can now launch the training job by:

```bash
sbatch stage_A-metno.sh
```

Monitor the training progress by checking the queue:

```bash
squeue -u YOUR-USERNAME
```

Or check the `.log` and `.err` files created by the script.

### 🧪 Monitoring Training Evolution

You can check the evolution of the training by checkig progress in the [MLFlow server](https://servicedepl.meteoswiss.ch/mlstore/#/experiments/1?searchFilter=&orderByKey=attributes.start_time&orderByAsc=false&startTime=ALL&lifecycleFilter=Active&modelVersionFilter=All+Runs&datasetsFilter=W10%3D).

You can find your specific run by looking at the values of `experiment_name` and `run_name`.

![MLflow UI Screenshot](resources/mlflow_ui.png)

Please refer to [MLFlow Introduction](mlflow_intro.md) for more details about MLFlow monitoring.


## Run Lightweight Verification

Once your training is completed, anemoi generates a checkpoint file containing all model weights. 
You can find it by looking at the checkpoint path in the config file.

### 📥 Install Verification Tool

Until it is moved to its own repository, you can find the verification tool here:
```
git clone git@github.com:MeteoSwiss/ml-pilot-anemoi.git
cd ml-pilot-anemoi
git checkout ml-forecasting
cd inference
```

### 🚧 Run Verification Script

1. Locate your `run_id` in MLflow.
![MLflow UI Screenshot](resources/mlflow.png)

2. Launch:

```bash
./submit_inference_array.sh 090030db9b0c4d3c8e9f42ef66053e6b -m stage_A-metno
```

---

### 📈 Verification Output

- `verif/`: metrics vs. analysis data  
- `plots/`: images and animations  
  ![Stage A Results](resources/stage_A.gif)
- `grib/`: GRIB files for NWP  
- `raw/`: raw NumPy predictions

---

### 🖼️ Visualize Metrics

Execute the `verification.ipynb` notebook to check the desired metrics and models.
