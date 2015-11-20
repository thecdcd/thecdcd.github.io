---
layout:     post
title:      Put a Python Script in a Container
date:       2015-11-18
summary:    How to containerize a Python script and distribute the docker image.
categories: docker python
---
This post covers putting a [Python](https://www.python.org/) [script](https://github.com/thecdcd/twitch-count) into a [docker container](https://hub.docker.com/) and distributing it across a [Mesos cluster](http://mesos.apache.org/). To skip to the end, [here are the necessary files and steps](https://github.com/thecdcd/twitch-count-docker).

<!-- more -->

# Requirements
* 1 docker host capable of pulling images from [docker hub](https://hub.docker.com/)
* (optional) 1 Mesos cluster with docker executors

# The Script
The script below makes an HTTP request to Twitch TV's stream API to extract the total number of live streams.

Save the content below as `twitchcounts.py`.

{% highlight python linenos %}
import requests

# Twitch TV 'streams' end point.
BASE_URL = 'https://api.twitch.tv/kraken/streams'

def crawl(url, params):
    r = requests.get(url, params=params, timeout=30)

    if r.status_code == 200:
        js = r.json()
        if '_total' in js:
            print("There are {0} streamers doing it live.".format(js['_total']))


def main():
    query_params = {
        'limit': 1,
        'stream_type': 'live'
    }

    crawl(BASE_URL, query_params)


if __name__ == '__main__':
    main()
{% endhighlight %}

# Containerize It
In order for docker to create an image, it needs a `Dockerfile`. A `Dockerfile` [contains a series of commands](http://docs.docker.com/engine/reference/builder/) for docker to execute in order to assemble an image.

Save the contents below as `Dockerfile` in the same directory as `twitchcounts.py`.

{% highlight text linenos %}
FROM python:3-onbuild

## Run the python script
CMD [ "python", "./twitchcounts.py" ]
{% endhighlight %}

## An Aside
The `Dockerfile` above uses `python:3-onbuild` as the base (starting) image, which is itself a `Dockerfile`!

{% highlight text linenos %}
FROM python:3.5

RUN mkdir -p /usr/src/app
WORKDIR /usr/src/app

ONBUILD COPY requirements.txt /usr/src/app/
ONBUILD RUN pip install --no-cache-dir -r requirements.txt

ONBUILD COPY . /usr/src/app
{% endhighlight %}

# Python Dependencies
The script begins with `import requests`, however, this library (`requests`) is not part of the base python installation. Executing the script as-is will result in an `ImportError`. The `python:3-onbuild` image will execute `pip install -r requirements.txt` and any dependencies listed inside the file will be installed into the  image.

Save the line below as `requirements.txt` in the same directory as `twitchcounts.py`.

{% highlight text linenos %}
requests==2.8.1
{% endhighlight %}

# Build It
Use the `build` command to build the new image. The tag argument (`-t`) allows naming the resulting image -- without it the image can only be identified through a hash.

Run "`docker build -t twitch-channels-app .`" -- the "`.`" references the local directory with `twitchcounts.py`.

{% highlight text linenos %}
$ sudo docker build -t twitch-channels-app .
Sending build context to Docker daemon 70.14 kB
Sending build context to Docker daemon
Step 0 : FROM python:3-onbuild
# Executing 3 build triggers
Trigger 0, COPY requirements.txt /usr/src/app/
Step 0 : RUN pip install --no-cache-dir -r requirements.txt
 ---> Running in 947f5ed0406d
Collecting requests==2.8.1 (from -r requirements.txt (line 1))
  Downloading requests-2.8.1-py2.py3-none-any.whl (497kB)
Installing collected packages: requests
Successfully installed requests-2.8.1
Trigger 2, COPY . /usr/src/app
Step 0 : COPY . /usr/src/app
 ---> 0333a991676d
Removing intermediate container e43e6fa4fcb9
Removing intermediate container 947f5ed0406d
Removing intermediate container 3f3747e2669d
Step 1 : CMD python ./twitchcounts.py
 ---> Running in 9e36355a9d84
 ---> 517db7e0fb8a
Removing intermediate container 9e36355a9d84
Successfully built 517db7e0fb8a
{% endhighlight %}

The `images` command will show docker images on the local system. `twitch-count-app` should be part of the listing.

Run `docker images` to view docker images on the current machine.

{% highlight text linenos %}
twitch-count$ sudo docker images
REPOSITORY            TAG                 IMAGE ID            CREATED             VIRTUAL SIZE
twitch-count-app      latest              517db7e0fb8a        31 seconds ago      695 MB
{% endhighlight %}

# Run It
Use the `run` command to run the new image and get the streamer count.

Run "`docker run --rm --name my-running-app twitch-count-app`" to run the docker container.

{% highlight text linenos %}
$ sudo docker run --rm --name my-running-app twitch-count-app
There are 32770 streamers doing it live.
{% endhighlight %}

The arguments are:

* `--rm` automatically removes the container when it exits; this prevents build-up of results when running `docker ps -a`
* `--name my-running-app` assigns a name to the container and prevents more than one from running at a time (with the same name)
* `twitch-count-app` is the image to execute, which was the tag (`-t`) used in the `build` command

# Distribute It
The image is built, but only available on the host that did the building. There are two options:

1. Build the image on every docker host
1. Save the image and load it on every docker host

Since the first option was demonstrated above, this tackles saving and loading docker images.

## Save
docker provides the `save` command, which outputs a docker image as a `tar` file. By default, it writes to `STDOUT`, but this can be changed with the output (`-o`) argument.

**NOTE:** The command below uses `sudo`, so the resulting `tar` file will be owned by `root`. Be sure to `chown` the file correctly for your setup.

Run "`docker save -o twitch-count-app.tar twitch-count-app`" to save the `twitch-count-app` image to a tar file.

{% highlight text linenos %}
$ sudo docker save -o twitch-count-app.tar twitch-count-app
$ ls -l
-rw-r--r-- 1 root root 716619264 Nov  7 11:58 twitch-count-app.tar
$ sudo chown $(id -un):$(id -gn) twitch-count-app.tar
$ ls -l
-rw-r--r-- 1 cdcd cdcd 716619264 Nov  7 11:58 twitch-count-app.tar
{% endhighlight %}

## Load
docker provides the `load` command to ingest `tar` files that were created with the `save` command. Transfer the file from the section above to a target host and run the `load` command.

Similar to `save`, use the input (`-i`) argument to read from a file instead of `STDIN`.

{% highlight text linenos %}
$ sudo docker load -i twitch-count-app.tar
$ sudo docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             VIRTUAL SIZE
twitch-count-app    latest              517db7e0fb8a        18 minutes ago      695 MB
{% endhighlight %}

# Conclusion
This post walked through creating a docker image specifically for a Python script and how to transfer that image to other docker hosts. With these basics, you can containerize Python applications and use them within a Mesos cluster.

The streamer counting script on Chronos:
![Chronos Task](/images/containerize-python-chronos.png)

A simple web service on Marathon:
![Marathon Service](/images/containerize-python-marathon.png)
