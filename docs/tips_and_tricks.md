# Remove unwanted files

Especially during development, it can happen that files get generated when running
your docker container that can only be removed by the `root` user again. If you 
do not have admin permissions (e.g., via `sudo`) on the machine then you can 
revert to using docker to remove them again:

* change into the directory with unwanted files and/or directories

* the following docker command will map the current directory to `/workspace`:

```
docker run --rm -v `pwd`:/workspace -it bash:5.2.32
```
  
* change into `/workspace` and remove all files/dirs (or just the files that need removing):

```
cd /workspace
rm -Rf *
```

# Python

## Alternative Python version

In case your base Debian/Ubuntu distribution does not have the right Python version 
available, you can get additional versions by adding the [deadsnakes ppa](https://launchpad.net/~deadsnakes/+archive/ubuntu/ppa):

```bash
add-apt-repository -y ppa:deadsnakes/ppa && \
apt-get update
```

**Notes:** 

* You may need to install package `software-properties-common` before you have `add-apt-repository` available.
* Consider installed `python3.x-full` and `libpython3.x` to also get the `venv` and `distutils` packages installed.
* For installing `pip`, use this:

```
wget https://bootstrap.pypa.io/get-pip.py -O /tmp/get-pip.py && \
python3.7 /tmp/get-pip.py && \
rm /tmp/get-pip.py
```


## Switch default Python version

If your base distro has an older version of Python and you do not want to sprinkle the
specific newer version throughout your `Dockerfile`, then you can use `update-alternatives`
on Debian systems to change the default.

The following commands switch from Python 3.5 to Python 3.6 for the `python3` executable
and also use Python 3.6 as the default for the `python` executable:

```commandline
update-alternatives --install /usr/bin/python python /usr/bin/python3.5 1 && \
update-alternatives --install /usr/bin/python python /usr/bin/python3.6 2 && \
update-alternatives --set python /usr/bin/python3.6 && \
update-alternatives --install /usr/bin/python3 python3 /usr/bin/python3.5 1 && \
update-alternatives --install /usr/bin/python3 python3 /usr/bin/python3.6 2 && \
update-alternatives --set python3 /usr/bin/python3.6 && \
```

## Install packages with pip ignoring errors

An installation with `pip` will fail if a dependency is not available, even if that dependency should not be required.
The following command-line install each dependency separately, therefore continuing, even if errors are encountered ([source](https://stackoverflow.com/a/54053100)):

```
cat requirements.txt | sed -e '/^\s*#.*$/d' -e '/^\s*$/d' | xargs -n 1 pip install
```


# Screen

The [screen](https://linux.die.net/man/1/screen) command-line utility allows you to 
pick up remote sessions after detaching from them. This is especially helpful when 
running docker in interactive mode via ssh sessions. If such an ssh session should
accidentally close (e.g., internet connection lost, laptop closed), then the
docker command would terminate as well.

On the remote host, **start** the `screen` command before you launch your docker command 
(you can run multiple `screen` instances as well) and skip the screen with the licenses 
etc using `Enter` or `Space`. Now you can start up the actual command. If you want to 
run multiple screen sessions on the same host, then you should name them using the 
`-S sessionname` option to easily distinguish them. 

When running multiple sessions on a single host that use the auto-generated names, 
it can get difficult to distinguish them. You can use 
`screen -S <PID>.<sessionName> -X sessionname <newSessionName>` to rename an 
existing session to use a more suitable name ([source](https://www.shellhacks.com/screen-rename-session/)).

For **detaching** a session, use the `CTRL+A+D` key combination. This will leave your 
process running in the background and you can close the remote connection.

In order to **reconnect**, simply ssh into the remote host and run `screen -r`. If there
is only one screen session running, then it will automatically reconnect. Otherwise, it
will output a list of available sessions. Supply the name of the session that you want to 
reattach to the `-r` option. In case a session got interrupted and not properly detached,
use the `-d` flag to first detach it.

You can **exit** a screen session by either typing `exit` or using `CTRL+D` (just like
with a `bash` shell).

If you need to increase the scrollback buffer, let us say to 100,000 lines, then you can do it

* in the current session as follows:

```
CTRL A : <Enter>
scrollback 100000<Enter>
```

* for all new sessions by adding the following line in your `$HOME/.screenrc` config file:

```
defscrollback 100000
```


# Default runtime

Instead of always having to type `--runtime=nvidia` or `--gpus=all`, you can simply define
the *default runtime* to use with your docker commands ([source](https://docs.nvidia.com/dgx/nvidia-container-runtime-upgrade/index.html)):

* edit the `/etc/docker/daemon.json` file as `root` user
* insert the key `default-runtime` with value `nvidia` so that it looks something like this:
  
```json
{
    "default-runtime": "nvidia",
    "runtimes": {
        "nvidia": {
            "path": "nvidia-container-runtime",
            "runtimeArgs": []
        }
    }
}
```

* save the file
* restart the docker daemon (e.g., `sudo systemctl restart docker`)

# Rebuild

## Rebuilding some layers

Since docker hashes the layer instructions rather than the content, it will not detect changes to a file 
(e.g., modifications in a script) since the last build and will keep using the cached version of that layer.

Assuming the following `Dockerfile` snippet, where `file1` changed:

```
RUN ...<layer X>...
COPY file1 ...<layer X+1>...
COPY file2 ...<layer X+2>...
```

You can **inject** a dummy **ARG** variable as follows to rebuild the layers from `layer X+1` onwards:

```
RUN ...<layer X>...
ARG blah=1
COPY file1 ...<layer X+1>...
COPY file2 ...<layer X+2>...
```

Every time you need rebuild from there on, simply change the value that you are assigning to the variable, 
to change the hash value.

Of course, once your `Dockerfile` has been finalized, you can remove all these unnecessary operations and
build your image properly.


## Complete rebuild

Especially during development of a docker image, it can happen that library versions have not been added to the `Dockerfile` just yet. Since docker hashes the command-lines, it will not rebuild a layer (and subsequent ones) if the hash does not change. If you do not want to clear your complete system (due to time constraints or capped internet usage), then you can force docker to build the image without using the hashed layers by adding the following flag to your [build](https://docs.docker.com/engine/reference/commandline/build/) command:

```
--no-cache
```


# Thorough clean up

You can stop/remove all containers and images with this one-liner:

```bash
docker stop $(docker ps -a -q) && docker rm $(docker ps -a -q) && docker system prune -a
```

If you need to remove dangling volumes, use this:

```commandline
docker volume prune
```

# Excluding files/dirs from docker context

By default, docker will use all files and directories within the directory that contains the `Dockerfile` to
the daemon to use as *context*. From this context, the image gets built. Should you have test data/models
in the same directory then this could mean sending quite a lot of unnecessary data to the daemon. To avoid this,
you can take advantage of the `.dockerignore` file, which lists files and directories to ignore. The following
example ignores the `test` directory and all vim backup files:

```
test/
*~
```

For more information, see the official documentation on the [dockerignore file](https://docs.docker.com/engine/reference/builder/#dockerignore-file).


# Testing the GPU

Images from the [gpu-test](https://github.com/waikato-datamining/gpu-test) repository
can be used to quickly test whether Docker containers can access the GPU properly.

The following command, based on CUDA 12.2.2, should simply run `nvidia-smi` within
the container and output some information on the available GPUs (RAM, load, etc):

```
docker run --rm \
  --gpus=all \
  -it waikatodatamining/gpu-test:cuda12.2.2
```

If the GPU is not passed through correctly (e.g., due to mismatching drivers, etc), 
an error message will get output instead. Here is an example of such an error:

```
docker: Error response from daemon: could not select device driver "" with capabilities: [[gpu]].
```


# Environment variables

You can provide environment variables via a [.env file](https://hexdocs.pm/dotenvy/0.2.0/dotenv-file-format.html)
to your Docker container without having to provide them as parameters. That approach
avoids having variables like access tokens appear in your command history, leaking
sensitive information.

A `.env` file is basically a file with one `key=value` pair per line, with the
`key` being the name of the environment variable and `value` the associated
value of the variable.

If you have such a `.env` in the directory that you are starting the container from,
you can make them available with the following option of the `docker run` command:

```
--env-file=`pwd`/.env
```
