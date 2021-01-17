----
author: Mark Boyer
----

# IDEs in Containers

Containers, as managed by [Docker](https://www.docker.com/), [Singularity](https://sylabs.io/docs/), 
or [Shifter](https://docs.nersc.gov/development/shifter/how-to-use/) can dramatically simplify 
the process of porting code from one system to another. 
Conceptually, by being a minimal managed operating system that can run efficiently (enough) anywhere, 
many of the circles of [dependency hell](https://en.wikipedia.org/wiki/Dependency_hell) can be avoided.

This is particularly useful when running on HPC systems. 
Generally, one has to use a [`module` system](https://hpc-wiki.info/hpc/Modules) to manage dependencies.
One issue with this is that the set of modules on [Mox](https://wiki.cac.washington.edu/display/hyakusers/Hyak+mox+Overview) can differ from 
the set of module of [Cori](https://docs.nersc.gov/systems/cori/) and when [upgrades happen](https://www.nersc.gov/systems/perlmutter/) 
there are no assurances that everything will remain the same.
Moreover, the set of modules might be insufficient for what you want to do. 
One can try to get around this by using [conda environments](https://docs.conda.io/projects/conda/en/latest/user-guide/tasks/manage-environments.html),
but this still requires synchronization across platforms, and, not to put too fine a point on it,
once you're managing that kind of environment, you're half-way
to the container world anyway, but without any promises of portability.

Maybe all of that's convincing; maybe it's not. Regardless, we're going to present an example of how
containers can make life nice, specifically, getting [VS Code](https://github.com/cdr/code-server) to run on an HPC node with basically
0 effort using `singularity`.

## Basic Setup

Happily, initial configuration is trivial.
First, get onto your HPC. 
I use a [persistent connection](https://www.cyberciti.biz/faq/linux-unix-reuse-openssh-connection/) to avoid extra log ons. 
Especially in [2FA](https://authy.com/what-is-2fa/) world, this is nice.

```console
Mark$ MCDEFAULT_CONNECTION=~/.ssh/b3m2a1\@mox.hyak.uw.edu 
Mark$ mcssh -h hyak
Last login: Thu Jan 14 18:19:10 2021
         __  __  _____  __  _  ___   ___   _  __
        |  \/  |/ _ \ \/ / | || \ \ / /_\ | |/ /
        | |\/| | (_) >  <  | __ |\ V / _ \| ' < 
        |_|  |_|\___/_/\_\ |_||_| |_/_/ \_\_|\_\

    This login node is meant for interacting with the job scheduler and 
    transferring data to and from Hyak. Please work by requesting an 
    interactive session on (or submitting batch jobs to) compute nodes.

    Visit the Hyak user wiki for more details:
    http://wiki.hyak.uw.edu/Hyak+mox+Overview

    Questions? E-mail help@uw.edu with "hyak" in the subject.

    Run "scontrol show res" to see any reservations in place that will 
    prevent your jobs from running with a "(ReqNodeNotAvail,*" error.

[b3m2a1@mox2 mccoygrp]$ 
```

Next, get a build node/interactive node/whatever your local HPC calls it.

```console
[b3m2a1@mox2 mccoygrp]$ build_node
srun: job 790748 queued and waiting for resources
srun: job 790748 has been allocated resources
[b3m2a1@n2233 mccoygrp]$ 
```

Now, we `module load singularity` to get it exposed (this is the only time we'll use the `module` system).
Then we just [`singularity build`](https://sylabs.io/guides/3.5/user-guide/build_a_container.html) 
the container from the [DockerHub repo that the developers provided](https://hub.docker.com/r/codercom/code-server)

```console
[b3m2a1@n2233 mccoygrp]$ singularity build code-server.sif docker://codercom/code-server:latest
INFO:    Starting build...
Getting image source signatures
Copying blob sha256:6c33745f49b41daad28b7b192c447938452ea4de9fe8c7cc3edf1433b1366946
 48.06 MiB / 48.06 MiB [====================================================] 0s
Copying blob sha256:164e2f04ea590ef419079d0a4d13e62ba04c48c0a269e270ac2d5d76b2e42ac6
 63.87 MiB / 63.87 MiB [====================================================] 1s
Copying blob sha256:2b9a6c0892dbfd0f9f3ab0d56b8ed20ac9abec558b22276b2c26036f4a773a7b
 842.51 KiB / 842.51 KiB [==================================================] 0s
Copying blob sha256:425135d0a4ab3f2b55e91cecf6834fbf5d10ae83165d86f31b6cff7c65c522f9
 4.12 KiB / 4.12 KiB [======================================================] 0s
Copying blob sha256:0ebec939099795da919a35a045bea5649dfc73d388d993664b35d05eb0a8e7ba
 2.20 MiB / 2.20 MiB [======================================================] 0s
Copying blob sha256:d581310c026e8471f757610673455c3255113f6e98d9340e1a72a50e0c89a5a3
...
INFO:    Creating SIF file...
INFO:    Build complete: code-server.sif
```

at this point we're 80% of the way there!

Now we need to get the server going and [exposing a port](http://www.linuxandubuntu.com/home/what-are-ports-how-to-find-open-ports-in-linux) 
so we can connect a browser to it.
That just means we need to `export` appropriate a `PORT` variable. This is up to you. I like `8666`

```console
[b3m2a1@n2233 mccoygrp]$ export PORT=8666
[b3m2a1@n2233 mccoygrp]$ singularity run /gscratch/ilahie/mccoygrp/VSCode/vs-server.sif
[2021-01-16T20:29:56.086Z] info  code-server 3.7.4 11f53784c58f68e7f4c5b3b8dae9407caa41725b
[2021-01-16T20:29:56.090Z] info  Using user-data-dir ~/.local/share/code-server
[2021-01-16T20:29:56.105Z] info  Using config file ~/.config/code-server/config.yaml
[2021-01-16T20:29:56.106Z] info  HTTP server listening on http://0.0.0.0:8666 
[2021-01-16T20:29:56.106Z] info    - Authentication is enabled
[2021-01-16T20:29:56.106Z] info      - Using password from ~/.config/code-server/config.yaml
[2021-01-16T20:29:56.106Z] info    - Not serving HTTPS
```

You'll note that the output said `Using password from ~/.config/code-server/config.yaml`.
You'll need to `cat` that file to get the password the first time you log on.

Now, midly annoyingly, we need to do two layers of [port forwarding]() to connect to the server that `VSCode` is running.
First, from our local machine we set up port forwarding to the HPC. I reuse my persistent connection. 
Then from a login node on our HPC we connect to the build node (or compute node, there are no restrictions beyond `singularity` running)
using the same port

```console
Mark$ mcportforward 8666 -h hyak
Mark$ mcssh -h hyak
[b3m2a1@mox2 mccoygrp]$ uqueue
    JOBID PARTITI                             NAME     USER ST       TIME  CPUS  NODES NODELIST(REASON)
   790660   build                             bash   b3m2a1  R    3:32:04     1      1 n2233
[b3m2a1@mox2 mccoygrp]$ VSCODE_PORT=8666
[b3m2a1@mox2 mccoygrp]$ VSCODE_NODENAME=n2233
[b3m2a1@mox2 mccoygrp]$ ssh -N -f -L 127.0.0.1:$VSCODE_PORT:127.0.0.1:$VSCODE_PORT VSCODE_NODENAME
```

Then by going to `http://localhost:8666` in whatever browser we like, we have a working VSCode instance!

![vs-code-screenshot](img/vs-code-screenshot.png)

### Simplifying the Workflow



### Connecting to the Node Terminal


 ## Jupyter
