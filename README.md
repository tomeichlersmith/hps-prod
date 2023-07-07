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

#### job.cfg
```cfg
[Job]
enable_copy_input_files = False
enable_copy_output_files = False
```
#### job.json
```json
{
  "input_files" : {
    "input_full_name.ext" : "input.ext" 
  },
  "output_files": {
    "output.ext" : "output_full_name.ext"
  }
}
```
#### command line
```
hps-mc-job run <script> job.json -c /usr/local/share/container.cfg -c job.cfg -d scratch
```

### Example
```
apptainer \
  run \
  --home $PWD \
  --bind /path/to/jobs.json \
  --bind /path/to/job.cfg \
  /path/to/image.sif \
  hps-mc-job run \
    <script>
    /path/to/jobs.json \ 
    -c /usr/local/share/container.cfg \
    -c /path/to/job.cfg \
    -d scratch \
    -i <job_index>
```
