#Myriad

Myriad is UCL's multi-purpose cluster. It has some A100 and V100 GPUs that could be useful.


## Setting up `miniconda`

We will probably need a package manager on Myriad. We can use `miniconda`, if you are a bit more adventurous you can use `micromamba` instead. Since I installed `miniconda` all of my instructions afterwards use `miniconda` paths. If you do go for `micromamba` update paths accordingly.

```
wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh
bash Miniconda3-latest-Linux-x86_64.sh -b
~/miniconda3/bin/conda init bash
source .bashrc
```

I have also found that I needed to add the path to the `conda` versions of libraries to my $LD_LIBRARY_PATH, 
so add the following to the end of your `.bashrc` file

```
export LD_LIBRARY_PATH=/home/<username>/miniforge3/lib:$LD_LIBRARY_PATH
```

## Setting up a `pytorch`-based environment

Many of the environments we need will be running `pytorch`. Myriad's available installs are pretty patchy, so we probably want our own. Here's the general recipe.

Set up and activate a new `conda` environment, in this case I want `python` 3.10. You can replace `alignn-env` with any reasonable name you like:
```
conda create --yes --quiet --name alignn-env python=3.10
conda activate alignn-env
```
Install the `CUDA` toolkit that you need, in this case we want 11.8
```
conda install --yes --quiet cudatoolkit=11.8.0
```
Install `pytorch`, **note** the `conda` verison does not work on Myriad, so we use `pip` instead.
**Note** - check the `pytorch` [website](https://pytorch.org/get-started/locally/) to check that the command is the same.
```
pip3 install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu118
```

## Setting up `alignn` 

This is largely based on the above for `pytorch`, there are a few extra steps to install `alignn`

First install `dgl`
```
conda install -c dglteam/label/cu118 dgl==1.0.2.cu118
```
Clone and install the `alignn` code. I ususally have an `src` directory in my home folder for installing local versions of code, but you can put it anywhere sensible, the following assumes my directory structure
```
cd ~/src/
git clone https://github.com/usnistgov/alignn
cd alignn
python -m pip install -e .
```

Youll probably want `pymatgen` too
```
conda install -c conda-forge pymatgen
```
