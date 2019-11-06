# Nginx/Kubernetes Reverse proxy configuration 

This Example demonstrates a simple nginx reverse-proxy configuration that can be used in a local Kubernetes deployment (e.g Docker for Windows).
The https://github.com/kubernetes/ingress-nginx image is used from https://quay.io/repository/kubernetes-ingress-controller/nginx-ingress-controller?tab=info

The instructions were adapted from hackernoon.com/setting-up-nginx-ingress-on-kubernetes-2b733d8d2f45

This nginx is setup with the --watch-namespace flag so that it only monitors the namespace it's deployed into (the 'nginx' namespace).

## Installation

1. generate a self signed cert and key for TLS - FQDN must match the host specified in the backend services ingress (ingress.yml)

    `openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout nginx-selfsigned.key -out nginx-selfsigned.crt`

    `openssl dhparam -out dhparam.pem 2048`

2. create the namespace
   
   `kubectl apply -f namespace.yml`

3. create the tls-certificate secret referenced in the ingress.yml for the backend service
   
    `kubectl create secret tls tls-certificate --key nginx-selfsigned.key --cert nginx-selfsigned.crt --namespace nginx`

    `kubectl create secret generic tls-dhparam --from-file=dhparam.pem --namespace nginx`

4. apply the rest of the yml

    `kubectl apply -f .`


## Useful Notes

### YOU NEED TO ADD THE HOST/IP MAPPINGS TO THE WINDOWS AND/OR LINUX HOSTS FILE ON THE MACHINES YOU'RE TESTING ON !! THIS IS BECAUSE NGINX WILL REDIRECT REQUESTS TO THAT HOST SO IT NEEDS TO BE ABLE TO BE RESOLVED
 
Nginx will only forward requests for the host defined in the ingress to the backend service so trying to hit the nginx service via localhost or anything else will result in an unmatched requests (and should return the default backend 404 response)

You need to add the following to \windows\system32\drivers\etc\hosts in order to be able to hit `https://nginx.triad.co.uk/web1` and have it succesfully
recognised at the nginx server.

`127.0.0.1 nginx.triad.co.uk`
 
When testing this from within the cluster on a linux box, you need to add the IP address of the nginx service mapped to `nginx.triad.co.uk` in /etc/hosts

`<nginx-ingress service ip>  nginx.triad.co.uk`

you can find the IP of the nginx-ingress service from the dashboard or via nslookup nginx-ingress

When testing, you can 'impersonate' the requerst url by inline resolving the host you want to use, with a --resolve option

`curl --insecure -L -v 'https://nginx.triad.co.uk:443/web1/' --resolve 'nginx.triad.co.uk:443:<nginx-ingress service ip>'`

You don't need to do this once you've added the mapping to the hosts file..the following will work 

`curl  --insecure -L -v 'https://nginx.triad.co.uk:443/web1/'`

`curl --insecure -L -v 'https://nginx.triad.co.uk/web1/'`
 
### NOTE ON PATH ELEMENTS IN INGRESSES

The PATH element within the ingress is quite explicit e.g you may find `https://nginx.triad.co.uk:443/web1` doesn't work without the trailing slash.

In the example I've used the path `/web1(/|$)` to denote `/web1` followed buy another `/`, or an end of line (no trailing `/`)

### NOTE ON INGRESS ANNOTATIONS


When we hit `http://nginx.triad.co.uk/web1/` at the nginx server, nginx just forwards the entire request to the backend service
including the `/web1/` which may or may not be what you need.
To work around this we use an annotation in the backend services ingress to rewrite the URL and strip out the elements of the
path specification to create the desired 'new' path for the target backendservice.

In this example case, the backend service only repsonds on `/` so we use the rewrite-target annotation to effectively strip the path 
from the request url as described in `https://kubernetes.github.io/ingress-nginx/examples/rewrite/#rewrite-target`
Basically, the path defines regex 'captured groups' which are references in the rewrite-target

`path: /web1(/|$)(.*)`

`web1` is followed by a `/` or an end of line `$` (no trailing `/`). The slash or end of line is captured group 1 (which can be referenced as `$1` in a rewrite target)
`.*` matches any character/number of characters and this is captured group 2 (`$2`)
so we want `/$2` as the rewrite target to remove /web1 from the new request.

`nginx.ingress.kubernetes.io/rewrite-target: /$2`

### NOTE ON HTTP PORTS


Hitting the LoadBalancer from outside the cluster worked in the first instance using https however it was failing with http on my machine.
The reason for this was that port 80 was in use by an IIS service running in the background. On closer inspection the 404 was coming from IIS and not from nginx. Moving the nginx-ingress-svc http port to another port will work around it, but it's just worth looking closely at any errors you get!

### NOTE ON LoadBalancer URLS


This install provides both an HTTP and HTTPS endpoint, with HTTP forwarding to HTTPS
You may not notice, but in the Kubernetes dashboard, the URLs for the nginx-ingress service are listed as `localhost:85` and `localhost:443` without the http/https protocol prefixes.
clicking on `localhost:443` will give a bad request since the endpoint in the dashboard doesn't specify `https://localhost:443`
so it will try an http connection to 443 and you'll get the fairly obvious error 

`"The plain HTTP request was sent to HTTPS port"`

Nothing to worry about, but might be a source of confusion...just prefix with https if you want to test that route...you just can't click directly on it from the dashboard that's all.

Also....it should be obvious...but `localhost` won't hit your service anyway since the service is  mapped to `nginx.triad.co.uk/web1` not `localhost/web1` so 
you'll just get the default backend unless you give it the proper host. Essentially, the service links in the dashboard are of little use to you.

### STICKY SESSIONS

If you need session affinity (e.g if your webservices create antiforgery tokens locally to the service), uncomment the relevant line in the ingress.yml
Note that the method described in https://github.com/kubernetes/ingress-nginx/blob/master/docs/user-guide/nginx-configuration/annotations.md#session-affinity didn't appear to work for me whereas using remote_addr(clients IP)  to hash-by did.