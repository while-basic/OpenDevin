# 💿 How to Create a Custom Docker Sandbox

The default OpenDevin sandbox comes with a [minimal ubuntu configuration](https://github.com/OpenDevin/OpenDevin/blob/main/containers/sandbox/Dockerfile). 

Your use case may need additional software installed by default.

There are two ways you can do so:
1. Use an existing image from docker hub. For instance, if you want to have `nodejs` installed, you can do so by using the `node:21` image.
2. Creating your own custom docker image.

If you want to take the first approach, you can skip the next section.

## Setup

Make sure you are able to run OpenDevin using the [Development.md](https://github.com/OpenDevin/OpenDevin/blob/main/Development.md) first.

## Create Your Docker Image
To create a custom docker image, it must be debian/ubuntu based. 

For example, if we want OpenDevin to have access to the "node" binary, we would use the following Dockerfile:

```dockerfile
# Start with latest ubuntu image
FROM ubuntu:latest

# Run needed updates
RUN apt-get update && apt-get install -y

# Install node
RUN apt-get install -y nodejs
```

Next build your docker image with the name of your choice, for example "custom_image". 

To do this you can create a directory and put your file inside it with the name "Dockerfile", and inside the directory run the following command:

```bash
docker build -t custom_image .
```

This will produce a new image called ```custom_image``` that will be available in Docker Engine.

> Note that in the configuration described in this document, OpenDevin will run as user "opendevin" inside the sandbox and thus all packages installed via the docker file should be available to all users on the system, not just root.
>
> Installing with apt-get above installs node for all users.

## Specify your sandbox image in config.toml file

OpenDevin configuration occurs via the top-level `config.toml` file. 

Create a `config.toml` file in the OpenDevin directory and enter these contents:

```toml
[core]
workspace_base="./workspace"
persist_sandbox=false
run_as_devin=true
sandbox_container_image="custom_image"
```

For sandbox_container_image, you can specify either:
1. The name of your custom image that you built in the previous step (e.g., "custom_image")
2. A pre-existing image from Docker Hub (e.g., "node:21" if you want a sandbox with Node.js pre-installed)

## Run
Run OpenDevin by running ```make run``` in the top level directory.

Navigate to ```localhost:3001``` and check if your desired dependencies are available.

In the case of the example above, running ```node -v``` in the terminal produces ```v18.19.1```

Congratulations!

## Technical Explanation

The relevant code is defined in [ssh_box.py](https://github.com/OpenDevin/OpenDevin/blob/main/opendevin/runtime/docker/ssh_box.py) and [image_agnostic_util.py](https://github.com/OpenDevin/OpenDevin/blob/main/opendevin/runtime/docker/image_agnostic_util.py).

In particular, ssh_box.py checks the config object for ```config.sandbox_container_image``` and then attempts to retrieve the image using [get_od_sandbox_image](https://github.com/OpenDevin/OpenDevin/blob/main/opendevin/runtime/docker/image_agnostic_util.py#L72) which is defined in image_agnostic_util.py.

When first using a custom image, it will not be found and thus it will be built (on subsequent runs the built image will be found and returned).

The custom image is built using [_build_sandbox_image()](https://github.com/OpenDevin/OpenDevin/blob/main/opendevin/runtime/docker/image_agnostic_util.py#L29), which creates a docker file using your custom_image as a base and then configures the environment for OpenDevin, like this:

```python
dockerfile_content = (
        f'FROM {base_image}\n'
        'RUN apt update && apt install -y openssh-server wget sudo\n'
        'RUN mkdir -p -m0755 /var/run/sshd\n'
        'RUN mkdir -p /opendevin && mkdir -p /opendevin/logs && chmod 777 /opendevin/logs\n'
        'RUN wget "https://github.com/conda-forge/miniforge/releases/latest/download/Miniforge3-$(uname)-$(uname -m).sh"\n'
        'RUN bash Miniforge3-$(uname)-$(uname -m).sh -b -p /opendevin/miniforge3\n'
        'RUN bash -c ". /opendevin/miniforge3/etc/profile.d/conda.sh && conda config --set changeps1 False && conda config --append channels conda-forge"\n'
        'RUN echo "export PATH=/opendevin/miniforge3/bin:$PATH" >> ~/.bashrc\n'
        'RUN echo "export PATH=/opendevin/miniforge3/bin:$PATH" >> /opendevin/bash.bashrc\n'
    ).strip()
```

> Note: the name of the image is modified via [_get_new_image_name()](https://github.com/OpenDevin/OpenDevin/blob/main/opendevin/runtime/docker/image_agnostic_util.py#L63) and it is the modified name that is searched for on subsequent runs.

## Troubleshooting / Errors

### Error: ```useradd: UID 1000 is not unique```
If you see this error in the console output it is because OpenDevin is trying to create the opendevin user in the sandbox with a UID of 1000, however this UID is already being used in the image (for some reason). To fix this change the sandbox_user_id field in the config.toml file to a different value:

```toml
[core]
workspace_base="./workspace"
persist_sandbox=false
run_as_devin=true
sandbox_container_image="custom_image"
sandbox_user_id="1001"
```

### Port use errors

If you see an error about a port being in use or unavailable, try deleting all running Docker Containers (run `docker ps` and `docker rm` relevant containers) and then re-running ```make run```

## Discuss

For other issues or questions join the [Slack](https://join.slack.com/t/opendevin/shared_invite/zt-2jsrl32uf-fTeeFjNyNYxqSZt5NPY3fA) or [Discord](https://discord.gg/ESHStjSjD4) and ask!
