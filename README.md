# DBT docker image

Purpose of this sample is to demonstrate how we can create docker image for DBT.


## Table of contents

 * [General Info](#general-info)
 * [Technologies](#technologies)
 * [Installation](#installation)
 * [Sections](#sections)


## General Info

In the sample, we will see what is DBT, where it can be used, installation and creating a _DBT_ docker image.


## Technologies

 * Python
 * **D**ata **B**uild **T**ool (**DBT**)
 * Docker

## Installation
 * Verify whether Python is installed in your machine. Open ```cmd``` and type ```py```. If python is installed then it will show the version of python that is installed.

> **NOTE**
> If python is not installed. Please follow instructions [here](https://www.python.org/downloads/).

 * Ones python is installed, install DBT by running below commands.

   There are multiple data platforms that are supported in *dbt*. List of platforms that are supported can be referenced [here](https://docs.getdbt.com/docs/supported-data-platforms#supported-data-platforms).

   In this sample, I'll be installing ```snowflake``` provider.

   ```pip install dbt-snowflake ```

   Run the below command to verify whether the _dbt_ version that is installed.

   ``` dbt --version ```
  
## Sections

#### Create project
 * Create a git repo for *dbt* project and clone the repo.
 * Open Powershell and navigate to repo directory.
 * Run the command to create *dbt* project ```dbt init <project name>```. This creates a skeleton project and *profiles.yml* file under .dbt folder in the users directory.
 * Provide neccessary information in profiles.yml.
```yml
dbt_sample:
  outputs:
    dev:
      account: <snowflake account name>  e.g: https://<account name>.snowflakecomputing.com/console/login
      database: snowflake db name
      password: pwd
      role: role assigned for the warehouse
      schema: schema name
      threads: 10
      type: snowflake
      user: user id
      warehouse: warehouse name
   prod:
     account:...
     .
     .
     .
     .
  target: dev // target env. is dev, so runtime will read dev env settings
```
 > **Note**
 > Name in profiles.yml should match with the profile in _dbt_sample_.
 > In this sample, name in the profiles.yml is *dbt_sample*. which matches with *profile: 'dbt_sample'* in dbt_project.yml

 * All the *sql* files will be under models folder, these sql files will be executed and transform the data accordingly.

  > _dbt_ models only has **Select** statement.


Ones the code changes are done, run the ```dbt build``` will validate the connectivity with the data providers, and runs the sql files, tests.

#### So after following the above instructions, we should be able to deploy models to snowflake database.
 


### Create docker image for the dbt application

To ensure that everyone in the team is running the same version/instance of python, dbt and also to scale the solution on demand we can run the dbt environment in a docker container which includes an explicit, tested recipe for setting up dbt with all the right dependencies.

Below is the basic docker file we can use to containerize the dbt solution.
```docker
ARG build_for=linux/amd64

FROM --platform=$build_for python:3.10.7-slim-bullseye as base

ARG dbt_snowflake_ref=dbt-snowflake@v1.2.0

# Set docker basics
WORKDIR /dbt_sample

RUN apt-get update \
    && apt-get dist-upgrade -y \
    && apt-get install -y --no-install-recommends \
        git \
        ssh-client \
        software-properties-common \
        make \
        build-essential \
        ca-certificates \
        libpq-dev \
    && apt-get clean \
    && rm -rf \
        /var/lib/apt/lists/* \
        /tmp/* \
        /var/tmp/*

ENV PYTHONIOENCODING=utf-8
ENV LANG=C.UTF-8

# Update python
RUN python -m pip install --upgrade pip setuptools wheel --no-cache-dir


# Create directory for dbt config
RUN mkdir -p /root/.dbt


FROM base as dbt-snowflake
RUN python -m pip install --no-cache-dir "git+https://github.com/dbt-labs/${dbt_snowflake_ref}#egg=dbt-snowflake"

COPY profiles.yml /root/.dbt/profiles.yml

COPY . /dbt_sample

``` 

Above dockerfile sample will help in containerizing the dbt solution that was built locally.



 - Build the docker image ``` docker build -t dbt-snowflake-test .```
 - Run the docker image ```docker run -p 8581:8580 --name dbt_run -v "/$(pwd)/dbt_sample" -it dbt-snowflake-test```

**OR**
 
 - Open the docker cli and run the below commands(ensure the docker cli is in the same directory as ```dbt_project.yml``` file).

   ```dbt build``` : which builds the dbt project (which execute the models, tests in the solution).

   ``` dbt run```  : which executes the sql files under models folder and create the views in the snowflake database (which is the configured provider in this sample).
