# Using NGINX as a Frontend with Docker
You can run a simple instance of nginx with minimal configuration as a load balancing front-end proxy to multiple containers.

## Build the Image
within the app directory run:
```bash
docker build -t app .
```

## Run the container
- Run the following command:

```bash
docker run -p 9000:80 -d app
```

- `-p 9000:80` -> I want remap port 9000 on localhost to be port 80 in the container.

- `-d` -> I want to run the container in the background.  Omitting the `-d` will leave your terminal shell hanging.

- `app` -> The name of the image we just built.

- Now if you get your browser and type `localhost:9000`. You can navigate to that `index.html` page.

# Load Balancing
We could have lots of demands for the page we just created.  So we need to put it behind a load balancer.

## Setting up a Proxy
We are going to set up a proxy, which is also a version of nginx.  We are going to put it in front of our app.

- `proxy/Dockerfile`
```docker
FROM nginx:alpine

RUN rm /etc/nginx/conf.d/*

COPY proxy.conf /etc/nginx/conf.d/
```
- We removing whatever configurations come with the nginx:alpine image from docker hub because we want to upload our own configuration which is done in the third line `COPY`

- `proxy/proxy.conf`
```
server {
    listen 80;

    location / {
        proxy_pass http://app;
    }
}
```
- All this configuration does is listen to port 80 within its container. Any requests that come in basically gets forwarded to the web server running on the host named app the container in this cased named app.

# Connecting the Two Containers
We are going to connet these two together and run them under one system using the docker compose file.

## Docker Compose
In our docker-compose.yml we have the following:
```yml
version : '2'

services:
  app:
    build: app

  proxy:
    build: proxy
    ports:
      - "80:80"
```
- `version: '2'` -> Significant, but basically means we define stuff under our `services:` setion.

- `services:` -> We have two services here.  One called `app:` and another called `proxy:`.

- `app:` -> We get that by building (`build:`) within the stuff that is in the `app` directory (`build: app`).

- `proxy:` -> We are going to map a port (`ports:`) to port 80 in the proxy directory.  
  - Note: (Mac and Linux users should not have a problem with permissions to mapping to port 80 but other OS users might want to try a different mapping.  Be sure to stop any other programs such as apache2 by running `sudo apache2ctl graceful-stop`)

### Run docker-compose
Within the parent directory run:
```bash
docker-compose build
```
- This will build your app and proxy service.

- Then to run the services, run:
```bash
docker-compose up
```

- Note: You could run just `docker-composer up` rather than the build first, but the terminal will spit out a warning for each of the services.

- Now in your browser you can navigate to `localhost:80` to view your page, so our very simple proxy in front of the app is working.

# Significance of docker-compose.yml version: '2'
In older versions of docker-compose we needed to add a `links:` field under `proxy:` to connect the proxy container with the app container as done below:
```yml
version : '2'

services:
  app:
    build: app

  proxy:
    build: proxy
    ports:
      - "80:80"
    links:
      - app
```

- How does the proxy container know where to find the app container in `docker-compose.yml` without the `links:` field under `proxy:`?

- Sol: In this new version of docker, the docker engine runs a little DNS server which when you are using these services, can actually return a DNS entry to each container. Telling it about the other things within this collection of services.

- Within your parent directory run:
```bash
docker-compose ps
```
- This will show the containers that are running.
```bash
           Name                     Command          State         Ports       
-------------------------------------------------------------------------------
nginxdockertutorial_app_1     nginx -g daemon off;   Up      80/tcp            
nginxdockertutorial_proxy_1   nginx -g daemon off;   Up      0.0.0.0:80->80/tcp

```

- `docker-compose exec proxy sh` -> I want to get into the proxy container `docker-compose exec proxy` and run `sh`.  
  - Note: We do not have bash on this because were just using alpine Linux.

- Once your in run `nslookup app`.
    - Note: Ignore the error for a moment.  Sure enough the proxy service knows that the app container lives at that address.

- Run `ping app`.  You'll see we can visit it without acctually having to navigate to it.

- Run `mount`.  This will show all things that are mounted inside your proxy container.  You will see these new lines that were not there before:
```
/dev/mmcblk0p1 on /etc/resolv.conf type ext4 (rw,relatime,discard,errors=remount-ro,data=ordered)
/dev/mmcblk0p1 on /etc/hostname type ext4 (rw,relatime,discard,errors=remount-ro,data=ordered)
/dev/mmcblk0p1 on /etc/hosts type ext4 (rw,relatime,discard,errors=remount-ro,data=ordered)
```

- These lines say `/etc/resolv.conf`, `/etc/hostname`, and `/etc/hosts` all of which determine what this container thinks about itself and what it thinks about other host names.  These are mounted from the outside by a special device in the docker container.

- So if we take a look at `/etc/resolv.conf`, which is where you go to look up host names.  Run:
  - `more /etc/resolv.conf`
```
nameserver 127.0.0.11
options ndots:0
```
- We can see there is a new special DNS server running on `127.0.0.11`.  This is why our proxy container is able to find out about the app container and vice versa.  Thus, they know about eachother and so you do not have to do those `links:` in order to specify which host should know about which other hosts. Basically the new docker system does it for you.



# Proxy
Looking at our `proxy/prox.conf`:
```conf
server {
    listen 80;

    location / {
        proxy_pass http://app;
    }
}
```
- This line `proxy_pass http://app;` is saying pass stuff through to the app container at whatever address the app container is actually started on by docker.

# Recap
We now have a proxy in front of our app but it does not really get us anything very useful.
However, another thing we can do with services is this nice command called `docker-compose scale`.  Run `docker-compose scale -h` within the parent directory, where `-h` is the help flag.

# Docker Compose Scale
The `docker-compose scale` command allows you to run multiple containers for a given service.  So you can essentially do scaling up of your particular service by specifiying `service=number`.
- Note: a command like `docker-compose scale app=4` has been deprecated.  We need to use `up` with the `--scale` flag.

- Run: 
  - `docker-compose up --scale app=4`

- Run `docker ps` to see the additonal containers.

- Now if we go back to our proxy shell, run:
  - `docker-compose exec proxy sh`
- Then run:
  - `nslookup app`
- We can now see four IP addresses back from this special built-in DNS server.  This means that the proxy normally when it asks for one of these will get back all of these addresses.  It can pick one and nginx does `round robin` DNS load balancing by default unless specified otherwise, across those four different containers.
```
Name:      app
Address 1: 172.21.0.5 nginxdockertutorial_app_4.nginxdockertutorial_default
Address 2: 172.21.0.4 nginxdockertutorial_app_3.nginxdockertutorial_default
Address 3: 172.21.0.6 nginxdockertutorial_app_2.nginxdockertutorial_default
Address 4: 172.21.0.2 nginxdockertutorial_app_1.nginxdockertutorial_default
```
#### Reason why `docker-compose scale service=num` was deprecated 
- However, there is a problem with our proxy.conf.
  - It looks up the hostname `proxy_pass http://app;`, at which the configuration is read.
  - If you actually try running `localhost:80` and clicking refresh multiple times.  You will see that all of those requests are only going to my first app container. 
  - nginx has not worked out that there are three other containers running.  Stop it `ctrl-c`, and run `docker-compose up --scale app=4` again.  You will see the other app containers now running as well.  Spam the refresh button and you should see all your containers doing work.
```
app_2    | 172.21.0.6 - - [18/Mar/2018:01:46:58 +0000] "HEAD / HTTP/1.0" 200 0 "-" "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Ubuntu Chromium/63.0.3239.84 Chrome/63.0.3239.84 Safari/537.36" "-"
proxy_1  | 172.21.0.1 - - [18/Mar/2018:01:46:58 +0000] "HEAD / HTTP/1.1" 200 0 "-" "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Ubuntu Chromium/63.0.3239.84 Chrome/63.0.3239.84 Safari/537.36" "-"
app_3    | 172.21.0.6 - - [18/Mar/2018:01:46:58 +0000] "HEAD / HTTP/1.0" 200 0 "-" "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Ubuntu Chromium/63.0.3239.84 Chrome/63.0.3239.84 Safari/537.36" "-"
proxy_1  | 172.21.0.1 - - [18/Mar/2018:01:46:58 +0000] "HEAD / HTTP/1.1" 200 0 "-" "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Ubuntu Chromium/63.0.3239.84 Chrome/63.0.3239.84 Safari/537.36" "-"
```

- We can kill off 2 of the containers by running `docker-compose up --scale app=2` in another terminal within the parent directory.
```
nginxdockertutorial_app_3 exited with code 0
nginxdockertutorial_app_4 exited with code 0
app_1    | 172.21.0.6 - - [18/Mar/2018:01:52:49 +0000] "HEAD / HTTP/1.0" 200 0 "-" "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Ubuntu Chromium/63.0.3239.84 Chrome/63.0.3239.84 Safari/537.36" "-"
proxy_1  | 172.21.0.1 - - [18/Mar/2018:01:52:49 +0000] "HEAD / HTTP/1.1" 200 0 "-" "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Ubuntu Chromium/63.0.3239.84 Chrome/63.0.3239.84 Safari/537.36" "-"
app_2    | 172.21.0.6 - - [18/Mar/2018:01:52:49 +0000] "HEAD / HTTP/1.0" 200 0 "-" "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Ubuntu Chromium/63.0.3239.84 Chrome/63.0.3239.84 Safari/537.36" "-"
proxy_1  | 172.21.0.1 - - [18/Mar/2018:01:52:49 +0000] "HEAD / HTTP/1.1" 200 0 "-" "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Ubuntu Chromium/63.0.3239.84 Chrome/63.0.3239.84 Safari/537.36" "-"
```
- Now if you spam the refresh button you will see only app 1 and 2 doing the work.

# Adjusting Our Proxy Config for Scaling Up or Down
There is a problem with our `proxy/proxy.conf`.  When we try to change the scaling.  In fact sometimes when we scale down and we refresh our browser it blocks because it is trying to talk to one of those containers that is not there anymore (you will notice a latency time before it figures out to select another app container).  So what we would like is for nginx withing the proxy to have a much more dynamic view of the various containers running behind it.  So we fix our config as done below to:
```config
server {
    listen 80;
    resolver 127.0.0.11;
    set $upstream http://app;
    location / {
        proxy_pass $upstream;
    }
}
```
- We added a `resolver` which is using the ip address docker uses for its DNS server.  This is the only slightly dodgy bit of this I am not sure whether that address is defined or that it will stay the same in the future versions of docker and so on but it seems to work reliably at the moment.  There may be a better way of discovering this automatically if anyone else knowns away do let me know.

- We then defined a variable `$upstream`.  Passing the variable in this manner using `proxy_pass` makes our proxy try and understand what the contents of that variable mean in various different ways including looking it up in a `resolver` if it has been given a resolver.  So what this will do is it will now make sure that each time this request comes in, it will use the `resolver` to look it up and that will help load balance across multiple nodes/containers.

- One other thing you want to do particulary in ***testing***, unfortunately when nginx looks things up from this special built-in DNS server, it actually caches the results for quite a long time and so typically 5 mins by default and at least 30 seconds.  So if your scaling things up and down regularly here you actually want to make that cache a lot shorter here so that it will discover changes much more quickly.  So what you can actually add on the end of the resolver is `valid=5s`, giving you `resolver 127.0.0.11 valid=5s;`.  
  - Note: You do not want to do this in ***production***, you would want a longer time in production becase the smaller the number there, the more the load it will put on the little DNS server and so on but for ***experimentation and testing*** this works very nicely.  
- Changing this number to something higher will make the performance higher but it will take more time to see the effects of any scaling change we make.

# Recap proxy.conf
```config
server {
    listen 80;

    resolver 127.0.0.11 valid=5s;
    set $upstream http://app;
    location / {
        proxy_pass $upstream;
    }
}
```
- With that little change to the proxy config were saying:
    - `resolver 127.0.0.11 vaild=5s` -> go and find out names from `127.0.0.11` and rember those results for 5 seconds (`valid=5s`).
    - `set $upstream http://app;` -> set your upstream variable to `http://app`.
    - `proxy_pass $upstream` -> proxy pass onto the value of that upstream variable using a `resolver` to look up anything in here if you don't understand it already.

- Now with those changes in place stop the services `ctrl-c`.  

- Run within parent directory: `docker-compose build` to ensure we've got a new version with this updated proxy config.

- Run within parent directory: `docker-compose up`.

- Open another terminal shell, and run within parent directory: `docker-compose scale app=30`

- Spam refresh and you should see all 30 doing work:
```
app_28   | 172.21.0.3 - - [18/Mar/2018:02:38:39 +0000] "GET / HTTP/1.0" 304 0 "-" "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Ubuntu Chromium/63.0.3239.84 Chrome/63.0.3239.84 Safari/537.36" "-"
proxy_1  | 172.21.0.1 - - [18/Mar/2018:02:38:40 +0000] "GET / HTTP/1.1" 304 0 "-" "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Ubuntu Chromium/63.0.3239.84 Chrome/63.0.3239.84 Safari/537.36" "-"
app_28   | 172.21.0.3 - - [18/Mar/2018:02:38:40 +0000] "GET / HTTP/1.0" 304 0 "-" "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Ubuntu Chromium/63.0.3239.84 Chrome/63.0.3239.84 Safari/537.36" "-"
proxy_1  | 172.21.0.1 - - [18/Mar/2018:02:38:41 +0000] "GET / HTTP/1.1" 304 0 "-" "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Ubuntu Chromium/63.0.3239.84 Chrome/63.0.3239.84 Safari/537.36" "-"
app_18   | 172.21.0.3 - - [18/Mar/2018:02:38:41 +0000] "GET / HTTP/1.0" 304 0 "-" "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Ubuntu Chromium/63.0.3239.84 Chrome/63.0.3239.84 Safari/537.36" "-"
app_16   | 172.21.0.3 - - [18/Mar/2018:02:38:41 +0000] "GET / HTTP/1.0" 304 0 "-" "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Ubuntu Chromium/63.0.3239.84 Chrome/63.0.3239.84 Safari/537.36" "-"
proxy_1  | 172.21.0.1 - - [18/Mar/2018:02:38:41 +0000] "GET / HTTP/1.1" 304 0 "-" "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Ubuntu Chromium/63.0.3239.84 Chrome/63.0.3239.84 Safari/537.36" "-"
proxy_1  | 172.21.0.1 - - [18/Mar/2018:02:38:42 +0000] "GET / HTTP/1.1" 304 0 "-" "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Ubuntu Chromium/63.0.3239.84 Chrome/63.0.3239.84 Safari/537.36" "-"
```

- Now because the resolver was timing out fairly promptly within 5 seconds of me making that scaling change, nginx was discovering that there are a whole lot more containers it could use because the DNS was returning in that case 30 names rather than just the 30 IP addresses rather than just the view had before. 

- Similarly if we scale it back down to 3 (in another terminal run):
```
docker-compose scale app=3
```
- It will kill off all the other containers leaving only 3.


