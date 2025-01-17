# ThetaGPU (ALCF)

[ThetaGPU](https://www.alcf.anl.gov/theta) is an extension of Theta and
is comprised of 24 NVIDIA DGX A100 nodes at Argonne Leadership Computing
Facility (ALCF). See the
[documentation](https://argonne-lcf.github.io/ThetaGPU-Docs/) of
ThetaGPU from the Datascience group at Argonne National Laboratory for
more information. The system documentation from the ALCF can be accessed
[here](https://www.alcf.anl.gov/support-center/theta-gpu-nodes/getting-started-thetagpu).

## Connect to Theta

To connect to Theta via terminal, use:

```bash
$ ssh <username>@theta.alcf.anl.gov
```

Submitting jobs on ThetaGPU is then done from there using the command
(make sure `<submission_script>.sh` is executable by performing
`chmod +x <submission_script>.sh`):

```bash
$ qsub-gpu <submission_script>.sh
```

## DeepHyper Installation

> **Note**: `DeepHyper` comes pre-installed in the most recent `conda` environment.
>
> We can activate this environment by first loading the `conda/2022-07-01`
> module, and activating the `base` environment via:
>
> ```bash
> $ module load conda/2022-07-01
> $ conda activate base
> $ which python3
> /lus/theta-fs0/software/thetagpu/conda/2022-07-01/mconda3/bin/python3
> $ python3 -c 'import deephyper; print(deephyper.__version__)'
> 0.4.2
> $ python3 -c 'import deephyper; print(deephyper.__file__)'
> /lus/theta-fs0/software/thetagpu/conda/2022-07-01/mconda3/lib/python3.8/site-packages/deephyper/__init__.py
> ```

For detailed instructions on how to build and install `DeepHyper` from source, refer to [install DeepHyper](./install-deephyper.md).


## HPS Definition

We will be using the problem from the [first notebook of the
workshop](https://github.com/deephyper/anl-22-summer-workshop/blob/main/notebooks/1-Hyperparameter-Search.ipynb), optimizing a surrogate geophysical model from data.

In this `search.py` script we define the hyperparameter search
space as well as run function, and feed it to a search instance
using an MPI-based evaluator.

Note that you need to copy
[utils.py](https://github.com/deephyper/anl-22-summer-workshop/blob/main/scripts/ALCF-ThetaGPU/utils.py)
in the current directory for this script to work.


```python
import os
os.environ["TF_CPP_MIN_LOG_LEVEL"] = "3"
import gzip
import numpy as np
from utils import load_data_prepared
from deephyper.nas.metrics import r2, mse
import mpi4py

mpi4py.rc.initialize = False
mpi4py.rc.threads = True
mpi4py.rc.thread_level = "multiple"

from mpi4py import MPI

if not MPI.Is_initialized():
    MPI.Init_thread()

comm = MPI.COMM_WORLD
# GLOBAL_RANK: [0, 1, ..., N_ranks]
RANK = comm.Get_rank()

import tensorflow as tf
gpus = tf.config.list_physical_devices("GPU")
# LOCAL RANK (index): [0, 1, ..., len(gpus)]
IDX = (RANK % len(gpus))
if gpus:
    # Attribute 1 GPU to each rank
    try:
        tf.config.set_visible_devices(gpus[IDX], "GPU")
        tf.config.experimental.set_memory_growth(gpus[IDX], True)
        logical_gpus = tf.config.list_logical_devices("GPU")
    except RuntimeError as e:
        # Visible devices must be set before GPUs have been initialized
        print(f"{e}") 

from deephyper.problem import HpProblem
from deephyper.search.hps import CBO
from deephyper.evaluator import Evaluator

n_components = 5
# Only load data from RANK 0
if IDX == 0:
    load_data_prepared(
        n_components=n_components
    )


# Baseline LSTM Model
def build_and_train_model(
        config: dict,
        n_components: int = 5,
        verbose: bool = 0
):
    tf.keras.utils.set_random_seed(42)

    default_config = {
        "lstm_units": 128,
        "activation": "tanh",
        "recurrent_activation": "sigmoid",
        "learning_rate": 1e-3,
        "batch_size": 64,
        "dropout_rate": 0,
        "num_layers": 1,
        "epochs": 20,
    }
    default_config.update(config)

    (X_train, y_train), (X_valid, y_valid), _, _ = load_data_prepared(
        n_components=n_components
    )

    layers = []
    for _ in range(default_config["num_layers"]):
        lstm_layer = tf.keras.layers.LSTM(
            default_config["lstm_units"],
            activation=default_config["activation"],
            recurrent_activation=default_config["recurrent_activation"],
            return_sequences=True,
        )
        dropout_layer = tf.keras.layers.Dropout(default_config["dropout_rate"])
        layers.extend([lstm_layer, dropout_layer])

    model = tf.keras.Sequential(
        [tf.keras.Input(shape=X_train.shape[1:])]
        + layers
        + [tf.keras.layers.Dense(n_components)]
    )

    if verbose:
        model.summary()

    optimizer = tf.keras.optimizers.Adam(learning_rate=default_config["learning_rate"])
    model.compile(optimizer, "mse", metrics=[])

    history = model.fit(
        X_train,
        y_train,
        epochs=default_config["epochs"],
        batch_size=default_config["batch_size"],
        validation_data=(X_valid, y_valid),
        verbose=verbose,
    ).history

    return model, history


def filter_failures(df):
    if df.objective.dtype != np.float64:
        df = df[~df.objective.str.startswith("F")]
        df = df.astype({"objective": float})
    return df


# ---- Hyperparameter search space definition ----------------------------------------
problem = HpProblem()
problem.add_hyperparameter((10, 256), "units", default_value=128)
problem.add_hyperparameter(["sigmoid", "tanh", "relu"], "activation", default_value="tanh")
problem.add_hyperparameter(["sigmoid", "tanh", "relu"], "recurrent_activation", default_value="sigmoid")
problem.add_hyperparameter((1e-5, 1e-2, "log-uniform"), "learning_rate", default_value=1e-3)
problem.add_hyperparameter((2, 64), "batch_size", default_value=64)
problem.add_hyperparameter((0.0, 0.5), "dropout_rate", default_value=0.0)
problem.add_hyperparameter((1, 3), "num_layers", default_value=1)
problem.add_hyperparameter((10, 100), "epochs", default_value=20)

# ---- Definition of the function to optimize (configurable model to train) ----------
def run(config):
    # important to avoid memory exploision
    tf.keras.backend.clear_session()

    _, history = build_and_train_model(config, n_components=n_components, verbose=0)

    return -history["val_loss"][-1]


# ---- Definition of an MPI Evaluator xecution of a Bayesian optimization search -----
if __name__ == "__main__":
    with Evaluator.create(
            run,
            method="mpicomm",
        ) as evaluator:
            if evaluator is not None:
                print(f"Creation of the Evaluator done with {evaluator.num_workers} worker(s)")

                # Search creation
                print("Creation of the search instance...")
                search = CBO(
                    problem,
                    evaluator,
                    initial_points=[problem.default_configuration],
                    log_dir="cbo-results",
                    random_state=42
                )
                print("Creation of the search done")

                # Search execution
                print("Starting the search...")
                results = search.search(timeout=540)
                print("Search is done")

                results.to_csv(os.path.join("cbo-results", "results.csv"))

                results = filter_failures(results)

                i_max = results.objective.argmax()
                best_config = results.iloc[i_max][:-4].to_dict()

                best_model, best_history = build_and_train_model(
                    best_config,
                    n_components=n_components,
                    verbose=1
                )

                scores = {"MSE": mse, "R2": r2}

                (X_train, y_train), (X_valid, y_valid), (X_test, y_test), _ = (
                    load_data_prepared(
                        n_components=n_components
                    )
                )

                for metric_name, metric_func in scores.items():
                    print(f"Metric {metric_name}")
                    y_pred = best_model.predict(X_train)
                    score_train = np.mean(metric_func(y_train, y_pred).numpy())

                    y_pred = best_model.predict(X_valid)
                    score_valid = np.mean(metric_func(y_valid, y_pred).numpy())

                    y_pred = best_model.predict(X_test)
                    score_test = np.mean(metric_func(y_test, y_pred).numpy())

                    print(f"train: {score_train:.4f}")
                    print(f"valid: {score_valid:.4f}")
                    print(f"test : {score_test:.4f}")
```

## Executing the Search on ThetaGPU

With the evaluator using MPI, we can simply use `mpirun` on ThetaGPU to
launch it on all the gpus of every allocated node. This is what is done
in this `job-run-hps.sh` submission script (replace the `$PROJECT_NAME`
with the name of your project allocation, e-g: `#COBALT -A datascience`)
:

```bash
#!/bin/bash
#COBALT -q full-node
#COBALT -n 1
#COBALT -t 60
#COBALT -A $PROJECT_NAME
#COBALT --attrs filesystems=home,grand,eagle,theta-fs0
#COBALT -O job-run-hps

# Nodes Configuration
COBALT_JOBSIZE=$(wc -l < ${COBALT_NODEFILE})
RANKS_PER_NODE=$(nvidia-smi -L | wc -l)
NGPUS=$(( ${COBALT_JOBSIZE} * ${RANKS_PER_NODE} ))

# Initialization of environment
. /etc/profile
# Tensorflow optimized for A100 with CUDA 11
module load conda/2022-07-01
conda activate base

# Execute python script
mpirun \
  -x LD_LIBRARY_PATH \
  -x PYTHONPATH \
  -x PATH \
  -n ${NGPUS} \
  -N $RANKS_PER_NODE \
  --hostfile $COBALT_NODEFILE \
  python search.py
```

If you want to set the number of allocated nodes for the job to `k`,
make sure to change accordingly these two lines :

```bash
#COBALT -n k
COBALT_JOBSIZE=k
```
