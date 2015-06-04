# Dream Deploys: Atomic, Zero Downtime Deployments

They're not hard to do. So why do most shops struggle on, afraid to deploy because the site will be unavailable, in an inconsistent state, or both during a deployment window? Let's conquer our fear. Let's deploy whenever we damn well feel like it.

This is a tiny demo project. It's purpose is to convince you that Dream Deploys are not only possible, they're easy. You don't need a fancy load balancer. You don't need magic "clustering" infrastructure. You don't even multiple instances of your server running. All you need is a couple old-school Unix tricks.

## A Quick Demo

In a terminal, run this and visit the link:

    ./serve

In a second terminal, deploy whenever you want, and refresh the page to see it change:

    ./deploy

## Questions

> What do you mean by a "zero downtime" deployment?

At no point is the site unavailable. Requests will continue to be served without transport layer errors before, during, and after the deployment.

> What do you mean by an "atomic" deployment?

In serving a request, either you will talk to the new code working against the new static files, or you will talk to the old code working against the old static files. You will never see a mix of old and new.

> How does the zero downtime part work?

This brings us to Unix trick #1. If you keep the same listen socket open throughout the deployment, clients won't get `ECONNREFUSED` under normal circumstances. The kernel places them in a listen backlog until our server calls `accept(2)`. This means our server process can't be the thing to call `listen(2)` if we want to stop and start it, or we'll incur visible downtime. Something else -- some long running process -- must call `listen(2)` and keep the listen socket open across deployments.

The trick in a nutshell, then, is this:

- A [tiny, dedicated program](./tcplisten) calls `listen(2)` and then passes the listen socket to child processes as descriptor 0 (stdin). This process replaces itself by executing a subordinate program.
- The subordinate program is [just a loop](./loop-forever) that repeatedly executes our server program. Because this loop program never exits, the listen socket on descriptor 0 stays open.
- Our server program, instead of calling `bind(2)` and `listen(2)` like everyone **loves** to do, humbly calls `accept(2)` on stdin in a loop and handles one client connection at a time.
- When it's time to restart the server process, we tell the server to exit after handling the current connection, if any. That way deployment doesn't disrupt any pending requests. We tell the server process to gracefully exit by sending it a `SIGHUP` signal.

> How does the atomic part work?

A given request will either be served by the old server process or the new server process. The question is whether the old process might possibly see new static files, or the new process might see old static files. We can't update any of the static files in-place, or one or the other problematic scenario will happen. This forces us to keep two complete copies of the static files, an old copy and a new copy.

While we're updating the new static files, we don't want the server to start serving them. If the old server process is restarted at this point, intentionally or accidentally, it should continue to serve the old static files. The trick, then, is to make the "throw the switch" act of cutting over from old to new files atomic.

There are a number of [things Unix can do atomically](http://rcrowley.org/2010/01/06/things-unix-can-do-atomically.html). Among them: use `rename(2)` to replace a symlink with another symlink. If the "switch" is a simply a symlink pointing at one directory of files versus the other, then we deployments are atomic. This is Unix trick #2.

> What about concurrency? Your example only serves one connection at a time.

You can run as many `accept(2)`-calling server processes as you want on the same listen socket. The kernel will efficiently multiplex connections to them. In production I use a small program I wrote called `forkpool` that keeps N concurrent child processes running and nothing more.

> What about deployment collisions?

Yes, you really should prevent concurrent deployments via a lock. That's not demonstrated here, but it's extremely easy and reliable to do with [the setlock(8) program from daemontools](http://cr.yp.to/daemontools/setlock.html).

