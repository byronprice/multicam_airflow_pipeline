# Multicamera 3D Keypoint and MoSeq Pipeline with Airflow Automation

This library is comprised of a series of analysis tasks for working with video data collected with the library [multicamera_acquisition](https://github.com/dattalab-6-cam/multicamera_acquisition). 

The first version of the pipeline is stored in multicamera_airflow_pipeline.tim_240731. It comprises the following steps:
- 2D keypoint prediction
- Camera sync
- 3D triangulation
- Gimbal refinement
- Size normalization
- Arena alignment (align 3d keypoints to rectangular arena)
- Egocentric alignment to mouse
- Continuous variable computation
- Sync ephys to camera frames
- Spikesorting (kilosort4) ephys signals

Because these steps are computed in a somewhat a complex dependency graph, we use Apache Airflow to manage the pipeline. In addition, these tasks are too computationally expensive to run on a single computer, so we have methods to run tasks on both the HMS data cluster [O2](https://harvardmed.atlassian.net/wiki/spaces/O2/overview) and locally. O2 is a slurm cluster; in principal this code should work equally well on other slurm clusters. For example, we could use the [HMS Longwood cluster](https://harvardmed.atlassian.net/wiki/spaces/Longwood). 

# Installation

I reccomend creating a separate conda environment for different steps of the pipeline. E.g. a `pipeline` environment for general tasks, a `jax` environment for Gimbal & KPMS, an `mmdeploy` environment for 2D keypoint inference, a `spikesorting` environment for kilosort/spikesorting, and an `airflow` environment for running airflow. 

The pipeline library can be installed in each of these environments with `pip install -v .e` after navigating to the repo directory. 

# How to use this code
There are three ways you can interact with this pipeline. 
## 1. **Run python functions directly.** 
Each processing step can be run individually. For example, you can run the camera synchronization function directly in python by pointing it at the video directory. This outputs a `camera_sync.csv` in `output_directory`. 

```{python}
from multicamera_airflow_pipeline.tim_240731.sync.sync_cameras import CameraSynchronizer
synchronizer = CameraSynchronizer(
    recording_directory = my_video_directory, # where the video is stored
    output_directory = output_directory_camera_sync, # directory to save the output csv
    samplerate=150, # samplerate of video
    trigger_pin=2, # which pin controls the camera (usually 2 or 3)
)
synchronizer.run()
```
## 2. **Run O2 jobs without Airflow.** 
If you want to run a single step of the pipeline on O2, you can submit a job to O2 directly. To aid in job submission we have a class `O2Runner`, which submits the job for you and keeps track of its progress in the slurm cluster. 

 Note that you need to set up SSH keys so that you can run commands on O2 before using this code to run an O2 job. For example, here is how we would run the camera sync as a direct O2 job: 
```{python}
from multicamera_airflow_pipeline.tim_240731.interface.o2 import O2Runner

runner = O2Runner(
    job_name_prefix = 'test_submit_camera_sync',
    remote_job_directory = remote_job_directory,
    conda_env = "/n/groups/datta/tim_sainburg/conda_envs/peromoseq",
    o2_username = "tis697",
    o2_server="login.o2.rc.hms.harvard.edu",
    job_params = params, 
    o2_n_cpus = 1,
    o2_memory="2G",
    o2_time_limit=duration_requested,
    o2_queue="priority",
)
runner.python_script = f"""
# load params
import yaml
params_file = "{runner.remote_job_directory / f"{runner.job_name}.params.yaml"}"
config_file = "{config_file.as_posix()}"

params = yaml.safe_load(open(params_file, 'r'))
config = yaml.safe_load(open(config_file, 'r'))
    
# grab sync cameras function
from multicamera_airflow_pipeline.tim_240731.sync.sync_cameras import CameraSynchronizer
synchronizer = CameraSynchronizer(
    recording_directory=params["recording_directory"],
    output_directory=params["output_directory_camera_sync"],
    samplerate=params["samplerate"],  # camera sample rate
    # trigger_pin=params["trigger_pin"],  # Which pin camera trigger was on
    **config["sync_cameras"],
)
synchronizer.run()
"""
runner.run()

runner.run() # submits the job
```

### Setting up SSH keys
To run jobs on o2, we need to SSH into `login.o2.rc.hms.harvard.edu` with python. To make this possible, we need to be able to login without entering our password interactively each time. Google "How to set up SSH keys" if you don't already know how to do this. 

## 3. **Run O2 jobs with Airflow**. 
This requires setting up an Airflow server which automatically maintains jobs for you. 

### How Airflow works
1. Airflow runs a local computer (my desktop).
2. Airflow reads which recordings to process from a [google sheet](https://docs.google.com/spreadsheets/d/1jACsUmxuJ9Une59qmvzZGc1qXezKhKzD1zho2sEfcrU/edit?gid=0#gid=0)
3. For each recording, Airflow generates a DAG with all of the `tasks` it needs to run. 
4. `tasks` are submitted to o2 using `sbatch` after waiting until their dependencies are completed (i.e. the tasks that need to precede it)
5. Airflow automates the process of running each step in succession, and keeps track of which tasks have run successfully, and which have failed (and how they've failed.)


### Creating a new pipeline
The best way to create a new pipeline is to copy a pipeline folder (e.g."tim_240731"), modify/delete steps you don't need, and add in new steps. 

### Creating a new pipeline step
New pipeline steps requires two components. 
1. Create a python function or class to run your processing step. E.g. the `CameraSynchronizer` class in `multicamera_airflow_pipeline.tim_240731.sync.sync_cameras`.
2. Create a an Airflow `task` to run. For example, the task for camera synchronization is located at `multicamera_airflow_pipeline.tim_240731.airflow.jobs.sync_cameras.py`. `sync_cameras` is a function that runs locally on your machine, which submits a job on o2 with `O2Runner`. These jobs are called in order by the Airflow DAG. 

### Create a new DAG
When you create a new pipeline, you need to specify the order of Airflow tasks to run. My Airflow DAG is located at `multicamera_airflow_pipeline.tim_240731.airflow.dag`

The Airflow DAG is represented by a class `AirflowDAG`. Calling AirflowDAG.run() loops through each recording and generates a pipeline DAG for it. 

You don't need to run `AirflowDAG.run()` yourself. Airflow automates this process by referencing it in Airflow's DAG folder (by default `~/airflow/dags`, see section below "Adding your Airflow job"). 

### Setting up airflow on your local machine

First, install airflow on your local computer. 

```
# create an airflow conda environment
```

You can check that the Airflow install worked with the following steps:

Start the airflow scheduler in terminal (alternatively, use tmux to run these in the background):
```
airflow scheduler
```
Start the airflow web server in a separate tab or tmux
```
airflow webserver
```
Finally, navigate to `http://localhost:8080/` in your browser to check that it works. You won't be able to log in until you create a user profile (see below).


By default, Airflow does not allow for parallelization. To setup parallelization, we need to switch to `postgresql` from `SQLite`. 

First, quit `airflow scheduler` and `airflow webserver`.

Next, install `postgresql`

```
sudo apt update
sudo apt install postgresql postgresql-contrib
sudo systemctl start postgresql
sudo systemctl enable postgresql
```

Setup the database. This first command will open an interactive session within your current one.
```
sudo -i -u postgres
psql
CREATE DATABASE airflow_db;
CREATE USER airflow_user WITH PASSWORD 'airflow_pass';
GRANT ALL PRIVILEGES ON DATABASE airflow_db TO airflow_user;
-- PostgreSQL 15 requires additional privileges:
GRANT ALL ON SCHEMA public TO airflow_user;
\q
```

Now exit the interactive session with Ctrl-D or equivalent. Then reload postgres:
```
sudo systemctl reload postgresql
```

Update the airflow config (usually at `~/airflow/airflow.cfg`) from
```
#sql_alchemy_conn = sqlite:////home/peromoseq/airflow/airflow.db
```
to 
```
sql_alchemy_conn = postgresql+psycopg2://airflow_user:airflow_pass@localhost/airflow_db
```
Then create a username and password to use. This username and password will allow you to login when you go to `http://localhost:8080/`.
```
airflow users create \
    --role Admin \
    --username timsainb \
    --password <password> \
    --firstname tim \
    --lastname sainburg \
    --email timsainb@gmail.com
```

You may have to install psycopg2: `pip install psycopg2-binary`

Finally, we need to switch airflow config setting from `SequentialExecutor` in `airflow.cfg` to allow for parallelization.

```
executor = LocalExecutor
```

Now airflow is parallelizable. 

To test, start `airflow scheduler` and `airflow webserver` and navigate to `http://localhost:8080/`




### Mount /n/groups/datta using `sshfs`
First, set up your ssh key to O2. To be able to check whether files are present, I mount `/n/groups/datta` locally using `sshfs`. Specifically, I set up `sshfs` so that `/n/groups/datta/` is mounted in `/n/groups/datta` in my local computer. (You will probably need to create your own local /n directory.) Google 'how to install sshfs' if you don't already have it installed. The command should look something like:
```
sshfs jop9552@transfer.rc.hms.harvard.edu:/n/groups/datta /n/groups/datta/
```

In principle, this shouldn't be strictly necessary, but for my jobs I use `sshfs` to check whether files are present without having to SSH. In future updates, we could remove the need for sshfs.

### Starting Airflow
- To run airflow, in one terminal tab type: `airflow webserver`
    - alternatively, use tmux to run these in the background
- In a second tab, type: `airflow scheduler`
- Then navigate to `http://localhost:8080/` in your browser. 

### Adding your Airflow job
Airflow DAGs are generated dynamically each time you start the scheduler. Ideally, we would also want DAGs to be generated when new rows are added to your google sheet. In the example located in `airflow_examples/example_dag.py`, I create a second DAG, `refresh_pipeline_dag` which does this by 'touching' the file containing it. Airflow looks for when dag files have changed to rerun them, so everytime `refresh_pipeline_dag`, it will also re-read the google sheet. 


## TODO / future tasks
- add longwood cluster to DAG
- estimate unit locations (?)
- add keypoint moseq
- create a DAG that runs fully locally/without o2. 
