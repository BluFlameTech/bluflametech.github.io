---
title: "Sidecar Containers"
subtitle: Sometimes Your Container Needs Friend
date: 2021-11-18T09:00:00-04:00
header:
  teaser: /assets/images/posts/sidecar_containers/sidecar_teaser.png
categories:
- blog
tags:
- containers
- kubernetes
- solutions
- deployments
- devops
---

Sidecar containers can be a great way to provide atomic Deployments that bind containers together. But like all things,
sidecar containers can have drawbacks, like increased deployment times. 

![Sidecar](/assets/images/posts/sidecar_containers/sidecar.png)

_<small>Photo by [LuckyBusiness](https://www.istockphoto.com/portfolio/LuckyBusiness?mediatype=photography) on [iStock](https://www.istockphoto.com) by Getty Images</small>_

Here, we talk about sidecar containers, how to create [Deployments](/blog/kubernetes/#deployments) that use sidecar containers, how sidecar containers 
talk to each other and the considerations that need to be taken into account before you implement a Deployment that uses 
one or more sidecar containers.

### What Are Sidecar Containers?

[Kubernetes](/blog/kubernetes) runs Docker containers in [Pods](/blog/kubernetes/#pods), which in many cases contain only a single container.
But, Pods don't have to contain only one container. When pods have multiple containers, we call those extra containers
_sidecar containers_. 

### Why Would You Have a Sidecar Container?

There are a few reasons to have a sidecar container. But the biggest reason is probably because you have multiple
processes that are highly dependent upon each other - when one updates, you want to make 100% sure that other updates, too.
Of course, you are not limited to only a single sidecar, you can have multiple sidecar containers in a Pod. But the more
sidecar containers you have, the more complex your Deployment becomes and the longer it can take to complete a deployment
(discussed more below). This is why typically when a sidecar container is used, there is only one - that's not a hard rule,
it's just less common to have more than one sidecar container if there are any at all.

### Example: Express Container + NGINX Sidecar Container

In this example, we will show a complete Deployment + LoadBalancer [Service](/blog/kubernetes/#services) that you can run on your local Kubernetes
installation. We assume some basic familiarity with Kubernetes, [Docker](https://www.docker.com/) and [Helm](https://helm.sh/).

What we will achieve is an Express (NodeJS) deployment with an NGINX sidecar that acts as a reverse proxy to Express. You could,
of course, use a Kubernetes [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/) instead of an NGINX
sidecar container. However, there are some benefits to using an NGINX sidecar container: more control, in-cluster L7 routing, ability
to host/serve static artifacts without burdening Express, etc.

_Why have NGINX in a sidecar instead of a different Pod?_

That is a natural question, which will be answered with a scenario.

Let's assume that Express acts as a REST API. Let's further assume that Express only accepts requests that have a path starting
with _/api/_. The NGINX configuration is pretty static; it consistently routes all requests that start with _/api/_ to Express.
However, as its functionality increases, let's assume there is a need for a less protected api, which is prefixed with _/public/_ in its path.
Now, both _/api/_ and _/public/_ must be routed to Express. To do that, a change must be made to NGINX. We want to be 100% sure that
when a request is made to _/public/_, and it is routed to Express then Express will be updated to be able to handle the request.
If Express is deployed in a separate Pod then there is no such guarantee. In effect, Express's reverse proxy (NGINX) should be bound
to its container - that is what a sidecar container does.

Additionally, sidecar containers also eliminate networking concerns between other grouped containers. Arguably, this is 
a secondary concern since presumably, any deployments would be tested prior to installations or updates. But, networking
between containers in the same Pod is treated like networking between processes on the same computer - different ports on
localhost. It doesn't get any easier than that. And, of course, the ports that are exposed within the Pod can be restricted
to the Pod.

#### The Express Container

First, we will set up the Express container. To do this, we will first create a NPM project.

In your favorite bash or bash-like terminal (assuming you already have [NodeJS](https://nodejs.org) installed), run the following:
```bash
mkdir express && cd express && npm init
```

Hit \[Enter] all the way through the different input items and when you are done, you should have a _package.json_ file.

In the same terminal, run the following to install Express:
```bash
npm install express --save
```

Now, we need an application setup that uses Express. So, create a file called _app.js_ in your _express_ directory, edit
it and make it look like this:
```javascript
const express = require('express')
const app = express()
const port = 3000

app.get('/', (req, res) => {
    res.send('Hello from Express!')
})

app.listen(port, () => {
    console.log(`listening at http://localhost:${port}`)
})
```

Once we have Express configured in the _app.js_ file, let's configure a NPM script to run it. Edit your _package.json_ and make it look like this:
```javascript
{
  "name": "express",
  "version": "1.0.0",
  "description": "",
  "main": "app.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1",
    "start": "node app.js"
  },
  "author": "",
  "license": "ISC",
  "dependencies": {
    "express": "^4.17.1"
  }
}
```

Now, if you open a terminal in your _express_ directory and type ```npm start``` then you should see something like this:
```bash

> express@1.0.0 start
> node app.js

listening at http://localhost:3000
```

If you go open a browser and navigate to _http://localhost:3000_ then you should see 'Hello from Express!'

OK, now let's make that happen inside a Docker container.

In the _express_ directory, create a _Dockerfile_, which is just a text file named _Dockerfile_. And, make your Dockerfile look like this:
```dockerfile
FROM node:17

#use bash shell
SHELL ["/bin/bash", "-c"]

#configure express user
RUN adduser --disabled-password --home /home/express express
USER express:express
ENV HOME=/home/express

#copy over necessary files
COPY package.json ${HOME}/package.json
COPY app.js ${HOME}/app.js

#run the app
WORKDIR ${HOME}
RUN npm install
ENTRYPOINT npm start
```

You can build your Docker image by opening a terminal in the same directory as the Dockerfile (the _express_ directory)
and running the following:
```bash
docker build -t myexpress:latest .
```

That command instructs Docker to build an image named _myexpress_, with version _latest_ using the Dockerfile in the current directory.

Test it out by opening a terminal and running the following:
```bash
docker run -p 3000:3000 -d myexpress:latest
``` 

Then, navigate to _http://localhost:3000_ in your browser. You should see a web page that says 'Hello from Express!'

To terminate the container, type the following:
```bash
docker kill $(docker ps | grep myexpress | awk 'END {print $1}')
```

#### The NGINX Container

NGINX will be a reverse proxy to our Express container. So, let's start out by creating a NGINX project directory along 
with the NGINX configuration.

Again, in your favorite bash or bash-like terminal, run the following:
```bash
mkdir nginx
```

In the _nginx_ directory that we just created, create a file called _default.conf_ and make it look like this:
```
server {
    listen       80;
    listen  [::]:80;
    server_name  localhost;

    root /usr/share/nginx/html;

    location / {
        index index.html index.htm Default.htm;
    }

    location /express/ {
        proxy_pass http://localhost:3000/;
    }

    error_page 404 /404.html;
    error_page 500 502 503 504 /50x.html;
    location = /50x.html {
        root /usr/share/nginx/html;
    }
}
```

The above configuration tells NGINX to route any requests that begin with _/express/_ to Express and to serve all other
requests itself. So, let's give it something to serve.

Create an _index.html_ file in the same _nginx_ directory and make it look like this.
```html
<html>
    <head><title>NGINX</title></head>
   <body>Hello from NGINX!</body>
</html>
```

Now, let's create a _Dockerfile_ in the same nginx directory. It should look like this.

```dockerfile
FROM nginx
ENV WWW_ROOT /usr/share/nginx/html
COPY *.html ${WWW_ROOT}/
COPY default.conf /etc/nginx/conf.d/
```

Let's try it out!

First, we need to build the image by running the following from the terminal in the _nginx_ directory:
```bash
docker build -t mynginx:latest .
```

Then, we need to run a container using the image we just created by executing the following from the terminal in the _nginx_ directory:
```bash
docker run -p 80:80 -d mynginx:latest
```

While the container is running, you should be able to open a browser, navigate to _http://localhost_ and see 'Hello from NGINX!'

You might notice that if you have both a _mynginx_ and a _myexpress_ container running then navigating to _http://localhost/express/_ does not work. 
Don't worry, though. Kubernetes will fix that for us. And that's what we are doing next!

#### Creating a Kubernetes Deployment

We could create a Kubernetes Deployment that includes Express and NGINX and then attach a LoadBalancer Service to that
Deployment. But instead, we are going to use Helm to do all of that at once under a single installation. 

One thing that should be noted is that we will be using Helm v3.6.3 and that is important because Helm introduced some breaking 
changes with v3.7 (so much for [semantic versioning](https://semver.org/)). This is a pretty simple deployment. So, it will
probably work in the latest version of Helm, but we have only confirmed that it does work v3.6.3. So, who knows?
¯\\\_(ツ)\_/¯

_Note: This was also tested using Kubernetes v1.21.5_

Let's start by first creating a separate directory for our Helm chart called _helm_. Inside our _helm_ directory, we will
create a _Chart.yaml_ file that looks like this.
```yaml
apiVersion: v2
name: sidecar
appVersion: latest
description: Express with NGINX sidecar
version: "1.0.0"
type: application
```

Inside the _helm_ directory, we will also create a _templates_ directory. So, our directory structure now looks like this.

```
- express
- helm
  - templates
- nginx
```

Inside the _templates_ directory, we need two files (OK, you can smush it all into one. But two is more readable, IMHO.):
_deployment.yaml_ and _loadbalancer.yaml_.

Make the _deployment.yaml_ file look like this.
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sidecar
  labels:
    app: sidecar
spec:
  replicas: { { .Values.replicas | default 1 } }
  selector:
    matchLabels:
      app: sidecar
  template:
    metadata:
      labels:
        app: sidecar
        type: deployment
    spec:
      containers:
        - name: nginx
          image: mynginx:latest
          imagePullPolicy: "IfNotPresent"
          ports:
            - name: web-port
              containerPort: 80
          livenessProbe:
            httpGet:
              path: /index.html
              port: web-port
            failureThreshold: 2
            periodSeconds: 30
          startupProbe:
            httpGet:
              path: /index.html
              port: web-port
            failureThreshold: 10
            periodSeconds: 30
        - name: express
          image: myexpress:latest
          imagePullPolicy: "IfNotPresent"
          ports:
            - name: app-port
              containerPort: 3000
          livenessProbe:
            httpGet:
              path: /
              port: app-port
            failureThreshold: 2
            periodSeconds: 30
          startupProbe:
            httpGet:
              path: /
              port: app-port
            failureThreshold: 10
            periodSeconds: 30
```

The above _deployment.yaml_ configuration tells Kubernetes to create a Deployment (a grouping of Pod replicas), each with two containers: a NGINX container and an Express container.
By default, the number of replicas is one. So, by default the deployment will create one replica (Pod) with two containers, one for NGINX and one for Express.

Our Helm chart is just missing one thing: a LoadBalancer. LoadBalancers are a type of Kubernetes networking service that
let us access our Pods externally. They work on Layer 4 of the OSI Model - the same as TCP. That means our LoadBalancer
will not be able to route based on the contents of the message, like the URL path. However, we don't need it to route based on the URL path
because NGINX does that! We just need it to expose NGINX externally, and it is perfectly suited to do just that.

So, let's create a _loadbalancer.yaml_ file in our _templates_ directory that looks like this:
```
apiVersion: v1
kind: Service
metadata:
  name: sidecar-lb
  labels:
    app: sidecar
    type: loadbalancer
spec:
  type: LoadBalancer
  selector:
    app: sidecar
    type: deployment
  ports:
    - name: web
      port: 80
      targetPort: 80
```

OK, let's try it out. At this point, it is assumed that Kubernetes is installed and running on your local workstation and that
both the _mynginx_ and the _myexpress_ images have been built.

Open a terminal in your _helm_ directory run the following.
```bash
helm install sidecar ./
```

Because there is a startup probe configured for both of the containers in the Pod with a 30-second test period, it will
take about 30 seconds for the Pod to be considered live and therefore, accessible via the LoadBalancer. But, after 30-seconds,
you should be able to navigate to _http://localhost_ in your browser and see 'Hello from NGINX!' Then, if you navigate to _http://localhost/express/_
you should see 'Hello from Express!' Congratulations! You just created a Kubernetes deployment with a sidecar container!

BTW, if you want to uninstall your LoadBalancer and your Deployment, in the _helm_ directory, just run the following.
```bash
helm uninstall sidecar
```

### Why Would You Not Want a Sidecar Container?

There are good reasons to have a sidecar container. But there are also good reasons to not have a sidecar container. 
If the functionality you are deploying does not require a strict dependency between containers then you should probably not deploy a sidecar 
container. Honestly, not having a sidecar container should be the default. 

To put it more bluntly...

> Only use a sidecar container if you have a really good reason to do so.

Here are a couple of reasons why you might not want to use a sidecar container.

**1. Sidecar containers couple containers together within a Pod**

Our example above is a pretty decent use case for a sidecar container. The reasoning was explained above. But take a different
scenario. What if we had two different microservices that worked together as part of a more robust API. Should we include
both of those services in the same Pod? Almost certainly not. Why? Because it binds them together - one will not be able 
to exist without the other. If one fails and is unrecoverable, that means the Pod fails. If that is because of a problem in 
one of the containers that cannot be resolved by bringing up a new Pod then the Deployment fails - one service brings down the
other. That is not something you want in a microservices architecture.

**2. Pod availability is based on the container with the longest load time**

Let's say you have a relatively simple web application that consists of a frontend container and a backend API container. Let's further
say that the API is pretty stable (does not change much). However, the frontend changes frequently. If the API takes 5min to start
and the frontend takes 30-seconds to start, and if they are both in the same Pod then it will take the Pod 5min to start. So, 
by combining both the frontend and the API into the same container, frontend deployments that otherwise would have taken
30-seconds to start, now take 10 times that.

### Summary

When Pods have multiple containers, we call those extra containers _sidecar containers_. Sidecar containers can be a 
great way to combine the functionality of multiple containers into a single atomic deployment. But they are not without
[their drawbacks](/blog/sidecar-containers/#why-would-you-not-want-a-sidecar-container), which is why they should be used with caution.

---

### Resources

* [Source Code for Example in GitHub](https://github.com/BluFlameTech/examples/tree/main/kubernetes/sidecar)
* [Docker Reference Documentation](https://docs.docker.com/reference/)
* [Dockerfile Reference](https://docs.docker.com/engine/reference/builder/)
* [Kubernetes Reference Documentation](https://v1-21.docs.kubernetes.io/docs/home/supported-doc-versions/)
* [NGINX Reverse Proxy](https://docs.nginx.com/nginx/admin-guide/web-server/reverse-proxy/)
* [Express Hello World Example](https://expressjs.com/en/starter/hello-world.html)
* [Stackoverflow: Kubernetes cannot pull local image](https://stackoverflow.com/questions/36874880/kubernetes-cannot-pull-local-image)
