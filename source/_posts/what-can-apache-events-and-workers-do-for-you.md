---
title: What Can Apache Events and Workers Do for You?
description: You can have your CPU and RAM back...
thumbnail: 0_XFC0-JClTkkgF3CO.jpg 
date: 2019-03-10 15:23:30
updated: 2020-11-27 15:06:27
tags:
    - Programming
    - Performance
    - DevOps
---

<!-- markdownlint-disable no-inline-html -->
![Image](0_XFC0-JClTkkgF3CO.jpg)<span class="caption">Image: [Ian Battaglia](https://unsplash.com/@ianjbattaglia)</span>
<!-- markdownlint-enable no-inline-html -->

There are _too many to count_ articles about configuring Apache or NGINX to optimize for multi-process, multi-threaded scenarios with one or more downstream, detached processors of HTTP requests ([PHP-FPM](https://www.php.net/manual/en/install.fpm.php), [Passenger](https://www.phusionpassenger.com), [ASP.NET](https://docs.microsoft.com/en-us/aspnet/core/host-and-deploy/linux-apache), etc.). This post won't be addressing the step-by-step configuration details. The key takeaway from most articles, that's relevant here, is that it is often very taxing on high-load, performance-critical web servers when all possible request handlers and processors are engaged to satisfy every incoming client request - regardless of context. For example, a web server process doesn't _need_ `ruby` to load a static JavaScript file off disk.

<!-- markdownlint-disable no-space-in-emphasis -->
Due to [MindTouch](https://mindtouch.com)'s {% post_link 'continuous-delivery-without-breaking-everything' evolution %} from delivering downloadable packages for on-premise installs to handling deployment ourselves in a SaaS model, decisions regarding _how_ a web server should be configured to best run our platform were not made by us in the early days. We adopted what seemed to be the most common way to run PHP (our middleware web layout application) with Apache: the [Apache Prefork MPM](https://httpd.apache.org/docs/2.4/mod/prefork.html) (Multi-Processing Model) with the `mod_php` module. _Preforking_ is quite nice and straightforward for serving HTML and static resources such as JavaScript, CSS, and images. With the addition of `mod_php`, every web server process that is forked from the main process has all the tools it needs to handle any supported incoming requests.
<!-- markdownlint-enable no-space-in-emphasis -->

MindTouch also had a unique requirement: a `mono` (.NET) hosted API host with its own rules for handling incoming requests. We used `mod_proxy` to direct traffic for a specific path segment to that downstream request handler.

![Image](0.png)

...and this is how things probably would have remained if it wasn't for _this_ eventual problem:

![Image](2.png)

_Update 2020-11-27_: It is worth mentioning that MindTouch now deploys these components as containers on [Amazon EKS](https://aws.amazon.com/eks) (k8s). When this post was written, all components described ran on EC2 application servers (with every application server configured with the same components).

Yes, on an average day, while our core API processes required less than 40% of available resources, `httpd` (with its bulky PHP add-on) spiked and was often CPU-bound. We analyzed traffic and found that we were loading the entire PHP interpreter to handle PHP-unnecessary requests such as images, JavaScript, and the API. We quickly realized we were not being particularly efficient with our compute resources. Much of our asset delivery was already handled through a content delivery network (CDN) as well, so we weren't even getting the worst of it.

We chose to implement worker threads with the [Event MPM](https://httpd.apache.org/docs/2.4/mod/event.html) for a leaner web server process. ~Our longer-term plans include an initiative to deploy our web server, PHP middleware, and API host on separately scalable units (likely [containers](https://www.docker.com/resources/what-container)), so breaking apart these components seemed like a step in achieving that goal as well.~ (_Update 2020-11-27_: Done!) MindTouch's hands-on VP of Technology, [Pete Erickson](https://www.linkedin.com/in/pete-erickson-b3455a1), was very instrumental in allowing me to execute these changes safely.

![Image](1.png)

For a quick course in how threads operate in this context, I'll do my best with the next few sentences (using the diagram above as a visual aid). The main web server process spawns child processes each with available worker threads and a single listener thread. The worker threads can be assigned to any incoming request received by the web server, and are expected to route the request to the appropriate downstream handler (the file system if fetching a static file, Fast CGI if another interpreter is necessary, etc.). This is already a much more efficient way to handle high rates of web traffic for different downstream destinations. However, the use of _events_ and the listener thread makes this deployment fire on all cylinders.

Typically the worker thread would be _bound_ to the web server socket, waiting for the downstream work to complete before returning some sort of response to the upstream client who originally sent the request. A CPU thread doesn't _need_ to sit around and do nothing while an operating system is trying to locate a file on disk, PHP is processing the received data, or the API is performing work. The listener thread _listens_ for events fired from the main web server process socket queue of incoming requests and the operating system. It works with the process's thread pool to determine when a worker thread needs to be called up to handle inbound or outbound communication for the webserver. If a request is presently being handled by a different process, there is no need for a web server thread to be tied up, and it can be available to handle incoming requests.

Incidentally, for you JavaScript enthusiasts, if this sounds a bit like concurrency as provided by the [JavaScript Event Loop](https://developer.mozilla.org/en-US/docs/Web/JavaScript/EventLoop), it's not 😛! While some similar benefits are realized in [Node.js](https://nodejs.org), such as non-blocking I/O, JavaScript achieves this by managing a _single_ thread very well. In the example above, we are talking about a _multi-threaded_ solution, which is a good use case to apply _parallel computing_.

After tuning the number of child processes and threads, the outcome, with the same steady request rate, were significant:

![Image](3.png)

We traded a collection of nearly CPU-bound Apache processes, for 15-20% utilization by PHP-FPM (the visualized "cliff" for the `httpd` processes represents when this change was fully rolled out to production servers). Getting the configuration right required a lot of testing and a lot of dead "canaries", but the breathing room that we gained back led to a significantly more stable service for our customers (with more room for the occasional spike) and a much happier DevOps team!
