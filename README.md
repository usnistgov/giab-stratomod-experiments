# StratoMod Experiments

The main pipeline to run EBM experiments using `snakemake` and `dvc`.

## Repository Layout

### Profiles

In order to have snakemake run reproducibly on many different
machines/architectures, several profiles can be provided at
`workflow/profiles/<profile_name>`.

For now, there is one profile:
- nisaba: for running on the NIST Nisaba cluster (specifically with slurm)

If running on a different compute environment, it will be easiest to copy one
of these files and modifying accordingly.

### Configurations

Each experiment is configured using `config/config.yml`. By convention, this
file is not tracked in any branch except for experiment branches (see below)
and needs to be created and tracked for each individual experiment.

### Snakemake

The generic StratoMod pipeline is stored as a submodule in `workflow/modules`.
See this for details regarding the inner workings of StratoMod.

In general, `snakemake` doesn't need to be run directly (see `dvc` below), but
if troubleshooting or otherwise bypassing `dvc` for whatever reason, the
profiles and configuration should be set up such that the only flags required to
run `snakemake` are `--configfile` and `--profile`.

### DVC

While `snakemake` is used to run the actual experiments, `dvc` is deployed as a
wrapper around `snakemake` to store the results in a (hopefully) sane manner.
Note that `dvc` is only necessary/useful in the context of running experiments;
for testing and development it is easier to simply run `snakemake` directly.

The `dvc` 'pipeline' (see `dvc.yaml`) consistes of one stage whose sole purpose
is to run `snakemake`. Generally `dvc.yaml` doesn't need to be edited.

The behavior of `dvc` is controlled using `config/dvc-params.yml`. Here the
configuration file and the profile for `snakemake` (as described above) can be
set. This should be modified for each experiment.

Upon running `dvc repro` (see below) the `dvc.lock` file will link the
config/profile paramaters, the configuration file itself, and model results to a
specific git commit. These can then be pushed to an external data store (S3)
using `dvc push` and then imported somewhere else using `dvc import` and the
desired git commit which holds the `dvc.lock` data.

### Branches

This repo has the following branches for specific purposes:

* `master`: most up-to-date version of this repository, including conda, dvc,
  submodules, and documentation
* `x_<name>`: an experimental branch, representing one experiment. These should
  initiate from `master`, and should never be merged back into `master`.
* `develop`: development branch for fixes, additions, etc

## Setup

All dependencies should be specified in `env.yml` via `conda` or `mamba`.

To install dependencies, run:

```
mamba env create -f env.yml
```

## Running an experiment

### 1. Create an experimental branch

By convention, each experiment is on its own dedicated branch whose name is
prefixed with `x_`:

```
git checkout -b x_blabla master
```

### 2. Create a config file

Each experiment needs a config file located at `config/config.yml`. A decent
starting point would be to copy the testing configuration in
`workflow/modules/stratomod/config/testing.yml` and modify accordingly.
This file also has annotations throughout which document the usage of each flag
and data structure.

### 3. Run the experiment/store results

#### Auto (dvc + snakemake)

Edit the dvc params with your favorite text editor to change the profile and
config as desired. If running on Nisaba, set the profile to "nisaba" (otherwise
local):

```
notepad config/dvc-params.yml
```

Run the entire pipeline and store results in the cloud:

```
dvc repro
dvc push
```

`dvc repro` will block the terminal so it is recommended to run it in the
background with `&` or use your favorite multiplexer (which is `tmux`).

#### Manual (snakemake only)

Run the entire pipeline via `snakemake` with the following (substitute any
options as desired in the place of `--profile`):

```
snakemake --profile workflow/profiles/<profname> --configfile=config/config.yml
```

Store results in the cloud:

```
dvc commit
dvc push
```

#### Manual on Nisaba (snakemake only)

Use the nisaba profile:

```
snakemake --configfile config/config.yml --profile workflow/profiles/nisaba
```

See the [profile](workflow/profiles/nisaba/config.yaml) for slurm/snakemake
options that are set.

The slurm logs will be found in `cluster_logs`, partitioned by each rule.

Note that this command will block the terminal so it is recommended to run it in
the background with `&` or use a multiplexer like `tmux`.

Commit and push as desired:

```
dvc commit
dvc push
```

### 4. Finalizing data

Once `dvc` is run using any of the above methods, the data will be linked via
`dvc.lock` which in turn is linked to a specific git commit which allows
traceable retrieval (see below).

To update `dvc.lock` simply commit the lock file and push:

```
git add dvc.lock
git commit -m 'updating lock file (or something)'
git push origin
```

## Using results

### Retrieval

Assuming that `dvc commit/push` was properly invoked and that the `dvc.lock`
file was committed on the results in question, data can be accessed for any
given commit/tag. To list the files for a given tag:

```
dvc list --rev <tag> https://gitlab.nist.gov/gitlab/njd2/giab-ai-ebm.git --dvc-only -R
```

To pull the data from the `results/ebm` folder (eg the model output) from a
given tag/commit to a local file path:

```
dvc get --rev <tag> https://gitlab.nist.gov/gitlab/njd2/giab-ai-ebm.git results/ebm -o <local_path>
```

To track in a local `dvc` repo (eg for rigorous analysis) use `import` instead
of `get` in the above command (must be done in a `dvc` repo, which can be
initialized with `git init && dvc init`)

Once an experiment is run, the results will be linked to a specific git commit, 
which in turn can be pulled using `dvc`

### Understanding the output

Each entry under `ebm_runs` in the config corresponds to one EBM run with its
corresponding features and settings to be used. After running the pipeline, each
run (note 'run' refers to a specific model training parameterization defined in
the config, thus one experiment has at least one 'run') should have a directory
under `results/ebm` named as `<run_entry_name>` where `<run_entry_name>` is the
key under `ebm_runs` in the configuration file.

Each directory will contain the input tsv of data used to train the EBM, a
config yml file with all settings used to train the EBM, and python pickles for
the X/Y train/test datasets as well as a pickle for the final model itself.
