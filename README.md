## Dream Deploys: Atomic, Zero-Downtime Deployments

Are you afraid to deploy? Do deployments always mean downtime, leaving your site in an inconsistent state for a window of time, or both? It doesn't have to be this way!

Let's conquer our fear. Let's deploy whenever we damn well feel like it.

<div style="text-align:center">
<img src="./img/donnie-darko-not-afraid-anymore.jpg" width="100%"/>
</div>

### You Don't Need Much

This is a tiny demo to convince you that Dream Deploys are not only possible, they're easy.

To live the dream, you don't need much:

- You don't need a fancy load balancer.
- You don't need magic "clustering" infrastructure.
- You don't need a specific language or framework.
- You don't need a queue system.
- You don't need a message bus or fancy IPC.
- *You don't even need multiple instances of your server running.*

All you need is a couple old-school Unix tricks.

## A Quick Demo

In a terminal, run this and visit the link:

    ./serve

In a second terminal, deploy whenever you want:

    ./deploy

Refresh the page to see it change.

Edit code, static files, or both under `./root.unused`. Then leave `./root.unused` and run `./deploy` to see your changes appear atomically and with zero downtime.

## Questions & Answers

#### What do you mean by a "zero downtime" deployment?

At no point is the site unavailable. Requests will continue to be served without transport layer errors before, during, and after the deployment. In other words, this is about **Availability**.

#### What do you mean by an "atomic" deployment?

For a given connection, either you will talk to the new code working against the new files, or you will talk to the old code working against the old files. You will never see a mix of old and new. In other words, this is about **Consistency**.

#### How does the zero downtime part work?

This brings us to Unix trick #1. If you keep the same listen socket open throughout the deployment, clients won't get `ECONNREFUSED` under normal circumstances. The kernel places them in a listen backlog until our server gets around to calling `accept(2)`.

This means, however, that our server process can't be the thing to call `listen(2)` if we want to stop and start it, or we'll incur visible downtime. Something else – some long running process – must call `listen(2)` and keep the listen socket open across deployments.

The trick in a nutshell, then, is this:

- A [tiny, dedicated program](./tcplisten) calls `listen(2)` and then passes the listen socket to child processes as descriptor 0 (stdin). This process replaces itself by executing a subordinate program.

- The subordinate program is [just a loop](./loop-forever) that repeatedly executes our server program. Because this loop program never exits, the listen socket on descriptor 0 stays open.

- Our server program, instead of calling `bind(2)` and `listen(2)` like everyone **loves to do**, humbly calls `accept(2)` on stdin in a loop and handles one client connection at a time.

- When it's time to restart the server process, we tell the server to exit after handling the current connection, if any. That way deployment doesn't disrupt any pending requests. We tell the server process to gracefully exit by sending it a `SIGHUP` signal.

##### A Quick Rant on Web Frameworks and SO_REUSEPORT

**Note**: a shocking, saddening number of web frameworks force you to call `listen(2)` in your Big Ball Of App Code That Needs To Be Restarted. The [connect](https://github.com/strongloop-forks/connect/blob/7edb875a9f305e38f4d960fa46ac674038241892/lib/proto.js#L231) HTTP server framework used by [express](https://github.com/strongloop/express), the most popular web app framework for [Node.js](https://nodejs.org/), is one of them.

"I'll just use the new [`SO_REUSEPORT` socket option in Linux](https://lwn.net/Articles/542629/)!" you say.

Fine, but take care that at least one server process is always running at any given time. This means some handoff coordination between the old and new server processes. Alternately, you could run an unrelated process on the port that just listens.

At any rate, an `accept(2)`-based server is simpler. It also has some nice added benefits unrelated to deployments:

- An `accept(2)`-based server is network-agnostic. For instance, you can run it behind a Unix domain socket without modifying a single line of code.

- An `accept(2)`-based server is a more secure factoring of concerns. If your server listens directly on a privileged port (80 or 443), you'll need root privileges or a fancy capabilities setup. After binding, a listen server should also drop root privileges (horrifyingly, some don't). The `accept(2)` factoring means a tiny, well-audited program can bind to the privileged port, drop privileges to a minimally empowered user account, and run a known program. This is a huge security win.

#### How does the atomic part work?

A connection will either be served by the old server process or the new server process. The question is whether the old process might possibly see new files, or the new process might see old files. If we update files in-place then one of these inconsistencies can happen. This forces us to keep two complete copies of the files, an old copy and a new copy.

While we're updating the new files, no server process should use them. If the old server process is restarted during this phase, intentionally or accidentally, it should continue to work off the old files. When the new copy is finally ready, we want to "throw the switch": deactivate the old files and simultaneously activate the new files for future server processes. The trick is to make throwing the switch an atomic operation.

<div style="text-align:center;">
<img src="./img/mad-scientist-with-switch.jpg" width="100%"/>
<p>&nbsp;</p>
</div>

There are a number of [things Unix can do atomically](http://rcrowley.org/2010/01/06/things-unix-can-do-atomically.html). Among them: use `rename(2)` to replace a symlink with another symlink. If the "switch" is a simply a symlink pointing at one directory or the other, then deployments are atomic. This is Unix trick #2.

#### What about serving inconsistent assets? Browsers open multiple connections.

This is a problem, but there's also a straightforward solution.

Let's clarify the problem first: during a deployment, a client may request a page from the old server, then open more connections that request assets from the new server. (Remember, consistency is only guaranteed within the same connection.) So you can get old page content mixed with new css, js, images, etc.

In the prevailing practice, the solution is to build a new tagged set of static assets for every deployment, then have the page refer to all assets via this tag. You can do this by modifying the [`./deploy` script](./deploy) to do this, like so:

- Update the new files.
- Generate a unique tag `$TAG`. Epoch timestamps are usually good enough.
- Record `$TAG` in a file inside the new file directory.
- Copy all the static assets into a new directory `assets.$TAG` outside of both file copies.
- Continue with the deployment.

When the server starts up, it should read `$TAG` from the file, and make sure all asset URLs it generates contain `$TAG`.

That's pretty much it. Eventually you'll want to delete them, but if you keep the old `assets.$TAG` directories around for a while, even sessions that haven't reloaded the page will continue to get consistent results across deployments.

The long term solution to this problem is [HTTP/2 multiplexing](https://http2.github.io/faq/#why-is-http2-multiplexed), which makes multiple browser connections unnecessary.

#### What about serving inconsistent ajax requests?

Let's clarify this problem: during a deployment, a client may request a page from the old server, then open more connections that make ajax requests of the new server using old client code.

There's a less technical solution to this one: simply make your API backwards compatible. This is a good idea regardless.

#### What about concurrency? Your example only serves one connection at a time.

You can run as many `accept(2)`-calling server processes as you want on the same listen socket. The kernel will efficiently multiplex connections to them.

In production, I use a small program I wrote called `forkpool` that keeps N concurrent child processes running. It doesn't do anything beyond this, which means it doesn't have any bugs at this point and never needs restarting. Remember, children are a precious resource, but without a parent to keep that listen socket open they're *orphans*.

#### What about deployment collisions?

Yes, you really should prevent concurrent deployments via a lock. That's not demonstrated here, but it's extremely easy and reliable to do with [the setlock(8) program from daemontools](http://cr.yp.to/daemontools/setlock.html).

#### What about deploying database schema changes?

This topic has been covered [well elsewhere](https://blog.rainforestqa.com/2014-06-27-zero-downtime-database-migrations/).

