# ⚠️ Deprecated
These packages have been moved into hps-env. This was decided because which packages
are being developed changes depending on the project and users can use the environment
variables with their denv to choose their custom local installations of packages rather
than the default system installations.

# hps-prod
Containerized HPS production.

The image built by this context is based off
[hps-env](https://github.com/tomeichlersmith/hps-env)
which is used to interactively develop HPS software.

## Usage
The general idea of this container is to leverage
hps-mc's recipes to run different jobs using the
dependencies that are installed. We do have to work
around some other features that hps-mc has like
input and output file copying.

If your batch system is configured to allow users to 
read/write directly from/to the data storage file
system, then simply make sure the directories holding any
input files or being destination directories are mounted 
to the container at run time.

If your batch system needs to handle copying itself, 
allow the batch system to handle copying files in/out
and always use relative paths for hps-mc configuration.

Finally, we need to set the home directory of the container
to be the current running directory so that the lcsim cache
directory can be properly created without crashing hps-java.
In addition, one can mount the various config files into the
container so they don't have to be copied into the working directory
if you desire.

## Examples
I've gotten this working at UMN and SLAC.

### SLAC
Since SDF has a distributed filesystem that we can read/write to,
the batch submission is relatively simple. Just use full paths like
normal, making sure to `--bind` directory we want to read/write to
the prod container so it also has access.

SLAC SDF limits users to 250 jobs running at any one time
(don't quote me on that number, I'm not sure), so you will probably
want to submit jobs in chunks of size ~200 if your total number of
jobs is greater than the limit.

#### run
This is the script provided to `sbatch` to submit jobs with.
You can test run it on an interactive node as long as you provide
a job ID as the 5th argument. `sbatch` defines the `SLURM_ARRAY_TASK_ID`
environment varibale for us when running in batch.
```bash
#!/bin/bash
#SBATCH --ntasks=1
#SBATCH --time=24:00:00
#SBATCH --partition=shared
#SBATCH --output=output/logs/%A-%a.out
#SBATCH --mem=8G
#
#  run an hps-mc job in the hps-prod container
# ARGUMENTS
#  1 - image    - full path to SIF to run
#  2 - script   - name of hps-mc job script to run
#  3 - job_json - full path to job json to run
#  4 - job_cfg  - full path to job cfg to use
#  5 - job_id   - which job id to run

set -o xtrace

image="$1"
script="$2"
job_json="$3"
job_cfg="$4"
job_id="${SLURM_ARRAY_TASK_ID:-$5}"

srun \
  /usr/bin/singularity \
  run \
  --home /scratch/$USER \
  --bind /sdf/group/hps/users \
  ${image} \
  hps-mc-job \
    run \
    -c /usr/local/share/container.cfg \
    -c ${job_cfg} \
    -d /scratch/$USER/batch/${job_id} \
    ${script} \
    ${job_json} \
    -i ${job_id}
```

### UMN
At UMN we use HTCondor which creates a working directory for us which
we should use as our scratch area and home for the container. There
are two additional complexities.

1. HTCondor runs the job under the `nobody` user.
2. We want HTCondor to handle copying input/output files

(1) means we have to do some special movement of the lcsim cache directory while
(2) means we have to have a funky hps-mc job definition so that the output files
are where HTCondor expects them to be.

#### run
This script is what was run by HTCondor as the executable within a job.
HTCondor copies out all files generated within the working directory that
aren't in a subdirectory and aren't labeled as input files in the job submission,
so we put all of our work and intermediate files in a `scratch/` subdirectory
so they are ignored (and eventually deleted) by HTCondor.
```bash
#!/bin/bash
#  run an hps-mc job in the hps-prod container
# ARGUMENTS
#  1 - image    - full path to SIF to run
#  2 - script   - name of hps-mc job script to run
#  3 - job_json - full path to job json to run
#  4 - job_cfg  - full path to job cfg to use
#  5 - job_id   - which job id to run

set -o xtrace

image="$1"
script="$2"
job_json="$3"
job_cfg="$4"
job_id="$5"

cwd=$(pwd -P)

mkdir ${cwd}/scratch
printf >  ${cwd}/scratch/java-args.cfg "[JobManager]\n"
printf >> ${cwd}/scratch/java-args.cfg "java_args = -Xmx1g -XX:+UseSerialGC -Dorg.lcsim.cacheDir=${cwd}/scratch\n"
printf >> ${cwd}/scratch/java-args.cfg "\n"
printf >> ${cwd}/scratch/java-args.cfg "[FilterBunches]\n"
printf >> ${cwd}/scratch/java-args.cfg "java_args = -Xmx1g -XX:+UseSerialGC -Dorg.lcsim.cacheDir=${cwd}/scratch\n"


/usr/bin/apptainer \
  run \
  --home ${cwd}/scratch \
  --bind ${job_json} \
  --bind ${job_cfg} \
  --bind ${cwd} \
  ${image} \
  hps-mc-job \
    run \
    -c /usr/local/share/container.cfg \
    -c ${job_cfg} \
    -c ${cwd}/scratch/java-args.cfg \
    -d ${cwd}/scratch \
    ${script} \
    ${job_json} \
    -i ${job_id}
```

#### job.cfg
```cfg
[Job]
dry_run = False
# delete output files if they already exist
delete_existing = True
# and delete the run directory
delete_rundir = True
# don't bother copying input files
enable_copy_input_files = False
```

#### job.json
Since HTCondor copies any input files in for us and copies output files in the HTCondor directory 
out for us, the output directory should be `.` and the input/output file mappings should just be
doing the simple task of renaming things.
```json
{
  "input_files" : {
    "input_full_name.ext" : "input.ext" 
  },
  "output_files": {
    "output.ext" : "output_full_name.ext"
  },
  "output_dir": "."
}
```
