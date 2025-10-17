# Instructions to get started on the Infinity machine

For the 2nd edition of the RAMSES school, participants are hosted by the Institut d'Astrophysique de Paris ([IAP](https://www.iap.fr/)) and given access to the Infinity cluster.

Special instructions are required to properly set up the dependencies needed to run the tutorials on Infinity.

For the 2025 school, you should work in the following folder
```bash
cd /data79/RamsesTrainingScratch
mkdir $USER
cd $USER
```

## Setting up a conda environment
Install conda (to have a working version of Python), need only be done once
```bash
mkdir -p ~/miniconda3
wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh -O ~/miniconda3/miniconda.sh
bash ~/miniconda3/miniconda.sh -b -u -p ~/miniconda3
rm ~/miniconda3/miniconda.sh
~/miniconda3/bin/conda init bash
```
After this, close the terminal and login again. Then, create a new environment (i.e., an installation of a given version of Python)
```bash
conda update -n base conda
conda create -n ramses-env python=3.13
```

To activate your new shiny Python environment, which needs to be done in each new terminal (equivalent of doing a module load), do
```bash
conda activate ramses-env
```
You should see the prompt change to indicate you're now in the ramses-env environment. Once this is done, confirm you use the right version of python:
```bash
which python      # should be ~/miniconda3/envs/ramses-env/bin/python
python --version  # should be Python 3.13.7 (as of 6 October)
```

### Install the packages required for the tutorial
This step only needs to be done once. Make sure you're in the ramses-env environment (see above).

```bash
pip install numpy astropy matplotlib jupyter scipy f90nml yt yt_astro_analysis colossus osyris
```

## Compiling DICE and MUSIC and RAMSES

### Compiling DICE and MUSIC

Follow instructions to compile DICE and MUSIC (https://ramses-tutorials.readthedocs.io/en/latest/Setup/general_requirements.html#music-v2)

### Compiling RAMSES

The recommended compiler on Infinity is the Intel compiler, which is provided by Stéphane Rouberol. To use it, load the corresponding module
```bash
module load inteloneapi/2025.2.1
# The following is also required for some tutorial, think about loading it!
module load fftw
```
As any module, you will need to load it each time you want to compile and/or run RAMSES.

In a working folder, clone the latest version of RAMSES:
```bash
git clone https://github.com/ramses-organisation/ramses
cd ramses
```

Now to compile RAMSES:
1. edit `bin/Makefile` to replace `ifort` with `ifx` in the filename,
2. change directory to `bin/`,
3. and invoke the `make` command with the following arguments:
   ```bash
    make COMPILER=INTEL NDIM=3 MPI=1 MPIF90=mpiifx [...]
    ```

The content of the `[...]` depends on what flavour of RAMSES you want to compile (see the tutorials). To verify everything works, you can try to compile the vanilla version of RAMSES, leaving `[...]` empty, i.e. do:
```bash
make COMPILER=INTEL NDIM=3 MPI=1 MPIF90=mpiifx
```
This should produce an executable named `ramses3d`.


**Note:**
- if you had access to clusters in the past, the Intel fortran compiler used to be named `mpiifort/ifort`. With more recent version of the Intel compiler, those are now named `mpiifx/ifx`,
- that additional options (such as `PATCH=...`) can be appended to the make command, as is done in the tutorials.

## Running jupyterlab/jupyter notebook on Infinity

A general rule of thumb is that one should _never_ run interactive programs on the login node of a cluster, including Infinity.

However, jupyter lab / jupyter notebooks (they're the same, the only thing that changes is how they look like) need to be run somewhere, while you access them through your browser on your laptop. Therefore, we need to set up a way for your laptop to communicate with the compute node. This is very easy when running jupyter notebooks on your machine (since your machine can communicate with itself). When running on a cluster, the workflow is more complex, since the compute nodes are typically not accessible from the outside. We can however circumvent that limitation with SSH tunnels:

    [ localhost ] ------> [ login_node ] ------> [ compute_node ]

Note that `localhost` is always the local machine you're connected to (so from Firefox/Chrome, if you try to access `localhost`, it tries to access something on your laptop).

An important concept to make sense of these tunnels is the concept of "port". When communicating over the internet, you always specify a given IP address (for example `localhost`, or a domain, e.g. `google.com` or `infinity01.iap.fr`) and a port. By default, the port is implicitly set to 80 or 443 when accessing a web page, set to 22 for SSH, etc. but it is used nonetheless. Furthermore, only one process can listen to a port. Indeed, if several processes were listening to the same port, the OS would have no way to know which process it needs to forward the request to. Hence, each user needs to work on a different port (since we're all going to use our own jupyter notebooks). The sketch above thus now reads:

    [ localhost:PORT1 ] ------> [ login_node:PORT2 ] ------> [ compute_node:PORT3 ],

`PORT1`, `PORT2` and `PORT3` are port numbers (a number between 1024 and 65535, ports between 0 and 1024 are reserved for the OS). In practice, it is easier to work with `PORT1 = PORT2 = PORT3`, which we will do in the following.

The first step can be achieved with a "forward tunnel", i.e. when you SSH to infinity, you instruct SSH to forward requests from `localhost:PORT1` to `login_node:PORT2`.

The second step is slightly more complicated (since you don't know beforehand which compute node you will be allocated) but can be achieved with a "reverse tunnel", i.e., from the compute_node, you connect back to the login_node and ask a tunnel to be created, instructing SSH to forward requests from `login_node:PORT2` to `compute_node:PORT3`.

Finally, on the compute_node, you can launch your jupyter notebook and ask it to listen to `PORT3`.

With that setup, opening `localhost:PORT1` from Firefox/Chrome on your laptop will make your browser communicate with SSH running on your laptop. SSH will forward your request to `login_node:PORT2`. There, another SSH process running this time on the `login_node` will pick up the request, and forward it to `compute_node:PORT3`. There, and finally, jupyter will receive a request and send back the page to view in your browser (following the opposite route).

### Step-by-step instructions

#### Pick a port number

First, pick a random number between 6000 and 65535 (to avoid conflicts with other users). This will be the port number you will use for your jupyter notebook. You can pick it randomly with the `shuf` command, or just pick one manually. On Linux/Mac:
```bash
echo $((RANDOM % (65536-6000) + 6000))
```
If this does not work, simply pick a number yourself (just avoid "obvious" choices, so that you don't pick the same port as someone else).

Take a note of that number, we will call it `<RANDOM_NUMBER>` in the following.

#### Setup SSH tunnel from laptop to infinity
Create an SSH tunnel from your laptop to Infinity
```bash
ping -c 2 -s 999 infinity02.iap.fr
ssh -L<RANDOM_NUMBER>:localhost:<RANDOM_NUMBER> user@infinity02.iap.fr
```
This is the step `[ localhost:PORT1 ] -> [ login_node:PORT2 ]` above, with `PORT1 = PORT2 = RANDOM_NUMBER`.

You can automatize this process by adding the following lines to your `~/.ssh/config` file on your laptop:
```
Host infinity
     LocalForward <RANDOM_NUMBER> localhost:<RANDOM_NUMBER>
     HostName infinity02.iap.fr
     User <USERNAME>
     ProxyCommand sh -c 'ping -c 2 -s 999 %h && exec nc %h %p'
```

This will allow you to connect using simply `ssh infinity`, which will always do the forwarding automatically.

At this point, we will assume that you can connect to infinity and that the port forwarding works. We now need to set up the reverse tunnel and launch jupyter lab on a compute node. For this, we need to be able to connect with no password from the compute node to the login node. This can be done with SSH keys.

#### Setup passwordless connection from infinity to infinity

So let's generate an SSH key. Importantly, when prompted for a password, leave it empty (just press enter). On infinity, do:
```bash
ssh-keygen  # follow the instruction, I saved mine as /home/cadiou/.ssh/id_rsa_local_connection
```

Let's now configure Infinity to allow connection with this key
```bash
ssh-copy-id -i ~/.ssh/<name_of_the_key_you_typed_above> localhost
```

And finally, let's test that it works:
```bash
ssh -i ~/.ssh/<name_of_the_key_you_typed_above> localhost
```
This should connect to infinity again, but this time without asking for a password. If it asks for a password, seek help.

#### Create a script to submit jupyterlab jobs

At this point, we have a way of connecting from your laptop to Infinity with proper port forwarding, and a way to connect from the compute node back to the login node without password (so it can be called from a script).

We can now create a script to submit jupyterlab jobs. On infinity, create a new file (e.g. `~/submit_notebook.sh`) with the following content (replace `<RANDOM_NUMBER>` by the port number you picked above):
```bash
#!/bin/bash
#SBATCH --job-name=Jupyterlab
#SBATCH -n 128
#SBATCH --exclusive
#SBATCH --reservation=RamsesTraining
#SBATCH --time 24:00:00
#SBATCH -o notebook.log
#SBATCH -e notebook.err

NOTEBOOKPORT=<RANDOM_NUMBER>

module load inteloneapi/2025.2.1
module load fftw

set -x

>&2 echo "HOSTNAME: $(hostname)"
pkill ssh
pkill jupyter-lab

cd $SLURM_SUBMIT_DIR
# setup reverse SSH tunnel between computing and login node
# this is the step [ login_node:PORT2 ] ------> [ compute_node:PORT3 ], with PORT2 = PORT3 = RANDOM_NUMBER
ssh -N -f -R $NOTEBOOKPORT:localhost:$NOTEBOOKPORT $SLURM_SUBMIT_HOST

# launch the notebook, listening on the forwarded port
~/miniconda3/envs/ramses-env/bin/jupyter lab --port=$NOTEBOOKPORT --no-browser
```

#### Running jupyterlab
From a directory where you want to use jupyter lab from (e.g., in the RAMSES tutorial), simply do:
```bash
sbatch ~/submit_notebook.sh
```

If everything goes well, it should create a tunnel between the compute node and the login node.

The file `notebook.err` should contain a string with the URL that you should access, e.g. `http://127.0.0.1:1<RANDOM_PORT>/lab?token=<SOME_LONG_STRING>` which you can open on your local machine in your favorite browser (Firefox/Chrome/~~Netscape~~).
