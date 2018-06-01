Kong Inc. is really excited about all of the amazing features in Kong .32. To that end, we've put together a training and demo to help explain some of the new features.

There is some software that we ask you install prior to coming to the demo. if you are unsure of how to download or install any of this, just let your CSE know and he'll be happy to assist:
	- [Docker](https://docs.docker.com/install/)
		This entire session will be done on your local computer using containers.
	- [httpie](https://github.com/jakubroztocil/httpie)
		Not required, this is. replacement for CURL which is simpler to use and outputs in formatted JSON

**To save time during the demo**, I'd like you to pull a copy of the images we'll be using ahead of time. I will be using Postgres for my example, so you should run the following command prior to the session:
```
  docker pull postgres:9.6
```

You will also need a copy of the Kong Enterprise image. You should have received instructions on how to do so, so if not, please ping JPK and he'll ensure you get access.

Once you have the image downloaded locally, the first thing I like to do is give it a new tag. This is not necessary, but I'd rather not need to type kong-docker-kong-enterprise-edition-docker.bintray.io/kong-enterprise-edition:0.32-alpine every time I launch. To do this, run the following command:
```
docker tag kong-docker-kong-enterprise-edition-docker.bintray.io/kong-enterprise-edition:0.32-alpine kong-ee
```

Now, when you want to use this image, you can simply enter kong-ee.

Because we are using Kong EE, we will also need to use a license file. You should already have access to this file, but if not, please contact JPK. We will set the license as an environmental variable to avoid using it in our commands. To do this, run the following command:
```
export KONG_LICENSE_DATA='<licenseDataHere>'
```
Note if your license has a special charcter like a ' you will need to escape it out with \ 

Now, we can use $KONG_LICENSE_DATA instead of the actual value. 

Now that all of the prep work is done, we're ready to start Kong. You may want to do this ahead of time so you know what to expect, but that is not required. First, we'll start the datastore. Please note that while I'm using Postgres, you could just as easily use Cassandra.

```
# Start postgres
docker run -d --name kong-database \
              -p 5432:5432 \
              -e "POSTGRES_USER=kong" \
              -e "POSTGRES_DB=kong" \
              postgres:9.6
```

Once that has started, we need to run migrations. Kong needs you to run migrations on the DB when it is first started as well as every time you do an upgrade.

```
#run migrations
docker run --rm --name kong \
    --link kong-database:kong-database \
    -e "KONG_LICENSE_DATA=$KONG_LICENSE_DATA" \
    -e "KONG_DATABASE=postgres" \
    -e "KONG_PG_HOST=kong-database" \
    kong-ee kong migrations up
```

Now that the Database is ready, it's time to start Kong:

```
#start Kong
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

We will also start a second instance of Kong, but point it to the same Database. Kong automatically clusters nodes with the same datastore, so there are no additional steps needed.

You may have noticed that I am using different Ports for this node. This was done because both nodes are on the same host and therefore cannot share the same ports. If the second node wass on another host, you could re-use the same ports


```
#Start a second node Note we are using different ports because this is all running locally)
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

You now have Kong running locally. In the training session, we'll be adding routes and services as well as adjusting what each node has the ability to do.
