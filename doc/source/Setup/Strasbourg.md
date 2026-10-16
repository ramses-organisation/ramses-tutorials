# Instructions to get started on the Strasbourg setup

For the RAMSES school in Strasbourg, each participant has their own desktop
machine, **`astromasNN`** (where `NN` is the number of the machine you are
assigned). These machines already provide the compilers, libraries and a
ready-made Python environment needed for the tutorials, so there is very
little to install by hand — essentially only the initial-conditions generators
(MUSIC, DICE) and RAMSES itself.

## 1. Getting started

> **Important:** your working directory is **local to your machine** — it is
> not shared between desktops. **Stay on the same `astromasNN` for the whole
> school**; if you move to a different desktop you will not find your files.

All the work below is done under `/data/ramses`:
```bash
cd /data/ramses
```

## 2. Software already installed

The following are installed **system-wide** on `astromasNN` — you do **not**
need to install them:

- **General tools:** `git`, `make`, `cmake`
- **Compilers:** `gfortran`, `gcc`/`g++`, **OpenMPI** (`mpif90`, `mpirun`; check with `ompi_info`)
- **Libraries** (used by MUSIC/DICE and the turbulence tutorial): `fftw`, `gsl`, `ffmpeg`
- **Python 3.12** with `jupyter`, `numpy`, `scipy`, `matplotlib`, `astropy`

The Python packages used for the analysis (`osyris`, `yt`, `yt_astro_analysis`,
`turbustat`, `colossus`, `f90nml`, …) are provided in a **ready-made virtual
environment** (see the *Running and analysing* section below). You do **not**
need to `pip install` anything.

## 3. Download the ramses tutorials repository

```bash
cd /data/ramses
git clone https://github.com/ramses-organisation/ramses-tutorials.git
```
This repository contains the tutorial notebooks **and the namelist (`.nml`)
files** you will use to run the simulations.

## 4. Compiling third-party initial-condition generators

Some tutorials use external codes to generate initial conditions. Build them
once, under `/data/ramses`:

### MUSIC (v2) — cosmological initial conditions
```bash
cd /data/ramses
git clone https://github.com/cosmo-sims/MUSIC2.git
cd MUSIC2
mkdir build && cd build
cmake ..
make -j
```
> On `astromasNN` the non-interactive `cmake ..` works directly (no need for the
> interactive `ccmake` step).

### DICE — idealised galaxy initial conditions
```bash
cd /data/ramses
git clone https://bitbucket.org/vperret/dice
cd dice
mkdir build && cd build
cmake -DCMAKE_C_FLAGS=-fcommon ..
make
make install
```

## 5. Building RAMSES

Get the code and open the Makefile:
```bash
cd /data/ramses
git clone https://github.com/ramses-organisation/ramses
cd ramses/bin
emacs Makefile          # (or your editor of choice)
```

Set the compiler commands for this machine:
```make
# Compiler command for regular compilation (mpi or serial)
F90 = mpif90
# Serial compiler command and flags for helper files
FC = gfortran
```
A plain `make` at this point builds a serial, 1D executable (`ramses1d`) — a
good first check that compilation works.

Now adjust the build options for the tutorials (still in `bin/Makefile`):
```make
NDIM     = 3        # number of dimensions
NPRE     = 8        # float precision (double)
SOLVER   = hydro
PATCH    =
RT       = 0
USE_TURB = 0
MPI      = 1        # build with MPI
```
and recompile:
```bash
make clean
make
```
`ls -ltr` should now show the executable **`ramses3d`**.

Quick sanity check:
```bash
./ramses3d
mpirun -n 4 ./ramses3d
```
RAMSES will **abort complaining that it needs a `.nml` file** — this is
expected and simply means the executable is working. You provide a namelist
when you actually run a simulation (next section).

## 6. Running and analysing

### Namelists — where to find them
> **Important:** the `.nml` files for the tutorials are **not** in
> `ramses/namelist/`. Use the ones provided in the **tutorials** directory
> (`ramses-tutorials/<tutorial>/…`) and run from there.

To run a simulation you call the executable with a namelist, e.g. for the 1D
Sod shock tube of the Fundamentals tutorial:
```bash
cd /data/ramses/ramses-tutorials/Fundamentals/sod-tube
/data/ramses/ramses/bin/ramses1d sod-tube.nml
```
Edit namelists with a text editor (e.g. `emacs`) as you go.

### The namelist structure
A RAMSES namelist is split into **blocks**, each starting with `&BLOCK_NAME`
and ending with `/`. The most common ones are:

- `&RUN_PARAMS` — what physics is on (hydro, poisson, rt, …) and control flags
- `&INIT_PARAMS` — initial conditions (region setup, or an IC file)
- `&AMR_PARAMS` — grid: `levelmin`, `levelmax`, `boxlen`, memory sizes
- `&OUTPUT_PARAMS` — when to write outputs (`tout`, `delta_tout`, `noutput`)
- `&HYDRO_PARAMS` — solver, `gamma`, Riemann solver, Courant factor
- `&REFINE_PARAMS` — refinement criteria
- `&UNITS_PARAMS` — code-unit definitions (density, length, time)

You can either follow the **website** or open the **`.ipynb` notebooks**
provided in the `ramses-tutorials` directory — they contain the same steps plus
the analysis cells.

### Python analysis environment
For the analysis/visualisation parts, activate the ready-made virtual
environment and start Jupyter:
```bash
source /opt/venv/ramses/bin/activate
jupyter-notebook
```
Then open the tutorial notebook (e.g. `Fundamentals/tutorial.ipynb`) and run the
analysis cells.

## 7. Notes specific to this setup (Fundamentals tutorial)

A few things noticed while running the Fundamentals tutorial on `astromasNN`:

- **Sod shock tube (1D):** runs out of the box; analyse it in the notebook with
  the activated `ramses` environment.
- **Sedov–Taylor blast (3D):** use `levelmin=6`, `levelmax=7` (i.e.
  `levelmin ≠ levelmax`). With `levelmin=levelmax` the analysis of the
  refinement-level map (its colour bar) does not behave well.
- The **optional units exercise** (how `p_region` maps to 10⁵¹ erg) is a good
  but challenging extra — take your time with it.
- **Strömgren sphere (2D):** the original 4-panel plotting cell did not render
  (osyris cannot slice a 2D simulation). The notebook cell has been updated to
  reshape and plot the 2D cells directly — see the Fundamentals notebook.
