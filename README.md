# Before we start:
### Have the following installed on your local machine:
* Docker - https://docs.docker.com/install/
* httpie - https://httpie.org/
### Run the following commands on your machine locally:
* First, we will pull a copy of the Postgres Image<br />
`docker pull postgres:9.6`<br />
* Next, pull a copy of Kong Enterprise Edition. You will need credentials to access this image, so please contact your CSE if you have images.<br /> 
`docker pull kong-docker-kong-enterprise-edition-docker.bintray.io/kong-enterprise-edition:0.32-alpine`<br />
* Because that is a rather long image name, tag it to something more managable<br />
`docker tag kong-docker-kong-enterprise-edition-docker.bintray.io/kong-enterprise-edition:0.32-alpine kong-ee` <br />
* Finally, we'll create an enviromental variable for the license file. Please note, there are 2 examples of this, the first is if your company name does not contain any special charcters (for example, ! or '). The second example comments out special charcters to allow special charcters<br />
##### no special charcters
```
export KONG_LICENSE_DATA='{"license":{"signature":"xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx","payload":{"customer":"Example Company","license_creation_date":"2018-05-31","product_subscription":"Kong Enterprise Edition","admin_seats":"5","support_plan":"Platinum","license_expiration_date":"2018-06-14","license_key":"xxxxxxxxxxxxxxxxxx_xxxxxxxxxxxxxxxxxxx"},"version":1}}'
```
##### using special charcters
```
export KONG_LICENSE_DATA="{\"license\":{\"signature\":\"xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx\",\"payload\":{\"customer\":\"JPK's Exmple Company\",\"license_creation_date\":\"2018-05-30\",\"product_subscription\":\"Kong Enterprise Edition\",\"admin_seats\":\"5\",\"support_plan\":\"Platinum\",\"license_expiration_date\":\"2018-06-14\",\"license_key\":\"xxxxxxxxxxxxxxxxxx_xxxxxxxxxxxxxxxxxxx\"},\"version\":1}}"
```

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

Now that the datastore is ready, let's start our first instance of Kong
```
docker run -d --name kong \
    --link kong-database:kong-database \
    -e "KONG_LICENSE_DATA=$KONG_LICENSE_DATA" \
    -e "KONG_DATABASE=postgres" \
    -e "KONG_PG_HOST=kong-database" \
    -e "KONG_VITALS=on" \
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

Assuming that started without an issue, let's start the second node. Note I added `admin_listen`, `proxy_listen` and `admin_gui_listen` to the enviromental variables in this command. I am changing these from the default, 8000, range The enviromental variables set in this command get rewritten everytime the container is restarted and for easeier demoing, I'm only stting it once :
```
docker run -d --name kong2 \
    --link kong-database:kong-database \
    -e "KONG_LICENSE_DATA=$KONG_LICENSE_DATA" \
    -e "KONG_DATABASE=postgres" \
    -e "KONG_ADMIN_LISTEN=0.0.0.0:9001, 0.0.0.0:9444 ssl" \
    -e "KONG_PROXY_LISTEN=0.0.0.0:9000, 0.0.0.0:9443 ssl" \
    -e "KONG_ADMIN_GUI_LISTEN=0.0.0.0:9002, 0.0.0.0:9445 ssl" \
    -e "KONG_PG_HOST=kong-database" \
    -e "KONG_VITALS=on" \
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

With our 2 nodes up and running, let's take a look inside both of them. To go into kong:<br />
`docker exec -it kong /bin/ash`


If you remember, when we first started the Kong container, we assigned a lot of the Kong environmental variables. However, if we `cat etc/kong/kong.conf.default`, we’ll notice none of those changes seem to be reflected there. This is because we passed those new values through as environmental variables. Environmental variables are saved `/usr/local/kong/.kong_env` and are are auto generated every time kong starts. 

### The next steps are not required because Kong remembers the values you have for the enviromental variables. We are changing them because this is a trainning excerise and this change will make parts of what we do later make more sense. 

Let's update some values in the actual kong.conf file. 

First, change the name of the file to kong.conf Otherwise, it will not be used. This can be done with the following command:<br />
`mv etc/kong/kong.conf.default etc/kong/kong.conf`

now that the file has the corrct name, let's open it with a text editor:<br />
`vi etc/kong/kong.conf`


Find the following value and update it as needed fir your system:<br />
    `admin_listen = 0.0.0.0:8001, 0.0.0.0:8444 ssl`<br />
** For security, the Admin API listens on 127.0.0.1 by default. For the demo, we'll update this to 0.0.0.0    
    

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
`http POST :8001/services/ip/routes paths:='["/t/.*"]'`<br />
<cite>Note the `/.*` this is regext that will accept any value</cite>

This will route any call made to the Kong node's root /t. In this case, that is localhost:8000/t/<anything> or localhost:9000/t/<anything>

Let's test both nodes:<br />
`http :8000/t`<br />
`http :9000/t/test`

Both of those should have returned your machine's local IP address. 

### Now, we'll add a second route, inside of the original route.

Endpoints inside of the API can be added as new routes. This gives you the ability to apply plugins to spefic endpoints. 

Maybe you are having a sale on some products, and need to restrict the amount of traffic there. Let's add a route inside of our API to the endpoint with a sale, and apply some rate limiting on it. This will ensure our end users all have an equal chance to access our sale, as well as protect our backend service from getting to heaily trafficed. 

First, simply add the new route:
```
http POST :8001/services/ip/routes \
 paths:='["/sale"]'
 ```

Now, let's add the EE Rate limiting plugin to that endpoint only. You can read the details about all of the configuration paramaters in the [Rate Limiting Advanced Plugin Documentation](https://getkong.org/docs/enterprise/0.32-x/plugins/rate-limiting-advanced/)
```
http POST :8001/routes/<routeID>/plugins \
    name=rate-limiting-advanced \
    config.dictionary_name=kong_rate_limiting_counters \
    config.identifier=ip \
    config.limit=5 \
    config.strategy=cluster \
    config.sync_rate=0 \
    config.window_size=120 \
    config.window_type=sliding
```

Now, let's test that endpoint. Run this command 6 times, and take note of the response on the sixth attempt:<br />
`http :8000/sale`

Note that you received a 429 error on your sixth attempt.

## Separating the Data Plane and Admin Plane

In this part of the training, we will turn our kong node first into a data only node, then into an admin only node.

### Creating a Data only Node

In your kong node, open kong.conf in a text editor<br />
`vi etc/kong/kong.conf`<br />

Find the value `admin_listen` and set its value to:
```
admin_listen = off
```
 Save the file and restart Kong.
```
kong stop
docker start kong
```

Making a request to `:8000/t/anything` will still work, however, attempting to call `:8001/` will now fail. <br />
You should also check the Kong GUI to see an error

### Creating an Admin only Node
In your kong node, open kong.conf in a text editor<br />
`vi etc/kong/kong.conf`<br />

We will be making 2 changes to the file. First, turning the admin plane back on which we turned off in the last step. Second, we will turn off the traffic side
```
admin_listen = 0.0.0.0:9001
proxy_listen = off
```

Save the file and restart Kong.
```
kong stop
docker start kong
```
Making a request to `:8000/anything` will fail, however, attempting to call `:8001/` will now work. <br />
You should also check the Kong GUI and succesfully be able to add administrative actions to Kong.

