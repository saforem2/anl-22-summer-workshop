# Install DeepHyper

As this procedure needs to be performed on ThetaGPU, we will directly
execute it in this `job-install-dhenv.sh` submission script (replace the
`$PROJECT_NAME` with the name of your project allocation, e-g:
`#COBALT -A datascience`):

```bash
#!/bin/bash
#COBALT -q single-gpu
#COBALT -n 1
#COBALT -t 40
#COBALT -A $PROJECT_NAME
#COBALT --attrs filesystems=home,theta-fs0,grand
#COBALT -O job-install-dhenv

. /etc/profile

# create the dhgpu environment:
module load conda/2022-07-01

mkdir build && cd build
conda create -p dhenv --clone base -y
conda activate dhenv/

# install DeepHyper in the previously created dhgpu environment:
pip install pip --upgrade
pip install deephyper["analytics"]

# install mpi4py in the previously created dhgpu environment:
git clone https://github.com/mpi4py/mpi4py.git
cd mpi4py/
MPICC=mpicc python setup.py install

# others
pip install progressbar2
```

Submitting it with the following command :

```bash
$ qsub-gpu job-install-dhenv.sh
```
