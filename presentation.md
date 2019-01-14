title: Concourse CI
class: animation-fade
layout: true

<!-- This slide will serve as the base layout for all your slides -->
.bottom-bar[
  {{title}}
]

---

class: impact no-counter

# {{title}}
## Mario Fernandez

---

class: center middle

# Outline

.menu[
- .item[🔗]
- .item[Initially...]
- .item[Basics]
- .item[CLI]
- .item[Pipelines]
- .item[UI]
- .item[.green[✓]]
- .item[🤔]
- .item[.red[⨯]]
]

---

class: impact no-counter

# Initially...

---


.col-6[
  <img src="images/jenkins.png" alt="jenkins" width="512" height="512"/>
]

.col-6[
  <img src="images/snowflake.jpg" alt="snowflake" width="512" height="512"/>
]

---

class: center middle

## thousands of lines of a self written groovy DSL

???

- predates jenkins pipelines

---

class: center middle

## 4 test systems

???

- still on premise

---

class: center middle

## self written deployment framework based on Chef

???

- unsuited for cloud migration

---

class: center middle

## decided to explore options, and chose Concourse

???

- situation was so bad that anything would have been an improvement

---

class: impact no-counter

# The basics

---

class: center middle

## Written in Go

???

- pivotal

---

class: full-height
background-image: url(images/architecture.png)

---

class: full-height
background-image: url(images/arch-simple.png)

---

class: center middle

## everything runs in containers

---

class: center middle

## yaml based

???

- code driven

---

class: center middle

## take it or leave it approach

---

class: impact no-counter

# CLI

---

class: center middle

## fly

---

# Easily available

```docker
FROM alpine:3.8

ENV SHA1='7b0805adfba328d09cfa7ac377777cb6e270d66f' \
    VERSION='4.2.1' \
    URL='https://github.com/concourse/concourse/releases/download'

RUN apk update && \
    apk --no-cache add curl bash

SHELL ["/bin/bash", "-o", "pipefail", "-c"]

RUN curl -Lk "${URL}/v${VERSION}/fly_linux_amd64" -o /usr/bin/fly && \
    echo "${SHA1}  /usr/bin/fly" | sha1sum -c - && \
    chmod +x /usr/bin/fly
```

---

# Update pipeline

```console
version 4.2.1 already matches; skipping
jobs:
  
  job audit has been removed:
  - name: audit
  - serial: true
  - plan:
  - - aggregate:
  -   - get: git
  -   - get: shared-tasks
  - - task: audit
  -   file: shared-tasks/tasks/yarn/audit/task.yml
  -   image: dev-container
  
configuration updated

```

---

class: center middle quirk

## pipelines don't update automatically

---

# Status

```console
$ fly -t ci workers

name              containers platform tags team state   version
ip-1.aws.compute  14         linux    none none running 2.1
ip-2.aws.compute  10         linux    none none running 2.1
ip-3.aws.compute  18         linux    none none running 2.1


the following workers have not checked in recently:

name              containers platform tags team state   version
ip-4.aws.compute  1          linux    none none running 2.1

these stalled workers can be cleaned up by running:

    fly -t ci prune-worker -w (name)
```

---

# Hijack

```console
$ fly -t concourse hijack -j service-api/test
1: build #95, step: test-integration, type: task
2: build #95, step: test-unit, type: task
choose a container: 2
bash-4.3# pwd
/tmp/build/1944c6a2
bash-4.3# cd git
bash-4.3# ./go test-unit
```

---

class: center middle quirk

## Hijack won't work if the container does not have bash

---

# And many more

```console
Available commands:
  abort-build          Abort a build (aliases: ab)
  builds               List builds data (aliases: bs)
  check-resource       Check a resource (aliases: cr)
  check-resource-type  Check a resource-type (aliases: crt)
  checklist            Print a Checkfile of the given pipeline (aliases: cl)
  clear-task-cache     Clears cache from a task container (aliases: ctc)
  containers           Print the active containers (aliases: cs)
  destroy-pipeline     Destroy a pipeline (aliases: dp)
  destroy-team         Destroy a team and delete all of its data (aliases: dt)
  execute              Execute a one-off build using local bits (aliases: e)
  (...)
```

???

- CLI approach can be scary for people used to the click, click, click of Jenkins

---

class: impact no-counter

# Defining pipelines

---

class: full-width
background-image: url(images/pipeline.png)

---

class: middle

# Building blocks

- ## Resources
- ## Jobs
- ## Tasks

---

class: center middle

# Resources

---

class: center middle

## artifacts used in the pipeline

---

class: center middle

## fetched from and pushed to

---

class: middle

# Types

- ## git
- ## docker
- ## s3
- ## ...

## https://concourse-ci.org/included-resources.html

???

- funky resources like time

---

```yaml
resources:
- name: git
  type: git
  source:
    uri: "git@github.com:sirech/example-concourse-pipeline.git"
    branch: master

- name: dev-container
  type: docker-image
  source:
    repository: "AWS_ACCOUNT.aws.com/dev-container"
```

---

class: center middle

## extensibility through new resource types

???

- github pull request
- slack integration
- ftp

---

class: center middle quirk

## it is the only way to store artifacts in concourse

???

- can be painful for things like getting screenshots produced by an e2e task

---

class: center middle

# Jobs

---

class: center middle

## **Jobs** determine the actions of your pipeline

???

- quoting the docs

---

```yaml
- name: lint
  serial: true
  plan:
  - aggregate:
    - get: git
      passed: [prepare] 
      trigger: true
    - get: dev-container
      passed: [prepare]
  - aggregate:
    - task: lint-sh
      image: dev-container
      params:
        <<: *common-params
        TARGET: sh
      file: "git/pipeline/tasks/linter/task.yml"
```

???

- the **plan** is the sequence of steps to execute
- sequential by default, **aggregate** allows concurrency
- interact with resources through **get**/**put**
 - **passed** builds dependencies to other jobs
 
---

class: center middle quirk

## you cannot pass anything between **jobs**

---

class: center middle quirk

## full pipeline cannot be retriggered

---

class: center middle

# Tasks

---

```yaml
- task: lint-js
  image: dev-container
  params:
    NPM_TOKEN: ((npm_auth_token))
    TARGET: js
  file: "git/pipeline/tasks/linter/task.yml"
```

???

- a task always needs an **image**
- params can be passed as env vars
 - using secrets is very convenient
- code: inline or as separate file
 - reuse task definitions
 
---

```yaml
platform: linux
inputs:
  - name: git
caches:
  - path: "git/node_modules"
params:
  TARGET:
run:
  path: "pipeline/tasks/linter/task.sh"
  dir: git
```

???

- inputs map resources or outputs to a folder
- run defines the command to execute
 - inline or script
 - dir, args
- **params** allow parametrization
- caching through **caches**

---

class: center middle quirk

## WORKDIR, ENTRYPOINT, USER, CMD are ignored

---

```yaml
#!/bin/sh

set -e

yarn
./go "linter-${TARGET}"
```

???

- regular script that can be executed locally

---

class: center middle

## reuse through shared tasks

???

- extra repo as new resource
- use parametrized tasks
- still, plenty of yaml

---

class: center middle

## shared containers

???

- a lot of containers need to be built, or one massive one
- tricky decission between reuse and repeat

---

class: center middle

## no arbitrary code

???

- It is a DSL
- you cannot run groovy scripts, like in jenkins
 - good: no magic untestable code
 - bad: a lot of repetition

---

class: impact no-counter

# The UI

---

class: full-width
background-image: url(images/failingpipeline.png)

???

- hard to see which tasks are manually triggered
- hard to see if there is a new version that needs releasing

---

class: full-height
background-image: url(images/dashboard.png)

---

class: impact no-counter

# ✓

---

class: center middle

## code driven

---

class: center middle

## isolation

---

class: center middle

## convenience (paralellization, caching)

---

class: center middle

## limiting (in a good way)

???

- the "in a good way" can be argued

---

class: impact no-counter

#  🤔

---

class: center middle

## cannot save artifacts

---

class: center middle

## does not update pipelines on push

---

class: center middle

## ignores some Docker directives

---

class: impact no-counter

# ⨯

---

class: center middle

## sparse documentation

???

- documentation missing, or incorrect

---

class: center middle

## operational burden

???

- cryptic errors
- does not really self report its health
- multiple downtimes

---

class: impact no-counter middle

# 🔗

---

class: center middle

### https://www.thoughtworks.com/insights/blog/modernizing-your-build-pipelines
### https://github.com/sirech/example-concourse-pipeline

---



