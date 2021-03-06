# Buildslaves

Starting recently we've been trying to ensure that all of our build slaves are
defined via a docker image instead of just using "some random AMI" and then
randomly pulling that AMI forward from time to time. This has some nice concrete
benefits:

* It's easy to review changes to build slaves
* The build slaves can be reproduced locally
* Knowing what's actually on a build slave is much easier

Currently the buildslave daemon is run inside of a docker container, but
eventually it would also be nice to run each **build** inside of its own docker
container (to ensure isolation). This is not currently set up, however.

## Image architecture

Each image does a few simple tasks:

1. Installs the buildbot buildslave
2. Sets up the `rustbuild` user
3. Installs build dependencies
4. Preps the `buildslave` command to run

Whenever an image boots it will first clone this repo and then run the
`setup-slave.sh` script at the top of the repo. These actions are controlled via
the entry script of the image, `start-docker-slave.sh`. One notable part of this
script is that the URL and branch of this repo to clone are passed in via "User
Data" on the AMI booted.

## Building a docker image

In the directory of this readme, run:

```
docker build -f linux/Dockerfile .
```

If you want to tag it you can also pass the `-t` flag with the image name.

## Publishing a new docker image

Run this inside this directory:

```
docker build -t alexcrichton/rust-slave-linux:2015-10-15 -f linux/Dockerfile .
```

(note that today's date should be used in the tag name)

```
docker push alexcrichton/rust-slave-linux:2015-10-15
```

## Debugging a docker image in prod

1. Obtain the IP of the slave from the AWS console
2. SSH into the VM, currently with the username `ec2-user`
3. Run `docker ps` and look under `NAMES` to find the name of the currently
   running container
4. Run `docker exec -it <name> bash`

That'll give you a shell into the container so you can poke around and do
whatever you like. Note, though, that if the buildslave daemon is killed it will
likely kill the container and need to be re-run from the external command line.

## Recreating the "base AMI"

All our build slaves run inside of a bare-bones AMI. It should be recreatable as
follows:

1. Start up a fresh AMI.
2. Install docker (go to docker's home page to see how)
3. Make docker runnable without `sudo` (the install normally says how)
4. Add the following to `crontab`:

    ```
    @reboot sh -c 'sleep 20 && docker run --privileged `curl -s http://169.254.169.254/latest/user-data | tail -n +2 | head -n 1`' 2>&1 | logger
    ```

   To break this down:

   * `@reboot` - this command is run whenever the AMI is booted
   * `sh -c '...' 2>&1 | logger` - run a shell command, piping all of its output
     into the `logger` command (e.g. syslog). On the current images this makes
     it appear at `/var/log/messages`
   * `sleep 20` - wait for the docker daemon to start
   * `docker run ...` - run a docker image
   * `--privileged` - needed for gdb tests to work (enables `ptrace` I believe)
   * `curl ... | tail | head` - the name of the docker image to run is in the
     "User Data" of the AMI when buildbot boots it, and this is what fetches it
     and parses it out.

   Note that this means that whenever the AMI boots the first thing it will do
   is download a likely multi-gigabyte image from the Docker hub to run locally,
   but hey that's not so bad!
