# Environments

This repository is stored in the directory `/gpfs/exfel/sw/software/euxfel-environment-management` on Maxwell.

An environment is created per European XFEL experiment cycle, this is done so that previous environments are preserved for reproducibility. The files defining the environments are stored `./environments/${CYCLE}` (note: `./` refers to **this git repository**, not the GPFS software directory).

Each environment directory will have a few files:

- `base.yml` - Conda environment file containing packages which are available on a Conda channel
- `custom.yml` - optional Conda environment file containing packages built from custom recipes (see [Recipes](./recipes.md))
- `conda-lock.yml` - file generated by `conda-lock lock` using the environment files as inputs
- `conda-linux-64.lock` - a 'render' of the lock file which can be installed by Conda, this contains 'explicit' package versions and channels
- `conda-linux-64.lock.yml` - an alternative 'render' of the lock file, but in the form of a standard Conda `environment.yml` file

Environments exist in an installation of Conda, setting up a new Conda installation is very rarely required and is covered in the [Instances](./instances.md) section.

## Environment Specification Setup

### Creating New Specifications

The first step to creating a new environment is activating an installation, this can be done with `module load exfel mambaforge`. Loading this module will initialise the Conda instance into the `base` environment which provides `grayskull` and `conda-lock` which are used to create the environments.

Once a Conda instance has been activated, you can create a new directory under either `./environments/${CYCLE}` or `./applications/${APPLICATION_NAME}/{APPLICATION_VERSION}` depending on whether the environment is intended to be a generic environment users will activate to write and execute their own code, or if the environment exists only to provide a specific application.

If a new **cycle environment** is being created, copy the `base.yml` and `custom.yml` files from the previous cycle, and use those as a starting point.

Otherwise create a standard Conda environment file defining the channels and the dependencies you require:

!!! info inline end

    If the environment is likely to be used as a Jupyter kernel on Max-JHub then you need to take care to have the same versions of a few packages, see [Interactive Plotting Issues in Jupyter Notebooks](#interactive-plotting-issues-in-jupyter-notebooks).

```yaml
channels:
  - conda-forge
  - defaults
dependencies:
  - numpy  # for example
```

And place any dependencies under the `dependencies` section. Note that these are Conda dependencies, from Conda channels, not from PyPI, so the package names may differ (see <https://anaconda.org/> to search through the official channels).

Once all required dependencies have been added to the dependencies list, carry on with the instructions in [Locking and Installing an Environment](#locking-and-installing-an-environment), as well as [Creating a Modulefine](#creating-a-modulefile).

### Modifying Existing Specifications

To add a new package to an existing environment, the package should be added to the `base.yml` if is is an existing package on a Conda channel, or added to `custom.yml` if it is a package where the recipe has to be created by us.

Once all required dependencies have been added to the dependencies list, carry on with the instructions in [Locking and Installing an Environment](#locking-and-installing-an-environment).

## Locking and Installing an Environment

First run `module load exfel mambaforge`, this will activate an environment containing environment management tools like `conda-lock`.

The following commands can be used to concretize the environment and update or create `conda-lock.yml`:

!!! warning inline end

    There is currently a bug causing issues with locking certain packages which can cause the locking to fail, to get around this you can add the `--no-mamba` flag to the first `conda-lock` command. This will slow down the locking process but will let it complete successfully.

```bash
conda-lock -f base.yml -f custom.yml --lockfile conda-lock.yml -p linux-64
conda-lock render -e base -e custom -p linux-64 -k env -k explicit ./conda-lock.yml
```

The first command will generate/update the `conda-lock.yml` file, the second command will generate the `conda-linux-64.lock` and `conda-linux-64.lock.yml` files.

If the locking completes successfully, you should **add and commit all files** with a description of the changes made in the commit message. Then you can then update/install the environment via:

```bash
mamba env update -n ${ENV_NAME} -f ./conda-linux-64.lock.yml
```

Which will also show a list of changes that will be made to the environments that should be double checked.

!!! note

    The installation was previously done with the `conda-lock install --name ${ENV_NAME} ./conda-lock.yml` command. Internally, this creates the `conda-linux-64.lock` file and then installs it, however for us it is preferable to use the `conda-linux-64.lock.yml` file as it will update the environment **in-place** instead of replacing it, which is faster and less likely to disrupt users.

## Creating a Modulefile

If you are updating an environment, the modulefile does not require changes. If you've created a new environment, you need to create a module file for it so that users can easily activate it.

As an example, here is the module file for the `202301` cycle:

```tcl
#%Module 1.0
proc ModulesHelp {} {
    puts stdout    "Mamba environment for cycle 202301"
}

module-whatis  "Module loads the mamba environment for cycle 202301"

prepend-path    PATH /gpfs/exfel/sw/software/mambaforge/22.11/envs/202301/bin
setenv      CONDA_DEFAULT_ENV 202301
setenv      CONDA_PREFIX /gpfs/exfel/sw/software/mambaforge/22.11/envs/202301
setenv      CONDA_PROMPT_MODIFIER {(202301) }
setenv      CONDA_SHLVL 1
setenv      GSETTINGS_SCHEMA_DIR /gpfs/exfel/sw/software/mambaforge/22.11/envs/202301/share/glib-2.0/schemas
setenv      GSETTINGS_SCHEMA_DIR_CONDA_BACKUP {}
```

??? note "Environment Variable Modification vs. Conda Init and Activate"

    This was previously done with `module load mambaforge` to load the base environment and then `puts stdout "mamba activate 202301;"` to execute the activate command in the users shell, however this process can be slow due to the shared filesystem.

    To avoid this problem the environment variables are modified directly instead, which is faster but comes with the downside of not being able to use the many `conda` commands as they rely on shell functions, and shell functions are not supported in our version of Environment Modules on Maxwell.

For new environments, it is possible to just adjust the paths in the above module file.

This file can be created by writing a small shell script that activates the environment via conda:

```bash
#!/bin/zsh -l

source /gpfs/exfel/sw/software/mambaforge/22.11/bin/mamba-init

mamba activate 202301
```

And then using either the the `sh-to-mod` command or the script to convert it to a modulefile:

```bash
# Script, for older versions of environment modules (<= 3)
/usr/share/Modules/bin/createmodule.sh ./script.sh

# Command, for newer versions of environment modules
module sh-to-mod bash ./script.sh
```

!!! warning "Environment Module Version on Maxwell"

    1. The version of Environment Modules on Maxwell is quite old, and does not support the `sh-to-mod` command. Instead, the `createmodule.sh` script should be used to create the modulefile.

    2. Newer versions ot Environment Modules have a `set-function` command which can be used to set shell functions and reproduce exactly what `conda init`/`conda activate` would do. However, again, this is not supported with the old version of Environment Modules on Maxwell.

### Setting a Default Modulefile

To set a default modulefile, go to the directory the modulefile is in (for cycle environments that would be `/gpfs/exfel/sw/software/xfel_modules/mamba`) and create a file called `.version` containing:

```tcl
#%Module1.0
set ModulesVersion "{NEW_CYCLE_NUMBER}"
```

This will set the default for that directory to the new cycle number.

## Creating a Kernel for Max-JHub

The following directory contains kernels which are automatically on the kernel list on Max-JHub:

```shell
/gpfs/exfel/sw/software/local/share/jupyter/kernels
```

To add a new kernel, create a new directory in this location with the name of the kernel, and then create a `kernel.json` file in that directory with the following contents:

```json
{
  "argv": [
    "${PYTHON_PATH}",
    "-m",
    "ipykernel_launcher",
    "-f",
    "{connection_file}"
  ],
  "display_name": "xfel (202301)",
  "language": "python"
}
```

The `display_name` is what will be displayed in the Max-JHub kernel list, this should be descriptive and (if for a cycle environment) include the cycle number.

Replace `${PYTHON_PATH}` with the path to the python executable for the environment, this can be easily found by loading/activating the environment and running `which python`.

## FAQ

### Interactive Plotting Issues in Jupyter Notebooks

If an environment has issues with interactive plotting in Jupyter notebooks it is likely that the packages in it are out of sync with those in the environment running Jupyter/JupyterLab on Max-JHub.

The module used for Max-JHub is mentioned on the DESY documentation page here: <https://confluence.desy.de/display/MXW/JupyterHub+on+Maxwell>. Currently this is `conda/3.8`.

You should load that environment and then check the versions of the following packages:

- ipympl
- ipywidgets
- matplotlib

And pin them to be the same as the versions currently used by Max-JHub. Currently (February 2023) this means:

```text
- ipympl=0.7.0
- ipywidgets=7.6.3
- matplotlib=3.4.2
```