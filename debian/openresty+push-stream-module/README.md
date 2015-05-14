## openresty+push-stream-module

#### What is this? What does it do?

It's a [Vagrant](http://vagrantup.com/)file and provisioning script which, when run, builds and packages [nginx-push-stream-module](https://github.com/wandenberg/nginx-push-stream-module) into an [OpenResty](https://openresty.org) installation. 

It spits out a [debian](http://debian.org/) `.deb` package into your Vagrant host's working directory, containing `openresty-nginx+push-stream`, and a slightly modified `init` script to run it. It really wouldn't take much work to have it spit out `.rpm` packages too.

I've tested it on [Virtualbox](http://virtualbox.org/). The provisioning file will probably work largely unmodified in `debian` images on [AWS](http://aws.amazon.com/)/[GCE](http://cloud.google.com/compute/)/etc, too.

#### Why does it do that?

- `OpenResty` is **cool**. It embeds [LuaJIT](http://luajit.org/) (a fast, just-in-time-compiling [Lua](http://www.lua.org/) runtime) into [nginx](http://nginx.org/), bundling up a bunch of useful `nginx` extensions and HTTP exposing swathes of `nginx` API to your Lua code. So you're basically scripting the `nginx` runtime, right down to request/response/header(etc) level, along with a bunch of added functionality (including async `mysql`/`redis`/`memcached` clients and more) - and you're doing it neat async code which doesn't have layers of painful callbacks, and which runs *really* fast. Meaning, you have a fully-fledged asynchronous application server running inside `nginx`. Basically, it does what `node.js` does, but with an "arguably" better language, and with full access to all the features and the proper configuration mechanisms & conventions of a proper webserver. *(cough)*

- The `push-stream-module` is also **extremely** cool. It basically terminates `WebSocket` and `EventSource` connections for you, keeping the connections alive with extremely low memory overhead, and letting you send messages to individual or broadcast "channels" (or any combination of the two) just by sending simple `HTTP POST`s to a `publish` URL configurable in `nginx.conf`. It's used in a bunch of very high-traffic sites and pretty much just handles most of the headaches of real-time server-to-client web messaging, leaving you code and reason in a pretty standard web-app fashion.


#### How do I use it?

Clone the repo:

```
git clone https://github.com/igorclark/vagrant-builds.git
```

Move into the working directory:

```
cd vagrant-builds/debian/openresty+push-stream-module
```

Start the VM:

```
vagrant up
```

#### How does it work?

It boots up and runs `bootstrap-scripts/ install-openresty+push-stream` as a provisioning file. This does the following:

- updates the OS and installs a bunch of utilities & prerequisites via `apt-get`

- installs the debian package `nginx-common`, to take advantage of all the extremely nice config & pacakaging the debian folks have done for their `nginx` packages

- pulls down the latest tagged versions of `ngx_openresty` and `nginx-push-stream-module`

- hacks the `nginx-push-stream-module` into `ngx_openresty`'s configure/build scripts (using `sed`, mmm brittle)

- runs `make ; make install` in the `ngx_openresty` distribution directory

- updates the debian `nginx` boot script `/etc/init.d/nginx` to point the `DAEMON` variable at the new `openresty` `nginx` binary in `/usr/share/nginx/nginx/sbin`

- packages up the `openresty` installation (in `/usr/share/nginx`) and the boot script (`/etc/init.d/nginx`) into a debian `.deb` file using [FPM](https://github.com/jordansissel/fpm)

- drops the `.deb` file into the `/vagrant` share on the VM so that it shows up in your Vagrant working directory

#### Are there any known issues with it?

It works pretty well, but it's not exactly, uh, elegant. The approach I take with these things is just to `set -euo pipefail` in all `bash` scripts so that any failure in any part of any command makes the whole thing crap out, forcing me to keep going back and tweaking it to make every step work before the whole thing will run. And, you know, fix it every time something changes upstream. Lovely.