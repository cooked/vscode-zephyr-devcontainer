# VSCode devcontainer for Zephyr RTOS

This is a template workspace for developing Zephyr RTOS application in a 
[VS Code development container](https://code.visualstudio.com/docs/devcontainers/containers). 

## Requirements

Make sure the prerequisites are satisfied, as described in the [Dev Containers tutorial](https://code.visualstudio.com/docs/devcontainers/tutorial). In particular, that [Docker](https://www.docker.com/products/docker-desktop/) is up and running and the [Dev Containers extension](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-containers) is installed. 

## Getting started

```bash
git clone https://github.com/cooked/vscode-zephyr-devcontainer.git
```

After cloning the repo adjust the devcontainer parameters according to your 
needs (for example, to mount folders that are outside of the project location, 
otherwise only the workspace content is available in the container).

```yml
// eg. mount "foo" folder from your home folder, to the containerized workspace 
"mounts": [
    "source={env:HOME}/foo,target=${containerWorkspaceFolder}/foo,type=bind,consistency=cached",
],
```

Finally, launch the container from inside VSCode with **CTRL+SHIFT+P** then 
**Dev Containers: Reopen in container**.

## Tasks

Few basic [tasks](https://github.com/cooked/vscode-zephyr-devcontainer/blob/master/.vscode/tasks.json) can be accessed with **CTRL+SHIFT+B** (to test everything is working try to run **Zephyr sample build > blinky**).

## Limitations

To keep the containers small, only the required toolchain for a 
specific target/manufacturer is installed.

Currently, only **stm32** is available as a package out of the box.

Additionally, the only tested host is **Linux**.

## Developers

The *devel* branch contains a modified *devcontainer.json* to build from the 
local Dockerfile, useful for development (i.e. the "image" key is replaced by 
the following snippet where fine tuning the build args is possible).

```yml
"dockerFile": "Dockerfile",
"context": "../",
"build": {
    "args": {
        "TARGET" : "stm32",
        "TOOLCHAIN" : "arm-zephyr-eabi",
        "ZSDK_VERSION" : "0.16.8",
        "WEST_MANIFEST" : ".manifests/west-stm32.yml"
    }
}
```
