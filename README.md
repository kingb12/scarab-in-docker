# My Docker Development Environment

This is a repo for creating a docker image which I use to develop my research projects. It is not for deployment, and typically neither are the applications developed using it. It is not intended to be lightweight or plug-and-play ready, but where possible, I tried to make it extensible for other users. Specifically, each of my projects initiates a new Dockerfile from this image, creates its own virtual environemtn and installs project dependencies, plus any other related project set up.

## Motivation

The features below facilitate a workflow like so:

**Development (Pod)**:
- Remote debugging with PyCharm. The gist of how this works is:
  - create a project-specific image which uses installed conda to create a virtual environment for execution and installing project specific dependencies
  - launch a pod running SSHD with a non-privileged user on a non-privileged port (`2022` here)
  - port forward to your local machine: `kubectl port-forward pods/<your_pod_name> 2022:2022`
  - set up an SSH interpreter in PyCharm using `localhost` and port `2022`, pointing to the virtual environment `venv/bin/python` on your container
  - Debug w/ automatic code syncing to the pod
  
**Running Experiments (Job)**
- Clone updated code before job launch
  - This takes some careful job definition, see an example with a few other useful things below, that likely requires adjustments before use.
  - Requires setting a Secret in k8s with `SSH_PRIVATE_KEY`, `SSH_PUBLIC_KEY`. Recommended to not using your personal ssh key here, but create a new one, since this may be visible to anyone in the k8s namespace. You can then register this new key as a [deployment key](https://docs.github.com/en/developers/overview/managing-deploy-keys#deploy-keys) for the appropriate private Github repo(s).
- Run!
 
 #### Example Job `command`
 
 Requires a secret defining the SSH key (public and private, load as environment variables), and if needed, a Huggingface API token (also env-variable loaded).
 
 ```yaml
         command: [ "/bin/sh" ]
        # commmand does the following:
        # 1) set up the github deployment (SSH) key: we store the relevant secrets as environment variables, because
        #    injecting them as files makes root the owner, and the container needs to run as non-privileged user. We
        #    also add githubs public keys to known_hosts to bypass the interactive fingerprint check on later clones
        # 2) clone repo: this clones the repo into temp, since the folder already exists and contains our venv
        #    definition that we don't want to overwrite. Then, we move the .git definition into the folder (which had
        #    none to begin with), and fetch and pull. This doesn't yet overwrite files, we then need to do a hard reset to
        #    origin/main (assuming this is the branch we are always running jobs from). This step allows us to not re-build
        #    the docker container for every code change, only those which are important to it.
        # 3) log in the huggingface for private model access and saving
        # 4) run the actual job: everything after 'job ready to start' is the script we want to run
        args:
          - -c
          - >-
            echo "$SSH_PRIVATE_KEY" > /home/bking2/.ssh/id_rsa &&
            echo "$SSH_PUBLIC_KEY" > /home/bking2/.ssh/id_rsa.pub &&
            chmod 644 /home/bking2/.ssh/id_rsa.pub &&
            chmod 600 /home/bking2/.ssh/id_rsa &&
            echo "github.com ssh-rsa AAAAB3NzaC1yc2EAAAABIwAAAQEAq2A7hRGmdnm9tUDbO9IDSwBK6TbQa+PXYPCPy6rbTrTtw7PHkccKrpp0yVhp5HdEIcKr6pLlVDBfOLX9QUsyCOV0wzfjIJNlGEYsdlLJizHhbn2mUjvSAHQqZETYP81eFzLQNnPHt4EVVUh7VfDESU84KezmD5QlWpXLmvU31/yMf+Se8xhHTvKSCZIFImWwoG6mbUoWf9nzpIoaSjB+weqqUUmpaaasXVal72J+UX2B+2RPW3RcT0eOzQgqlJL3RKrTJvdsjE3JEAvGq3lGHSZXy28G3skua2SmVi/w4yCE6gbODqnTWlg7+wC604ydGXA8VJiS5ap43JXiUFFAaQ==" >> /home/bking2/.ssh/known_hosts &&
            echo "github.com ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBEmKSENjQEezOmxkZMy7opKgwFB9nkt5YRrYMjNuG5N87uRgg6CLrbo5wAdT/y6v0mKV0U2w0WZ2YB/++Tpockg=" >> /home/bking2/.ssh/known_hosts &&
            echo "github.com ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIOMqqnkVzrm0SdG6UOoqKLsabgH5C9okWi0dh2l9GKJl" >> /home/bking2/.ssh/known_hosts &&
            cd /home/bking2 &&
            git clone git@github.com:kingb12/my_project.git temp &&
            mv temp/.git my_project/.git &&
            rm -rf temp &&
            cd my_project &&
            git fetch && git pull && git reset --hard origin/main &&
            python -c "from huggingface_hub.hf_api import HfFolder; HfFolder.save_token('${HF_API_TOKEN}')" &&
            echo "job ready to start" &&
            <actual job command(s) here>
 ```


## Features

#### SSH & SFTP Access
A non-privileged run of `sshd` on the container. combined with port forwarding on a non-privileged port like `2022`, this allows direct `ssh` and `sftp` access to the container, useful for setting up a remote interpreter in PyCharm.

##### Relevant Files
- `Dockerfile`: installs and sets up sshd, including the default command to run it on container launch (need to set this as part of the command as well when running in k8s)
- `sshd_config`: the config copied into place for the running `sshd` service
- `sshd-1.service`: a service definition for our non-privileged run of `sshd`
- `id_rsa.pub`: MY ssh public key. Replace with yours!

### Miniconda

Miniconda pre-installed, no significant packages added. Images should extend this one and run conda environment set up.

### github.com known hosts

See warning about the security of this, this bypasses the fingerprint check in git clone via ssh by pre-loading the public keys for github.com, so that a repo can be clone on container start. You're implicitly trusting that I did this correctly, see [this ServerFault discussion](https://serverfault.com/a/701637) for details.

### ZSH and oh-my-zsh

I install zsh and my own personal dotfile preferences, only some of which are generally applicable. Adjust or remove as needed.

### (my) ssh config

I found it useful to be able to ssh from my container to other hosts I use, especially for copying files while debugging and/or placing them in a mounted volume. Remove this build step if it doesn't sound useful or replace with your own config.
