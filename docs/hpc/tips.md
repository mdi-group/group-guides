# Tips

Some hints and tips to make your HPC life easier

## Setting up an ssh configuration

If you set up an ssh config file it will make life a lot easier. Say for example I regularly connect to a cluster called foo.bar normally I would do
```
ssh myname@foo.bar
scp myname@foo.bar
rsync myname@foo.bar
```

But with ssh config files you can easily set it up to alias this cluster to foo for example, then all you need is
```
ssh foo
scp foo
rsync foo
```

and the nice thing is that you can copy this to all your machines and tell it where `ssh` keys and so on are kept.

To do this, you just need to edit or make a file called `config` in your `.ssh/` directory. Here's an example from my one:
```
# Global settings

Host *
    Protocol 2
# Protocol 1 well dodgy
    Compression yes
    ServerAliveCountMax 5
    #UseKeychain yes

# Up to 5 pings with no reply before hang up
    ServerAliveInterval 30
# Seconds. Needs to be <60s for some home routers

# Enable Master Sessions - (for scp autocomplete + not hammering HPC facilities)
# See: http://unix.stackexchange.com/questions/32984/multiple-ssh-sessions-in-single-command/33012#33012
# To create a master: ssh -N -M target-host
# To delete a dead one: ssh -O exit target-host

ControlMaster auto
ControlPath ~/.ssh/control:%h:%p:%r

Host andrena
    User exx847
    HostName andrena.hpc.qmul.ac.uk
    IdentityFile ~/.ssh/id_rsa_apocrita

Host apocrita
    User exx847
    HostName login.hpc.qmul.ac.uk
    IdentityFile ~/.ssh/id_rsa_apocrita
```

You can also set up multiple github accounts
```
#Default GitHub
Host github-keeeto
  HostName github.com
  User git
  IdentityFile ~/.ssh/id_rsa

#MDIG GitHub
Host github-mdig
  HostName github.com
  User git
  IdentityFile /home/keeeto/.ssh/id_ed25519_mdig
```

## Adding ssh keys to github

You will want to add ssh keys from your various machines to github, this means that you can pull and push from those machine to git.

* On github go to your profile > settings > ssh keys
* Select `New SSH key`
* On the machine you want to access, in the command line type `ssh-keygen`
* You can add a passphrase if you like
* Now copy the public key - `vi <location>/.ssh/id_rsa.pub` - note that the full location will have been printed out during the key egenration process, so use that
* Copy the contents
* On github, give the key a name, then paste the contents of your clipboard into the `Key` box
* Click `Add SSH key`

## Set up alignn on Andrena

`ssh` to andrena

We then build a conda environment
```
module load miniconda
mamba create --name alignn
mamba activate alignn
```

Now we want to install `alignn`, I usually have an `src` directory in my home for any install from source packages

```
cd 
cd src
git clone https://github.com/usnistgov/alignn.git
cd alignn
python setup.py develop
pip install dgl-cu111
```

We are almost installed. 

```
pip install pymatgen
```

Now we need a submission script. The following script will run the example `train_from_folder.py` in the `alignn` source directory.

```
#!/bin/bash
#$ -cwd
#$ -j y
#$ -pe smp 8          # 8 cores per GPU
#$ -l h_rt=0:1:0    # 1 hour
#$ -l h_vmem=11G      # 11G RAM per core
#$ -l gpu=1           # request 1 GPU
#$ -l cluster=andrena # use the Andrena nodes

module load miniconda
module load cudnn/8.1.1-cuda11.2
mamba activate alignn

python train_folder.py --root_dir "examples/sample_data" --config "examples/sample_data/config_example.json" --output_dir=temp
```


