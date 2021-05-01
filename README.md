# General rocks notes
Rocks is at rebirth of old PDSF at CENPA (UW) for local use.
If you have an npl login, you can consult the [CENPA guide page](https://cenpa.npl.washington.edu/display/CENPA/Introduction).
The firewall is configured to require connections from inside the UW network, so off-campus users should route through the UW or CENPA VPN's.

## LEGEND-specific usage notes
With account creation, users should request membership in the `LEGEND` group.
Users are encouraged to verify their status using the `id` or `groups` commands at first login.

The common data storage element, shared between all Jason's projects, is `/data/eliza1/LEGEND`.
_FIXME: Should we link /data/LEGEND here after the RAID migration is complete?_
This is a ~40 TB RAID6, so space is moderately abundant and all data is reasonably secure.

# Computing notes
The head and worker nodes have singularity installed >= v3.4.
It is recommended that software be containerized for deployment on rocks and other compute environments.

As a common resource, builds of legend-base and legend-software are maintained at `/data/eliza1/LEGEND/software/containers/${name}/latest.sif`.
Each build is datestamped (e.g., legend-base-200414.sif), but then symlinks are maintained for `release.sif` and `stable.sif`.
If your workflow requires an old build of the software, make sure the group is aware so we don't accidentally delete your build; we only keep so many 3+ GB builds lying around.

Additionally, a "production" version of pygama is maintained at `/data/eliza1/LEGEND/software/pygama/`.
This is intended for general analysis use by non-developers, and will only work if $PYTHONPATH is updated to include this path.

## Singularity running notes
Note that singularity, unlike docker, allows you three parallel ways to [interact with an image](https://sylabs.io/guides/3.0/user-guide/quick_start.html#interact-with-images): shell, exec, or run.
Because the legend compute images are repackaged from docker images, the nuance to these options may be missing.

To use singularity, most users can open a shell in the base image with:

`singularity shell --bind /data/eliza1/LEGEND:/data/eliza1/LEGEND /data/eliza1/LEGEND/sw/containers/legend-base.sif`

(analog to `docker run legend-base bash`).
Unlike docker, the environment is supposed to mimic your out-of-container environment, your CWD hasn't changed, and common directories like /home/$USER and /tmp and $PWD are auto-mounted.
Users are encouraged to use the `--bind` flag to ensure the needed directories are available instead of relying on auto-mount of $PWD (e.g., `singularity shell --bind /data/eliza1/LEGEND:/awesome_mnt_point ...`, but matching the mount point to the filesystem path is encouraged for predictable $PYTHONPATH).

## Filesystem notes
Astute readers will notice that /data/eliza1 is unavailable on the "fast" infiniband network.
Duncan has a 10 Gb fiber from eliza1 to the switch, which provides better performance than the infiniband.
We're already short infiniband routers, so eliza1 remains disconnected, along with all the rack 5 worker nodes.
In tests, he observed the eliza systems to last up to ~50 I/O intensive jobs before slowing, whereas the other /data systems to slow at ~10 jobs.

If we need high I/O load for our workflow, we should revisit performance with Duncan.

## Singularity build notes
The singularity builds are made by converting the automatic docker container builds, as pulled manually from dockerhub.
While singularity does have its own buildfile format, this would inevitably lead to version shear or inconsistencies, and we'd rather deal with availability lag and/or the overhead.
A future enhancement could be to automate the build, possibly with a [dockerhub webhook](https://docs.docker.com/docker-hub/webhooks/).

Before building the images, two high-level cautions are provided, both because of the large image size:
The conversion is slow, taking ~45 minutes as of April 2020.
The /tmp directory used for the build artifacts is a shared resource and may fill, killing the build.

The build is fairly verbose, so there should not be >15 minute periods without receiving terminal output.
Lots of the output will be "<timestamp> warn xattr..." related to the rootless container restructuring; this can all be ignored.
The other main output is "<timestamp> info unpack layer...", allowing you to track progress of the build; larger layers will require more time to unpack.
The penultimate output is "INFO:    Creating SIF file..." (no timestamp), which will then take 10-15 minutes to create the SIF file.
Under an older singularity versions, some builds would stall indefinitely (at least a week) at this stage using ~100% CPU; it should take <<1 hr!

If the /tmp directory fills, there will be warnings indicating this when the build fails, google them to verify.
Using the `SINGULARITY_TMPDIR` environment variable can redirect these build elements to a larger storage element.

An example build interactive build sequence was:
```bash
screen
cd /data/eliza1/LEGEND/sw/containers
export SINGULARITY_TMPDIR=/data/eliza1/LEGEND/wcptest/sw/containers
date;singularity build legend-base.sif docker://legendexp/legend-base:latest;date
```
*FIXME: should also use SINGULARITY_DISABLE_CACHE?*

## Running Jobs

Follow the [CENPA Guide SGE Notes](https://cenpa.npl.washington.edu/display/CENPA/sge) for general usage. Here's an example script Jason used for running jobs executing g4simple sims in the container:

submit_job.sh:
```
#!/bin/bash

#$ -S /bin/bash #use bash
#$ -m n # don't send mail when job starts or stops.
#$ -w e #verify syntax and give error if so
#$ -V #inherit environment variables
#$ -N g4simple #job name
#$ -e /data/eliza1/LEGEND/users/jasondet/jobs/logs #error output of script
#$ -o /data/eliza1/LEGEND/users/jasondet/jobs/logs #standard output of script
#$ -l h_rt=10:00:00 #hard time limit, your job is killed if it uses this much cpu.
#$ -l s_rt=9:50:00 #soft time limit, your job gets signaled when you use this much time. Maybe you can gracefully shut down?
#$ -cwd #execute from the current working directory
#$ -t 1-1000 #give me N identical jobs, labelled by variable SGE_TASK_ID
#$ -l scratch=10G

#execute the $SGE_TASK_ID'th sub-job
singularity exec --bind /data/eliza1/LEGEND,$TMPDIR /data/eliza1/LEGEND/sw/containers/legend-base.sif /home/jasondet/sims/run1.sh
```

run1.sh runs the job tasks in the container:
```
set -x
date
source /opt/geant4/share/Geant4-10.5.1/geant4make/geant4make.sh
export LD_LIBRARY_PATH="$LD_LIBRARY_PATH:/opt/root/lib:/opt/hdf5/lib"
cd $TMPDIR
DATADIR="/data/eliza1/LEGEND/users/jasondet/jobs/data/${JOB_ID}"
mkdir -p ${DATADIR}
df -h .
g4simple /home/jasondet/sims/run.mac
ls -alh g4simpleout.hdf5
h5repack -v -f GZIP=5 g4simpleout.hdf5 ${DATADIR}/g4simpleout_${SGE_TASK_ID}.hdf5
ls -alh ${DATADIR}/g4simpleout_${SGE_TASK_ID}.hdf5
df -h ${DATADIR}
date
set +x
```
