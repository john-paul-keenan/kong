# Before we start:
### Have the following installed on your local machine:
* Docker - https://docs.docker.com/install/
* httpie - https://httpie.org/
### Run the following commands on your machine locally:
* docker pull postgres:9.6
* docker pull kong-docker-kong-enterprise-edition-docker.bintray.io/kong-enterprise-edition:0.32-alpine
		(you will need access to get this)
* docker tag kong-docker-kong-enterprise-edition-docker.bintray.io/kong-enterprise-edition:0.32-alpine kong-ee
* export KONG_LICENSE_DATA='<licenseDataHere>'
		(you will be provided with a licnese file to use. Because your company bname has a ' in it, please make sure to escape that charcter out first with a \)

## Getting Started
Now, we have everything we need to follow along with the demo. We'll do theis part as a group, but first, we'll need to get the datastopre up and running. We'll be using Postgres in this example but could just as easily use Cassandra

```
docker run -d --name kong-database \
              -p 5432:5432 \
              -e "POSTGRES_USER=kong" \
              -e "POSTGRES_DB=kong" \
              postgres:9.6 
```

Now that we have a daatastore up and ready, we need to run migrations on it. Migrations must be run when you start a fresh instance of Kong, or upgrade versions. Migrations should only be run from a single node
```
docker run --rm --name kong \
    --link kong-database:kong-database \
    -e "KONG_LICENSE_DATA=$KONG_LICENSE_DATA" \
    -e "KONG_DATABASE=postgres" \
    -e "KONG_PG_HOST=kong-database" \
    kong-ee kong migrations up
```

Now that the datastore is ready, let';s start our first instance of Kong
```
docker run -d --name kong \
    --link kong-database:kong-database \
    -e "KONG_LICENSE_DATA=$KONG_LICENSE_DATA" \
    -e "KONG_DATABASE=postgres" \
    -e "KONG_PG_HOST=kong-database" \
    -e "KONG_ANONYMOUS_REPORTS=off" \
    -e "KONG_ADMIN_LISTEN=0.0.0.0:8001, 0.0.0.0:8444 ssl" \
    -e "KONG_VITALS=on" \
    -e "KONG_PORTAL=on" \
    -p 8000:8000 \
    -p 8443:8443 \
    -p 8445:8445 \
    -p 8001:8001 \
    -p 8002:8002 \
    -p 8003:8003 \
    -p 8004:8004 \
    -p 8444:8444 \
    kong-ee
```

Assuming that started without an issues, let's start the second node:
```
docker run -d --name kong2 \
    --link kong-database:kong-database \
    -e "KONG_LICENSE_DATA=$KONG_LICENSE_DATA" \
    -e "KONG_DATABASE=postgres" \
    -e "KONG_PG_HOST=kong-database" \
    -e "KONG_ANONYMOUS_REPORTS=off" \
    -e "KONG_ADMIN_LISTEN=0.0.0.0:9001, 0.0.0.0:9444 ssl" \
    -e "KONG_ADMIN_GUI_LISTEN=0.0.0.0:9002, 0.0.0.0:9445 ssl" \
    -e "KONG_PROXY_LISTEN=0.0.0.0:9000, 0.0.0.0:9443 ssl" \
    -e "KONG_PORTAL_API_LISTEN=0.0.0.0:9004, 0.0.0.0:9447 ssl" \
    -e "KONG_VITALS=on" \
    -e "KONG_PORTAL=on" \
    -p 9000:9000 \
    -p 9443:9443 \
    -p 9445:9445 \
    -p 9001:9001 \
    -p 9002:9002 \
    -p 9003:9003 \
    -p 9004:9004 \
    -p 9444:9444 \
    -p 9447:9447 \
    kong-ee
```

You'll notice this is pretty much the exact same as we did for node1, with the only difference being the port numbers. 

With our 2 nodes up and running, let's take a look inside both of them. To go into kong:
`docker exec -it kong /bin/ash`

And to go into the kong2:<br />
`docker exec -it kong2 /bin/ash`

If you remember, when we first started the Kong container, we assigned a lot of the Kong environmental variables. However, if we `cat etc/kong/kong.conf.default`, we’ll notice none of those changes seem to be reflected there. This is because we passed those new values through as environmental variables. Environmental variables are saved `/usr/local/kong/.kong_env` and are removed every time Kong is restarted. because part of this demo will require us to resatart Kong, let's update the actual file. We will only be doing this in kong2, but the same concepts can be repeated in every Kong node.

First, change the name of the file to kong.conf This can be done with the following command:<br />
`mv etc/kong/kong.conf.default etc/kong/kong.conf`

now that the file has the corrct name, let's open it with a text editor:<br />
`vi etc/kong/kong.conf`


Find the following values and update them:<br />
	`anonymous_reports`<br />
	`proxy_listen`<br />
	`admin_listen`<br />
	`admin_gui_listen`<br />
	`vitals`<br />
	`portal`<br />
	`portal_gui_listen`<br />
	`portal_api_listen`<br />

Once they have all been updated, stop Kong with:
`kong stop`

This will styop Kong and put you back on your local machine. To start Kong again, just tell docker to start the container:
`docker start kong`

## Adding Routes and Services

In previous versions of Kong, incomming and outgoing traffic was controlled by the API object. While that is still available, we have replaced it with [Routes](https://getkong.org/docs/0.13.x/admin-api/#route-object) and [Services](https://getkong.org/docs/0.13.x/admin-api/#service-object).

A Service in Kong is where the request will be proxied to. This is typically a service you have somewhere upstream. Services can have a 1:1 or a 1:many relationship with routes, meaning 1 or more route can lead to the same service.

We can add Services either through the GUI, or the commandline. if you prefer the commandline, you can run the following command to add your first service:<br />
`http POST :8001/services host=httpbin.org name=ip path=/ip protocol=http`<br />
This means any request that gets routed to this service will be routed to http://httobin.org/ip

A Route defines how a request will come into KONG. Typically, this is a request to one of the APIs inside of Kong. Kong can identify a route based on any of the following:
	- methoods
	- hosts
	- paths
Let's go ahead and add a route. Take note that is being added as a route to a speific service:<br />
`http POST :8001/services/ip/routes paths:='["/ip"]'`<>br />
This will route any call made to one of the kong nodes at /ip. 

Let's test both nodes:<br />
`http :8000/ip`<br />
`http :9000/ip`

Both of those should have returned your machine's local IP address. 

# TODO
	- add a second service and rate limiting to show the value in this
	- add separation of data/admin planes 
