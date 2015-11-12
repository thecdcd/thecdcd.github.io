---
layout:     post
title:      Cross Compiling an Erlang Application
date:       2015-08-22
summary:    A short guide to cross compiling an Erlang application with rebar3.
categories: erlang freebsd rebar
---
I am developing an [Erlang](http://erlang.org) application on my laptop for a production system that runs [FreeBSD 10.x](http://www.freebsd.org). By default, the compiled releases are for OSX/darwin and do not run on the production target. This post details how to cross compile an Erlang project for FreeBSD, however, it should be applicable to any operating system.

<!-- more -->

This short guide begins by creating a new project with [rebar3](http://www.rebar3.org/) and ends with it running on both a development machine and a [FreeBSD](http://www.freebsd.org) server. This picks up after the [Getting Started guide](http://www.rebar3.org/v3.0/docs) for the `rebar3` tool and covers project creation, build configuration, and distribution packaging.

## Getting Started
The command `rebar3` should be available and return output similar to below:

{% highlight text linenos %}
$ rebar3
Rebar3 is a tool for working with Erlang projects.


Usage: rebar [-h] [-v] [<task>]

  -h, --help     Print this help.
  -v, --version  Show version information.
  <task>         Task to run.
[ snipped ]
{% endhighlight %}

First, create a new project called `myapp`; change directory (`cd`) to the new directory `myapp` afterwards.

{% highlight text linenos %}
$ rebar3 new release myapp
===> Writing myapp/apps/myapp/src/myapp_app.erl
===> Writing myapp/apps/myapp/src/myapp_sup.erl
===> Writing myapp/apps/myapp/src/myapp.app.src
===> Writing myapp/rebar.config
===> Writing myapp/config/sys.config
===> Writing myapp/config/vm.args
===> Writing myapp/.gitignore
===> Writing myapp/LICENSE
===> Writing myapp/README.md
$ cd myapp
tree
.
├── LICENSE
├── README.md
├── apps
│   └── myapp
│       └── src
│           ├── myapp.app.src
│           ├── myapp_app.erl
│           └── myapp_sup.erl
├── config
│   ├── sys.config
│   └── vm.args
└── rebar.config

4 directories, 8 files
{% endhighlight %}

Next, open `apps/myapp/src/myapp_app.erl` and add a short debug statement to the `start` function. It should look like the function below.

{% highlight erlang linenos %}
start(_StartType, _StartArgs) ->
    io:format("myapp is starting up~n"),
    'myapp_sup':start_link().
{% endhighlight %}

Finally, start the application with `rebar3 shell` and confirm the above line is displayed.

{% highlight text linenos %}
$ rebar3 shell
===> Verifying dependencies...
===> Compiling myapp
Erlang/OTP 18 [erts-7.0.3] [source] [64-bit] [smp:4:4] [async-threads:0] [hipe] [kernel-poll:false] [dtrace]

Eshell V7.0.3  (abort with ^G)
1> ===> The rebar3 shell is a development tool; to deploy applications in production, consider using releases (http://www.rebar3.org/v3.0/docs/releases)
myapp is starting up
===> Booted myapp
===> Booted sasl
[ snipped ]
1> q().
ok
$
{% endhighlight %}

The program starts and prints out the debug line; success!

## Packaged Release

Now to package it for distribution. To do this, run the `tar` command with `rebar3 as prod tar`.

{% highlight text linenos %}
$ rebar3 as prod tar
===> Verifying dependencies...
===> Compiling myapp
===> Starting relx build process ...
===> Resolving OTP Applications from directories:
          myapp/_build/prod/lib
          myapp/apps
          /usr/local/Cellar/erlang/18.0.3/lib/erlang/lib
===> Resolved myapp-0.1.0
===> Including Erts from /usr/local/Cellar/erlang/18.0.3/lib/erlang
===> release successfully created!
===> Starting relx build process ...
===> Resolving OTP Applications from directories:
          myapp/_build/prod/lib
          myapp/apps
          /usr/local/Cellar/erlang/18.0.3/lib/erlang/lib
          myapp/_build/prod/rel
===> Resolved myapp-0.1.0
===> tarball myapp/_build/prod/rel/myapp/myapp-0.1.0.tar.gz successfully created!
{% endhighlight %}

Here is what is happening:

* `as` selects the profile `prod`, which is defined inside `rebar.config`. This profile was created by the `new` command executed initially.
* `tar` is a [command](https://www.rebar3.org/docs/commands#tar) that creates an erlang application release and packs it into a tar file.

### Local Machine

On the local development machine, extract the archive and run the release with `bin/myapp-0.1.0 foreground`.

{% highlight text linenos %}
$ mkdir -p ~/unzip
$ tar xf _build/prod/rel/myapp/myapp-0.1.0.tar.gz -C ~/unzip
$ cd ~/unzip
$ bin/myapp-0.1.0 foreground
Exec: erts-7.0.3/bin/erlexec -noshell -noinput +Bd -boot releases/0.1.0/start -mode embedded -config releases/0.1.0/sys.config -boot_var ERTS_LIB_DIR erts-7.0.3/../lib -args_file releases/0.1.0/vm.args -- foreground
Root: ~/unzip
myapp is starting up

[ snipped ]
^C
$
{% endhighlight %}

Success! (`CTRL+C` to exit)

### Remote Machine

Copy the archive to a remote system of a different operating system. The rest of this post assumes the remote system is [FreeBSD 10.x](https://www.freebsd.org).

{% highlight text linenos %}
$ scp _build/prod/rel/myapp/myapp-0.1.0.tar.gz remotesystem
myapp-0.1.0.tar.gz                                                                          23% 1424KB 303.8KB/s   00:15 ETA
$ ssh remotesystem
remotesystem> uname -a
FreeBSD remotesystem 10.1-RELEASE FreeBSD 10.1-RELEASE #0 r274401: Tue Nov 11 21:02:49 UTC 2014     root@releng1.nyi.freebsd.org:/usr/obj/usr/src/sys/GENERIC  amd64
remotesystem> mkdir myapp
remotesystem> tar xf myapp-0.1.0.tar.gz -C myapp/
remotesystem> cd myapp
remotesystem> bin/myapp-0.1.0 foreground
exec: myapp/erts-7.0.3/bin/erlexec: Exec format error
Exec: myapp/erts-7.0.3/bin/erlexec -noshell -noinput +Bd -boot myapp/releases/0.1.0/start -mode embedded -config myapp/releases/0.1.0/sys.config -boot_var ERTS_LIB_DIR myapp/erts-7.0.3/../lib -args_file myapp/releases/0.1.0/vm.args -- foreground
Root: myapp
exec: myapp/erts-7.0.3/bin/erlexec: Exec format error
remotesystem>
{% endhighlight %}

Not a success!

## Preparing to Cross Compile

According to [the documentation](https://www.rebar3.org/docs/releases#cross-compiling), a different Erlang distribution can be used by setting the `include_erts` and `system_libs` configuration settings in the target profile (`prod`).

### Getting the Distribution

The two ways I have gotten an Erlang distribution for another operating system are:

1. Install onto a system and copy the directories
2. Download and extract the correct package from a package manager

The follow steps cover the second option - downloading and extracting a binary distribution.

Binary packages for FreeSBD 10.x (amd64) [are available here](http://pkg.freebsd.org/freebsd:10:x86:64/latest/All/). As of  August 23, 2015, the version available is `erlang-18.0.3,3.txz` [(download)](http://pkg.freebsd.org/freebsd:10:x86:64/latest/All/erlang-18.0.3,3.txz). If this is out of date, visit the link above and search for `erlang-`.

{% highlight text linenos %}
$ wget http://pkg.freebsd.org/freebsd:10:x86:64/latest/All/erlang-18.0.3,3.txz
$ mkdir -p ~/unzip/erlang/freebsd
$ tar xf erlang-18.0.3,3.txz -C ~/unzip/erlang/freebsd
$ mv ~/unzip/erlang/freebsd/usr/local/* ~/unzip/erlang/freebsd
$ rm -rf ~/unzip/erlang/freebsd/usr/local/
{% endhighlight %}

Update `rebar.config` to use this alternative erlang and erts distribution.

{% highlight erlang linenos %}
{erl_opts, [debug_info]}.
{deps, []}.

{relx, [{release, {'myapp', "0.1.0"},
         ['myapp',
          sasl]},

        {sys_config, "./config/sys.config"},
        {vm_args, "./config/vm.args"},

        {dev_mode, true},
        {include_erts, false},

        {extended_start_script, true}]
}.

{profiles, [{prod, [{relx, [{dev_mode, false},
                            {include_erts, "/ABSOLUTE/PATH/unzip/erlang/freebsd/lib/erlang"},
                            {system_libs, "/ABSOLUTE/PATH/unzip/erlang/freebsd/lib/erlang"}]}]
            }]
}.
{% endhighlight %}

### Naming Nodes

Before the final build, there is one more change to make.

By default, `config/vm.args` will try to use a [long name](http://erlang.org/doc/reference_manual/distributed.html). A long name must follow the format `node@host` and be resolvable, otherwise the application will not start. A short name, however, is just `node`. Use a short name by changing `-name myapp` to `-sname myapp`.

{% highlight erlang linenos %}
-sname myapp

-setcookie myapp_cookie

+K true
+A30
{% endhighlight %}

### Creating the Archive

Create the final artifact by executing the `tar` command again.

{% highlight text linenos  %}
rebar3 as prod do clean, tar
===> Cleaning out myapp...
===> Verifying dependencies...
===> Compiling myapp
===> Starting relx build process ...
===> Resolving OTP Applications from directories:
          myapp/_build/prod/lib
          myapp/apps
          ~/unzip/erlang/freebsd/lib/erlang
          myapp/_build/prod/rel
===> Resolved myapp-0.1.0
===> Including Erts from ~/unzip/erlang/freebsd/lib/erlang
===> release successfully created!
===> Starting relx build process ...
===> Resolving OTP Applications from directories:
          myapp/_build/prod/lib
          myapp/apps
          ~/unzip/erlang/freebsd/lib/erlang
          myapp/_build/prod/rel
===> Resolved myapp-0.1.0
===> tarball myapp/_build/prod/rel/myapp/myapp-0.1.0.tar.gz successfully created!
{% endhighlight %}

* `as` selects the profile `prod`
* `do` allows executing several commands in a single call
* `clean` will clean the environment of previous compiled files
* `tar` creates a tar archive and executes required steps (compile)

## Finale

`scp` the newest archive to the remote system and run it!

{% highlight text linenos %}
$ bin/myapp foreground
Exec: myapp/erts-7.0.3/bin/erlexec -noshell -noinput +Bd -boot myapp/releases/0.1.0/start -mode embedded -config myapp/releases/0.1.0/sys.config -boot_var ERTS_LIB_DIR myapp/erts-7.0.3/../lib -args_file myapp/releases/0.1.0/vm.args -- foreground
Root: myapp
myapp is starting up

[ snipped ]
{% endhighlight %}

Success!
