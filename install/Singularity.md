# Singularity image build (Rocky Linux 8.10, Singularity 2.x)

This walkthrough builds and distributes a Singularity image that bundles SpliceHarmonization with rMATS, LeafCutter, StringTie, MAJIQ, and the conda environment described in `install/install.yml`. rMATS, LeafCutter, StringTie, and MAJIQ are installed from their upstream sources rather than through the conda environment file.

## Prerequisites
- Singularity 2.x on Rocky Linux 8.10 (or compatible host)
- Root privileges for image creation (`singularity build` requires root in Singularity 2)
- ~10 GB free disk space for the build cache and final image

## Build steps
1. Clone the repository on the build host and move into it:
   ```bash
   git clone https://github.com/interactivereport/SpliceHarmonization.git
   cd SpliceHarmonization
   ```
2. (Optional) Validate the conda solve on your host before building to confirm that Python/R dependencies resolve correctly from `conda-forge`/`bioconda`:
   ```bash
   conda env create --dry-run -f install/install.yml -n spliceharmonization --solver libmamba
   ```
   The pinned versions in `install/install.yml` are published Linux-64 builds on Bioconda and have been chosen to avoid solver surprises on Rocky Linux 8.

3. Build the image from the provided definition file (produces `spliceharmonization.simg` for Singularity 2.x):
   ```bash
   sudo singularity build --force spliceharmonization.simg install/Singularity.def
   ```
   - The definition uses `rockylinux:8` as the base, installs Miniconda, and creates the `spliceharmonization` conda environment from `install/install.yml` for Python/R dependencies only. rMATS, LeafCutter, StringTie, and MAJIQ are installed explicitly in the `%post` section following their upstream guidance:
     - rMATS: https://github.com/Xinglab/rmats-turbo
     - LeafCutter: https://github.com/davidaknowles/leafcutter/tree/master/conda_recipe (installed from source checkout)
     - StringTie: https://github.com/gpertea/stringtie/releases
     - MAJIQ: https://biociphers.bitbucket.io/majiq-docs-academic/getting-started-guide/installing.html (installed from PyPI binaries)
   - If you prefer a writable directory for inspection, replace the target with a sandbox directory:
     ```bash
     sudo singularity build --sandbox spliceharmonization_sandbox install/Singularity.def
     ```

## Running analyses
- To run SpliceHarmonization directly (expects the config file on the host):
  ```bash
  singularity exec -B /host/data:/data spliceharmonization.simg \
    bash -lc "source /opt/conda/etc/profile.d/conda.sh; conda activate spliceharmonization; python /opt/SpliceHarmonization/src/spliceharmonization.py -cfig /data/config.yml"
  ```
- To open an interactive shell with the environment active:
  ```bash
  singularity shell spliceharmonization.simg
  source /opt/conda/etc/profile.d/conda.sh
  conda activate spliceharmonization
  ```

## Verifying installed tools
Run any of the following inside the container to confirm versions:
```bash
rmats.py --help
stringtie --version
python -c "import leafcutter; print(leafcutter.__version__)"
majiq --help
python -c "import tabulate, pandas, pysam; print('python deps ok')"
```

## Sharing the image
- Copy `spliceharmonization.simg` to collaborators (e.g., `scp` or shared storage).
- Provide them with this repository so they can reference example configs and the execution command above.

## Updating the environment
- Edit `install/install.yml` as needed.
- Rebuild the image with the same `singularity build` command to propagate the changes.
