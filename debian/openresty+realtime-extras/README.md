## openresty+realtime-extras

#### What is this? What does it do?

It's a [Vagrant](http://vagrantup.com/)file and provisioning script which, when run, builds and packages [nginx-push-stream-module](https://github.com/wandenberg/nginx-push-stream-module) and [ngx_cache_purge](https://github.com/FRiCKLE/ngx_cache_purge) into an [OpenResty](https://openresty.org) installation. 

It spits out a [debian](http://debian.org/) `.deb` package into your Vagrant host's working directory, containing `openresty` and the extra bundled modules and a slightly modified `init` script to run it. It really wouldn't take much work to have it spit out `.rpm` packages too.

I've tested it on [Virtualbox](http://virtualbox.org/). The provisioning file will probably work largely unmodified in `debian` images on [AWS](http://aws.amazon.com/)/[GCE](http://cloud.google.com/compute/)/la-la-la, too.

#### Why does it do that?

- `OpenResty` is **cool**. It uses the the [nginx](http://nginx.org/) [HttpLuaModule](http://wiki.nginx.org/HttpLuaModule) ("`ngx_lua`") to embed [LuaJIT](http://luajit.org/) (a fast, just-in-time-compiling [Lua](http://www.lua.org/) runtime) into `nginx`, bundling up a bunch of useful `nginx` extensions and exposing swathes of the `nginx` API to your Lua code. So you're scripting the `nginx` runtime, right down to request/response/header(etc) level, along with a bunch of added functionality (including async `mysql`/`redis`/`memcached` clients and more) - and you're doing it neat async code which doesn't have layers of painful callbacks, and which runs *really* fast. Meaning, you have a fully-fledged asynchronous application server running inside `nginx`. Basically, it does what `node.js` does, but with an "arguably" better language, and with full access to all the features - and the well-established configuration mechanisms & conventions - of an extremely powerful "proper" webserver. *(cough)*

- The `push-stream-module` is also **extremely** cool. It basically terminates `WebSocket` and `EventSource` connections for you, keeping the connections alive with extremely low memory overhead, and letting you send messages to individual or broadcast "channels" (or any combination of the two) just by sending simple `HTTP POST`s to a `publish` URL configurable in `nginx.conf`. It's used in a bunch of very high-traffic sites and pretty much just handles most of the headaches of real-time server-to-client web messaging, leaving you much freer to code and reason in a pretty standard web-app fashion.

- Running dynamic apps with real-time messaging is pretty likely to need some content caching. Combining `ngx_lua` and the `push-stream-module` with `ngx_cache_purge` means you can have your proxy/cache listen for cache-invalidation messages via lightweight `EventSource` or `WebSocket` streams, notify it when particular cached resources need re-fetching, and have it mark given resources stale with a simple HTTP `PURGE` to their specific URLs.

#### How do I use it?

Clone the repo:

```
git clone https://github.com/igorclark/vagrant-builds.git
```

Move into the working directory:

```
cd vagrant-builds/debian/openresty+realtime-extras
```

Start the VM:

```
vagrant up
```

#### How does it work?

It boots up and runs `bootstrap-scripts/install-openresty+realtime-extras` as a provisioning file. This does the following:

- updates the OS and installs a bunch of utilities & prerequisites via `apt-get`

- installs the debian package `nginx-common`, to take advantage of all the extremely nice config & packaging the debian folks have done for their `nginx` packages

- pulls down the latest tagged versions of `ngx_openresty`, `nginx-push-stream-module` and `ngx_cache_purge`

- hacks the `nginx-push-stream-module` and `ngx_cache_purge` code into `ngx_openresty`'s configure/build scripts (using `sed`, mmm brittle)

- runs `make ; make install` in the `ngx_openresty` distribution directory

- symlinks the `/usr/share/nginx/nginx/sbin/nginx` binary at `/usr/sbin/nginx`, so the debian `nginx` boot script `/etc/init.d/nginx` can point to it without any modification
 
- packages up the `openresty` installation (in `/usr/share/nginx`) and the boot script (`/etc/init.d/nginx`) into a debian `.deb` file using [FPM](https://github.com/jordansissel/fpm)

- drops the `.deb` file into the `/vagrant` share on the VM so that it shows up in your Vagrant working directory

That's it. You now have a `.deb` file which you can install in `Virtualbox` VMs to your heart's content.



#### Are there any known issues with it?

It works pretty well, but it's not exactly, uh, elegant. Also, it relies on naming conventions in the `openresty` project as well as version schemes for `openresty`, `nginx-push-stream-module` and `ngx_cache_purge`, so if these change it'll break. 

The approach I take with these things is just to `set -euo pipefail` in all `bash` scripts, so that any failure in any part of any command makes the whole thing crap out. Which forces me to keep going back and tweaking it to make every step work before the whole thing will run. (And, you know, fix it every time something changes upstream.) Lovely.
