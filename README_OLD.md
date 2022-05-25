# Lab 1: Setup class environment



## Objective

The goal of Lab 1 is to familiarize with preparing their environment for the class. During this lab we will create a simple `FastAPI` application. This API will be containerized, tested, and allow for you to develop locally quickly.



## Environment information

My lab environment comprise a **VMware Workstation VM** running **Ubuntu 20.04** with 8 cores, 200GB disk  and 32GB RAM

ubuntu@ubuntu:~/w255/spring22-sudhrity/lab_1/lab1$ lsb_release -a
No LSB modules are available.
Distributor ID:	Ubuntu
Description:	Ubuntu 20.04.3 LTS
Release:	20.04
Codename:	focal

#### Project file directory structure

The folder structure and files for the lab are shown below under **lab_1** *(lab root directory)*

```bash
ubuntu@ubuntu:~/w255$ tree spring22-sudhrity/
spring22-sudhrity/
├── lab_1
│   ├── lab1
│   │   ├── docker-build.sh
│   │   ├── Dockerfile
│   │   ├── docker-run.sh
│   │   ├── lab1
│   │   │   ├── __init__.py
│   │   │   ├── main.py
│   │   │   ├── __pycache__
│   │   │   │   ├── __init__.cpython-310.pyc
│   │   │   │   └── main.cpython-310.pyc
│   │   │   └── run.sh
│   │   ├── poetry.lock
│   │   ├── pyproject.toml
│   │   ├── README.rst
│   │   └── tests
│   │       ├── __init__.py
│   │       ├── __pycache__
│   │       │   ├── __init__.cpython-310.pyc
│   │       │   └── test_lab1.cpython-310-pytest-6.2.5.pyc
│   │       └── test_lab1.py
│   ├── README.md
│   └── run.sh
└── README.md

```



#### Project files

- **README.md** - This file containing description and tasks of the lab including the files used for this project as listed below
- **run.sh** - Bash script to build and run docker container,  and execute the curl commands to test the api application.
- **lab1/Dockerfile** - Multistage Docker file for packaging and running the api application
- **lab1/poetry.lock** - Poetry lock file
- **lab1/pyproject.toml** - Poetry project file
- **lab1/README.rst** - Poetry project readme file
- **lab1/run.sh** - File to run docker locally
- **lab1/build.sh** - File to build docker
- **lab1/tests** - Poetry project unit test files directory
- **lab1/lab1** - Poetry project API implementation file directory



## Instructions to build, run, and test code in app root directory

A **run.sh** script is provided in the lab_1 root directory to build, run and test the lab_1 API application. This is shown below.

```bash
#!/bin/bash

# This bash script builds and runs lab_1

# Change directory to lab root directory
cd lab1

# Stop container if running
docker stop lab1

# Build the docker container
docker build --no-cache -t sudhrity/lab1:v1 -f Dockerfile .

# Run the docker container
docker run -d --name lab1 -it --rm --network host sudhrity/lab1:v1

# Sleep for API service to be come up, prior to running the url commands
sleep 10

# Run curl commands for basic test on the API service
curl -o /dev/null -s -w "%{http_code}\n" -X GET "http://localhost:8000/hello?name=Winegar"
curl -o /dev/null -s -w "%{http_code}\n" -X GET "http://localhost:8000/"
curl -o /dev/null -s -w "%{http_code}\n" -X GET "http://localhost:8000/docs"

# Run unit tests using pytest
poetry run pytest
```

The command used to build the docker container for **lab_1**  is shown below:

```bash
docker build --no-cache -t sudhrity/lab1:v1 -f Dockerfile .
```

The command used to run the docker container for **lab_1**  is shown below:

```bash
docker run -d --name lab1 -it --rm --network host sudhrity/lab1:v1
```

The command used to run unit tests is shown below:

```bash
poetry run pytest
```



#### Dockerfile for lab_1

The *Dockerfile* used for **lab_1** in *lab_1/lab1* is shown below:

```bash
# Dockerfile
## Build venv
FROM python:3.10-buster AS venv

ENV POETRY_VERSION=1.1.12
RUN curl -sSL https://install.python-poetry.org | python -
ENV PATH /root/.local/bin:$PATH

RUN poetry self update

WORKDIR /app
COPY pyproject.toml poetry.lock ./

# The `--copies` option tells `venv` to copy libs and binaries
# instead of using links (which could break since we will
# extract the virtualenv from this image)
RUN python -m venv --copies /app/venv
RUN . /app/venv/bin/activate && poetry install

## Beginning of runtime image
# Remember to use the same python version
# and the same base distro as the venv image
FROM python:3.10-slim-buster as prod

# alpine image wasn't working
#FROM alpine:latest as prod

COPY --from=venv /app/venv /app/venv/
ENV PATH /app/venv/bin:$PATH

WORKDIR /app
COPY . ./

WORKDIR /app/lab1
CMD ["/app/lab1/run.sh"]

```



#### API Implementation

The *API implementation* for **lab_1** is done in *lab_1/lab1/lab1/main.py* and is shown below:

```python
from fastapi import FastAPI, HTTPException
app = FastAPI()

# Implementation of API endpoint '/hello' - takes a query parameter name
# for displaying hello [value]
@app.get("/hello")
#async def read_item(name: str | int):
async def read_item(name: str):

#    if name == None or name == "":
        # I have selected error 422 because the request was well-formed but
        # was unable to be followed due to semantic errors. This is a client
        # side error. The if condition is added, so that name without a value
        # returns an error.  Without the if condition
        # the method returns a 404 "Not found "

#       raise HTTPException(status_code=422,
#               detail="Unprocessable Entity. HTTP status_code=422")

    return {"message": "hello " + name}

# API endpoint '/' - Not implemented
# I added this so that a 501 is returned. If this method is not
# implemented, a "404  - Not found" is returned
@app.get("/")
async def root():
    raise HTTPException(status_code=501,
        detail="Not Implemented, HTTP status_code=501")
```



#### API Unit Test Implementation

The *API Unit Test implementation* for **lab_1** is done in *lab_1/lab1/tests/test_lab1.py* and is shown below:

```python
from lab1 import __version__
from fastapi.testclient import TestClient

from lab1.main import app

client = TestClient(app)

def test_version():
    assert __version__ == '0.1.0'

# Unit tests for API
def test_read_main1():
    response = client.get("/hello?name=Winegar")
    assert response.status_code == 200
    assert response.json() == {"message": "hello Winegar"}

def test_read_main2():
    response = client.get("/")
    assert response.status_code == 501
    assert response.json() == {"detail": "Not Implemented, HTTP status_code=501"}

def test_read_main3():
    response = client.get("/docs")
    assert response.status_code == 200

def test_read_main4():
    response = client.get("/openapi.json")
    assert response.status_code == 200

def test_read_main5():
    response = client.get("/hello?address=Winegar")
    assert response.status_code == 422
    assert response.json() == {"detail": [{'loc': ['query', 'name'],
                'msg': 'field required', 'type': 'value_error.missing'}]}

def test_read_main6():
    response = client.get("/hello/address=Winegar")
    assert response.status_code == 404
    assert response.json() == {"detail": "Not Found"}

def test_read_main7():
    response = client.get("/hello/Winegar")
    assert response.status_code == 404
    assert response.json() == {"detail": "Not Found"}

def test_read_main8():
    response = client.get("/hello?name=")
    assert response.status_code == 200
#    assert response.json() == {"detail": "Unprocessable Entity. HTTP status_code=422"}
    assert response.json() == {"message": "hello "}

```




## Environment Configuration and Development Steps

- Setup steps
  - Install and configure Azure CLI
  - Install and configure Kubectl
  - Install Python 3.10
  - Install Poetry environment and create lab1 project
  - Install Docker
  - Build docker container
  
- Development steps
  - Implement API (*lab1/lab1/main.py*)
  - Implement Unit Tests (*lab1/tests/test_lab1.py*)
  - Implement the *lab_1/run.sh* script




## Test Results

The execution results of **lab_1/run.sh** is shown below:

```bash
ubuntu@ubuntu:~/w255/spring22-sudhrity/lab_1$ ./run.sh
lab1
Sending build context to Docker daemon  60.93kB
Step 1/16 : FROM python:3.10-buster AS venv
 ---> 06132ec49b06
Step 2/16 : ENV POETRY_VERSION=1.1.12
 ---> Using cache
 ---> 524e6cacb984
Step 3/16 : RUN curl -sSL https://install.python-poetry.org | python -
 ---> Using cache
 ---> 4580cffe4663
Step 4/16 : ENV PATH /root/.local/bin:$PATH
 ---> Using cache
 ---> dbd9f233e166
Step 5/16 : RUN poetry self update
 ---> Using cache
 ---> 194f40ea3b22
Step 6/16 : WORKDIR /app
 ---> Using cache
 ---> 654674f78b49
Step 7/16 : COPY pyproject.toml poetry.lock ./
 ---> Using cache
 ---> 3edfb8d36a71
Step 8/16 : RUN python -m venv --copies /app/venv
 ---> Using cache
 ---> 1ae14b4f0462
Step 9/16 : RUN . /app/venv/bin/activate && poetry install
 ---> Using cache
 ---> 1fb5bd3b7275
Step 10/16 : FROM python:3.10-slim-buster as prod
 ---> 84b629347b26
Step 11/16 : COPY --from=venv /app/venv /app/venv/
 ---> Using cache
 ---> 49f8c1786ee9
Step 12/16 : ENV PATH /app/venv/bin:$PATH
 ---> Using cache
 ---> 14d925418cf5
Step 13/16 : WORKDIR /app
 ---> Using cache
 ---> a147a5362bdc
Step 14/16 : COPY . ./
 ---> 2a0355bb6f86
Step 15/16 : WORKDIR /app/lab1
 ---> Running in ac33a0f03fbd
Removing intermediate container ac33a0f03fbd
 ---> b55ae981b17d
Step 16/16 : CMD ["/app/lab1/run.sh"]
 ---> Running in c670d23f6082
Removing intermediate container c670d23f6082
 ---> 5147beef0f3d
Successfully built 5147beef0f3d
Successfully tagged sudhrity/lab1:v1
a3f41354b4807d2e3ad3027945902b4861007df84cf06e6bae9cae9084470100
200
501
200
================================================================================================== test session starts ==================================================================================================
platform linux -- Python 3.10.2, pytest-6.2.5, py-1.11.0, pluggy-0.13.1
rootdir: /home/ubuntu/w255/spring22-sudhrity/lab_1/lab1
plugins: anyio-3.5.0
collected 9 items                                                                                                                                                                                                       

tests/test_lab1.py .........                                                                                                                                                                                      [100%]

=================================================================================================== 9 passed in 0.18s ===================================================================================================
ubuntu@ubuntu:~/w255/spring22-sudhrity/lab_1$

```







