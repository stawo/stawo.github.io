---
title: Shell environment variables in self-hosted Github runner
tags:
  - github
  - env
---

Assume we set up a self-hosted Github runner, for example on an Ubuntu virtual machine.  
For whatever reason, we need to define some custom environment variables and make sure that they are accessible in all steps of the workflows executed by the Github runner.
An example could be if we install a software and have to update the env variable `PATH` to make sure we can call it.  
Or we want to set a specific identifier for the VM `VM_ID` and make it available to the github workflow.

To achieve it, we can create (or modify) the file `.profile` in the user folder and set the content to something like this:
```bash
export PATH="<path we want to add>:$PATH"
export VM_ID="<desired value>"
```

Now, we can verify


- create a `shell` script under `/etc/profile.d/` (e.g., `/etc/profile.d/set_envs.sh`)
- reboot the VM
- verify the correctness by executing a Github workflow with the following step:
  ```yaml
  - name: Check Conda local install
        run: |
          echo $PATH
          echo $CONDA
          conda -h
          conda env list
  ```

Notes:
- I tried setting `/etc/environment` but it did not work.


Did this trying to solve the following issue: https://github.com/conda-incubator/setup-miniconda/issues/218
but it seems there are other problems and it did not help.

Resources:
- https://help.ubuntu.com/community/EnvironmentVariables#System-wide_environment_variables