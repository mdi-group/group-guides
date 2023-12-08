#Archer2

The documentation for Archer2 can be found via [this link](https://docs.archer2.ac.uk).

## Connecting to ARCHER2

You have to generate a SSH key pair and provide the public part to the system administrators via SAFE. Instructions on how to do so can be found [here](https://docs.archer2.ac.uk/user-guide/connecting/).

It is really worthwhile setting up an ssh `config` file to save time. See the other wiki [entry](https://mdi-group.github.io/group-guides/#hpc/tips/#setting-up-an-ssh-configuration), e.g. with this I just need to type `ssh archer2` (and enter the passwords).

```
Host archer2
    User keeeto
    Hostname login.archer2.ac.uk
    IdentityFile ~/.ssh/rsa_new
```

I would also recommend creating a `.bash_profile` and adding this line for easier navigation: `alias ls='ls -G -ltr --color'`

## Setting up your file system

Archer2 is divided into three main systems: `HOME` for e.g. your own software to compile, `WORK` where you should run production run calculations, and `RDF` which serves as a data backup. `WORK` has no access to `HOME` during runs, so make sure that everything you need for your code to run (such as VASP binaries) is available on `WORK` itself!

You may have to create your own shortcut to work (at least it didn't appear for me): `ln -s /work/e05/e05/keeeto work` (with your own username)

## Compiling a code

Open source packages are available as modules or you can compile your own.

* https://github.com/hpc-uk/build-instructions

## Submitting a job

Different to Archer, the scheduling system on Archer2 is now _slurm_ instead of _pbs_. This means your commands for monitoring and submitting jobs have changed:

* `sinfo` gives you an overview over the current system-wide status of nodes
* `squeue` shows you all jobs on the cluster, `squeue -u username` shows only yours
* `sbatch job.slurm` submits a your `job.slurm` script
* `scancel jobid` aborts the job with the specified ID (given it belonged to you)
* `scontrol hold/release JOBID` holds or releases the job with ID `JOBID`.

#### Example of a job submission script

* Remember that each node has 128 cores.

```
#!/bin/bash

# Slurm job options (job-name, compute nodes, job time)
#SBATCH --job-name=Example_MPI_Job
#SBATCH --time=10:00:00
#SBATCH --nodes=4
#SBATCH --tasks-per-node=128
#SBATCH --cpus-per-task=1

# All of us should be part of e05 with our respective subgroup codes, e.g. for react-wal
#SBATCH --account=<account-id>
#SBATCH --partition=standard
#SBATCH --qos=standard
#SBATCH --export=none


# If you have compiled your binary with non-default modules or libraries, you have to load these as well - a few examples below:
#module -s restore /etc/cray-pe.d/PrgEnv-gnu
#module load cpe/21.09
#module swap gcc gcc/10.2.0
#export LD_LIBRARY_PATH=$CRAY_LD_LIBRARY_PATH:$LD_LIBRARY_PATH

# Set the number of threads to 1
#   This prevents any threaded system libraries from automatically
#   using threading.
export OMP_NUM_THREADS=1

# Launch the parallel job
#   Using 512 MPI processes and 128 MPI processes per node
#   srun picks up the distribution from the sbatch options

# If you want to distribute tasks as evenly on nodes as possible in case memory could be a problem, I've found this to be best for FHI-aims:
srun --cpu-bind=cores --hint=nomultithread /work/e05/e05/mat92/Codes/myexe.x > output.out

# If memory isn't a problem, this setup should have the lowest memory communication overhead:
srun --cpu-bind=rank --distribution=block:block --hint=nomultithread /work/e05/e05/mat92/Codes/myexe.x > output.out
```

#### Example of an array job submission script
If you want to submit a large number of similar jobs then it may be easier to submit them as an array job. The following script runs 56 subjobs, with the subjob index as the only argument to the executable. Each subjob requests a single node and uses all 128 cores on the node by placing 1 MPI process per core and specifies 4 hours maximum runtime per subjob:
```
#!/bin/bash

# Slurm job options (job-name, compute nodes, job time)
#SBATCH --job-name=Example_Array_Job
#SBATCH --time=04:00:00
#SBATCH --nodes=1
#SBATCH --tasks-per-node=128
#SBATCH --cpus-per-task=1
#SBATCH --array=1-56     # Set up the job array

# Replace [e05-xxx] below with your budget code 
#SBATCH --account=e05-xxx
#SBATCH --partition=standard
#SBATCH --qos=standard

# Set the number of threads to 1
#   This prevents any threaded system libraries from automatically 
#   using threading.
export OMP_NUM_THREADS=1

#Use this line only if you have named your folders using number 1-56. Otherwise see below
srun --distribution=block:block --hint=nomultithread /path/to/exe $SLURM_ARRAY_TASK_ID
```
* Each subjob gets a number assigned. You can access the number in the job script using `$SLURM_ARRAY_TASK_ID`.
* You may want to enter each folder in your current directory and run, regardless of their names, by:
```
   #Get names of all your folders by 'ls -p | grep /' and then enter the ($SLURM_ARRAY_TASK_ID)th folder
   cd $(ls -p | grep / | sed -n $SLURM_ARRAY_TASK_ID'p')  
   srun --distribution=block:block --hint=nomultithread /path/to/exe $SLURM_ARRAY_TASK_ID
   cd ../
```
* The number of subjobs cannot exceed the number of jobs limited by Qos. But you could submit more by assigning one subjob to more than one folder in sequence (e.g. two jobs within one subjob):
```
   jobrange_first=$(echo "$((($SLURM_ARRAY_TASK_ID-1)*2+1))") #These half of jobs will run first
   jobrange_second=$(echo "$(($SLURM_ARRAY_TASK_ID*2))")      #These half of jobs will start to run after the first half of jobs finish
   cd $(ls -p | grep / | sed -n $jobrange_first', '$jobrange_second'p') 
   srun --distribution=block:block --hint=nomultithread /path/to/exe $SLURM_ARRAY_TASK_ID
   cd ../
```

## Running `VASP` on Archer2

As noted in the [Archer2 VASP documentation](https://docs.archer2.ac.uk/research-software/vasp/vasp/), communication and/or memory overload can pose issues for large (>40 node) VASP jobs on Archer2, so things like large numbers of _k_-points, hybrid functionals, spin-orbit coupling, large `KPAR` etc. which increase the memory usage of VASP can lead to slow calculations on this machine.

One way to get round this is to underpopulate nodes (i.e. only using 64 of the 128 cores per node, so each core has twice the amount of memory available), via the `#SBATCH --ntasks-per-node=64` job script option. Note however that jobs on Archer2 do not share nodes, so you will be charged for the usage of the full node regardless of how many cores per node you actually use.

`VASP 6` has hybrid MPI available (using OpenMP threads alongside pure MPI) which may also help, details of which will (hopefully) be added to the [Archer2 VASP documentation](https://docs.archer2.ac.uk/research-software/vasp/vasp/) in the near future.

Adding the options `--cpu-bind=rank`, `--hint=nomultithread` and `--distribution=block:block` to the `srun` command in your job script will ensure optimal performance of `VASP` on Archer2 (about 20% faster than with the system default options).


Example of a job submission script:
```bash
#!/bin/bash
#SBATCH --job-name="Scaling_Test"
#SBATCH --get-user-env
#SBATCH --output=vasp_out
#SBATCH --error=stderr.txt
#SBATCH --partition=standard
#SBATCH --account=e05
#SBATCH --qos=standard
#SBATCH --nodes=10
#SBATCH --ntasks-per-node=128
#SBATCH --cpus-per-task=1
#SBATCH --time=02:00:00
​
# Set the number of threads to 1
#   This prevents any threaded system libraries from automatically
#   using threading.
export OMP_NUM_THREADS=1
​
module restore /etc/cray-pe.d/PrgEnv-cray
​
# Most efficient options for VASP
'srun' '--cpu-bind=rank' '--hint=nomultithread' '--distribution=block:block' '/work/e05/e05/kavanase/vasp6_in_the_mix/vasp_ncl' 
```

To automatically write a `STOPCAR` file to checkpoint VASP after a specified time (for jobs that are unlikely to complete within the available walltime), replace the last line with:
```bash
start=$(date +%s) # For auto-STOPCARing
# Most efficient options for VASP  # Run as background task to auto-STOPCAR
'srun' '--cpu-bind=rank' '--hint=nomultithread' '--distribution=block:block' '/work/e05/e05/kavanase/vasp6_in_the_mix/vasp_ncl'  &
​
# For auto-STOPCARing:
while [ -e /proc/$! ]; do
    runtime_sec=$(($(date +%s) - start))
    if  [ "$runtime_sec" -gt "$((3600*10))" ]; then # set STOPCAR time in seconds (i.e. 10 hours in this case)
        echo "LABORT = True" > STOPCAR
    fi
    sleep 5m  # Optional: slow the loop so we don't use up all the dots.
done
```


## Running Jupyter notebook on Archer2

As noted in the subsection 'Using JupyterLab on ARCHER2' in the [Using Python](https://docs.archer2.ac.uk/user-guide/python/), it is possible to view/run Jupyter notebooks both on ARCHER2 login nodes and computing nodes. In other words, you can directly connect your local machine to the ARCHER2 server. Please refer to the webpage above regarding installation of Jupyter notebook on your ARCHER2 account. 

After installing Jupyter notebook (or lab) and executing commands below,
```
module load cray-python
export JUPYTER_RUNTIME_DIR=$(pwd)
jupyter lab --ip=0.0.0.0 --no-browser
```
you will get messages something like,
```
Or copy and paste one of these URLs:
        http://ln04:8889/lab?token=<token_id>
     or http://127.0.0.1:8889/lab?token=<token_id>
```

And then connect to ARCHER2 once more (keeping the previous terminal opened) with
`ssh <username>@login.archer2.ac.uk -L <port_number>:<node_id>:<port_number>`
where the number obtained the message above like `8889` is `<port_number>` and `<node_id>` is the one like `ln04`

Afterwords, if you open a web browser with the address `http://127.0.0.1:8889/lab?token=<token_id>` on your machine, you can use Jupyter notebook connecting to the ARCHER2 node.

## Setting up Python

Load the module
```
module load cray-python
```
Set up a virtualenvironment for adding packages
```
python -m venv --system-site-packages /work/e05/e05/<username>/myvenv
```

Set up your Python environment by adding the following to your `.bashrc`
```
export PYTHONUSERBASE=/work/e05/e05/<username>/.local
export PATH=$PYTHONUSERBASE/bin:$PATH

source /work/e05/e05/<username>/myvenv/bin/activate
```

Make sure all of this is loaded up
```
source ~/.bashrc
```
Now you can add packages with `pip`
```
python -m pip install --user <pip>
```

## Setting up `ASE`

Make sure that python is configured as above.
Make sure that you are in the virtaulenvironment that you want - it should be written to the lef of your shell eg
```
(myvenv) e05-keeeto@ln02
```
Now pip install `ase`
```
python -m pip install --upgrade --user ase
```


