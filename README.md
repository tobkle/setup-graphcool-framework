# Guide to Setup a GraphQL Server Backend with graphcool/framework in Docker on an AWS EC2 instance
The GraphQL Server Backend https://www.graph.cool was open-sourced most recently. You can find it here: https://github.com/graphcool/framework

This is a step-by-step guide to setup a Graphcool Server on Amazon AWS Web services. Here, we deploy the graphcool full-example in Docker on our own EC2 instance. This example includes a webshop backend with signup, login, product, order and stripe payments.

## Steps
* setup AWS EC2 with docker-machine
* setup AWS RDS as our mysql endpoint
* setup Traefik as our reverse proxy and load balancer
* setup Letsencrypt to secure endpoints with SSL certificates
* setup Graphcool as docker containers

## Prepare aws
* Create an AWS account and create IAM credentials for the usage of an AWS EC2 administrator.
* Install the aws-cli command line programs.
* Configure aws on your pc/notebook with the provided credentials.

## Prepare stripe
* Create an account on https://www.stripe.com Dashboard
* Switch on "View test data"
* Go to API
* Get Publishable key "pk_test_...."
* Get Secret key "sk_test_..."

## Prepare docker-machine
As a preparation for the docker-machine usage, we need some command files in order to...

* create and start a docker-machine
* connect to the new docker-machine
* inspect the docker-machine (getting the public ip)
* ssh to log into the docker-machine (prepare filesystem)
* remove to kill the docker-machine if no longer required

Run in terminal:
```bash
mkdir aws-box
cd aws-box
touch start.sh
touch connect.sh
touch inspect.sh
touch ssh.sh
touch remove.sh
chmod +x start.sh
cmmod +x connect.sh
chmod +x ssh.sh
chmod +x inspect.sh
chmod +x remove.sh
```

Create file start.sh with `vi start.sh`:
```bash
#!/bin/bash
docker-machine create \
        --driver amazonec2 \
        --amazonec2-open-port 80 \
        --amazonec2-open-port 443 \
        --amazonec2-open-port 22 \
        --amazonec2-open-port 8080 \
        --amazonec2-open-port 2376 \
        --amazonec2-region eu-central-1 \
        --amazonec2-zone=b \
        aws-box
```
Here we use the docker-machine command, to create a new EC2 micro instance in the Frankfurt datacenter (eu-central-1) and availability "zone-1b" by using the amazonec2 driver. We open the following ports to the public for later usage:
* http port: 80
* https port: 443
* ssh port: 22
* admin port: 8080 (for Traefik load balancer)
* docker port: 2376

The name of or EC2 instance and docker-machine name will become "aws-box".

Create file connect.sh with `vi connect.sh`:
```bash
#!/bin/bash
#
# We will copy the ouput of the "start.sh" run
# into this file later.
# Example:
#
# export DOCKER_TLS_VERIFY="1"
# export DOCKER_HOST="tcp://18.192.200.34:2376"
# export DOCKER_CERT_PATH="/Users/toby/.docker/machine/machines/aws-box"
# export DOCKER_MACHINE_NAME="aws-box"
#
## Run this command to configure your shell:
# eval $(docker-machine env aws-box)
#
# Add this to your ~/.bashrc or ~/.zshrc:
# source ~/aws-box/connect.sh
# If you want to have this machine as your default docker-machine
```
Here with set some environment variables and with eval we start docker-machine aws-box as our active docker environment.

Create file inspect.sh with `vi inspect.sh`:
```bash
#!/bin/bash
docker-machine inspect aws-box
```
This is only to get information about our docker-machine instance, such as the public and private IPAddress.

Create file ssh.sh with `vi ssh.sh`:
```bash
#!/bin/bash
docker-machine ssh aws-box
```
As docker-machine already provisioned a ssh certificate, we can login to our aws-box easily with this command. With that, we get into a command shell on our box. So we are able to prepare the file system.

Create file remove.sh `vi remove.sh`:
```bash
#!/bin/bash
docker-machine rm aws-box
```
If we don't need our aws-box any more, we can remove it with that command.

## Create docker-machine
In your terminal run...
```bash
./start.sh
```

Copy the output and add this into your `vi connect.sh`.
It should be something like:
```bash
export DOCKER_TLS_VERIFY="1"
export DOCKER_HOST="tcp://18.193.211.53:2376"
export DOCKER_CERT_PATH="~/.docker/machine/machines/aws-box"
export DOCKER_MACHINE_NAME="aws-box"
# Run this command to configure your shell:
eval $(docker-machine env aws-box)
```
If there is a comment # in front of the eval line, remove it, as it has to run to make your new docker-machine as your default machine.

Run it once in your terminal with...
```bash
source ./connect.sh
```

And/or add it to your default environment by adding this line at the end of your ~/.bashrc or ~/.zshrc. Do a 
```bash
source ~/.bashrc
```
or for zsh shell a 
```bash
source ~/.zshrc
```

To test if your machine was created and connected successfully, run 
```bash
docker-machine ls
```
In the output your new aws-box should be marked * active in there.

To get your public ip address of your new EC2 machine, run in terminal 
```bash
./inspect.sh | grep 'IPAddress'
```

You should get something like:
```bash
❯ ./inspect.sh | grep 'IPAddress'
        "IPAddress": "18.199.200.53",
        "PrivateIPAddress": "172.31.25.92",
```

## Prepare docker-compose
Login to our aws-box via ssh by running our new command
```bash
./ssh.sh
```
Now we should see the command shell of our newly created EC2 instance (Ubuntu 16.04 LTS).

First we should update the machine image by:
```bash
sudo apt-get update
sudo apt-get upgrade
```

We will need later docker-compose, which isn't installed automatically:
```bash
sudo curl -L https://github.com/docker/compose/releases/download/1.17.0/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
```

Add the following line to the end of your ~/.bashrc or ~/.zshrc
```bash
export PATH=$PATH:/usr/local/bin
```

Then reload your shell environment by running:
```bash
source ~/.bashrc
```
Now you should get the version number of docker-compose by running:
```bash
docker-compose --version
```

## Prepare Reverse Proxy
In your aws-box-1 enter in terminal:
```bash
sudo su
mkdir -p /opt/traefik
cd /opt/traefik
touch docker.sh
touch traefik.toml
touch acme.json
chmod 0600 acme.json
chmod +x docker.sh
sudo apt-get install apache2-utils
htpasswd -nb admin secure_password
```
The output from the program will look for example like this:
```bash
admin:$apr1$ruca84Hq$mbjdMZBAG.KWn7vfN/SNK/
```

Copy the line with your user and encrypted password into your clipboard and add it into the following file at the apporpriate position in the [web] section.

Add the following lines to /opt/traefik/traefik.toml
```bash
debug = false
checkNewVersion = true
logLevel = "ERROR"
defaultEntryPoints = ["https","http"]

[entryPoints]
  [entryPoints.http]
  address = ":80"
    [entryPoints.http.redirect]
    entryPoint = "https"
  [entryPoints.https]
  address = ":443"
  [entryPoints.https.tls]

[retry]

[web]
address = ":8080"
  [web.auth.basic]
  users = ["admin:$apr1$ruca84Hq$mbjdMZBAG.KWn7vfN/SNK/"]

[docker]
endpoint = "unix:///var/run/docker.sock"
domain = "aws-box.example.com"
watch = true
exposedbydefault = true

[acme]
email = "me@example.com"
storage = "acme.json"
entryPoint = "https"
onHostRule = true
onDemand = false
```
See the documention of https://traefik.io

With the following example I figured out, how it really works:
https://www.digitalocean.com/community/tutorials/how-to-use-traefik-as-a-reverse-proxy-for-docker-containers-on-ubuntu-16-04

Add the following lines to /opt/traefik/docker.sh
```bash
#!/bin/bash
docker run -d \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v $PWD/traefik.toml:/traefik.toml \
  -v $PWD/acme.json:/acme.json \
  -p 80:80 \
  -p 443:443 \
  -l traefik.frontend.rule=Host:monitor.aws-box.example.com \
  -l traefik.port=8080 \
  --network proxy \
  --name traefik \
  traefik:1.3.6-alpine --docker
```
## Start Reverse Proxy
Run in your terminal sudo /opt/traefik/docker.sh 

Check if your traefik docker container has started successfully by running:

```bash
sudo docker ps
sudo docker logs traefik
```

Got to your web browser and enter URL:
```bash
https://monitor.aws-box.example.com
```

It asks you to enter your user and password for the basic authentication. You can store in your browser cache for later revisits.

You should see the traefik monitor now.

## Prepare Graphcool
Now we start with Graphcool. Therefore go into your aws-box and add in the terminal:
```bash
sudo su
mkdir -p /opt/graphcool
mkdir -p /opt/graphcool/faas
touch docker-compose.yml
docker network create proxy
```

Add the following lines to the /opt/graphcool/docker-compose.yml file:
```bash
version: "3.1"

services:

  localfaas:
    image: graphcool/localfaas:0.9.0
    restart: always
    ports:
      - "0.0.0.0:60050:60050"
    volumes:
      - /opt/graphcool/faas:/var/faas
    environment:
      FUNCTIONS_PORT: 60050
    labels:
      traefik.backend: "localfaas"
      traefik.frontend.rule: "Host:localfaas.aws-box.example.com"
      traefik.docker.network: "proxy"
      traefik.port: "60050"
    networks:
      - internal
      - proxy

  graphcool:
    image: graphcool/graphcool-dev:0.9.0
    restart: always
    ports:
      - "0.0.0.0:60000:60000"
    labels:
      traefik.backend: "graphcool"
      # change to your domain:
      traefik.frontend.rule: "Host:graphcool.aws-box.example.com"
      traefik.docker.network: "proxy"
      traefik.port: "60000"
    networks:
      - internal
      - proxy
    environment:
      PORT: "60000"
      FUNCTION_ENDPOINT_INTERNAL: "http://localfaas:60050"
      
    # the important lines, add your domain here:
      FUNCTION_ENDPOINT_EXTERNAL: "https://localfaas.aws-box.example.com"
      CLIENT_API_ADDRESS: "https://graphcool.aws-box.example.com/"
      API_ENDPOINT_EU_WEST_1: "https://graphcool.aws-box.example.com/simple/v1"
      API_ENDPOINT_US_WEST_2: "https://graphcool.aws-box.example.com/simple/v1"
      API_ENDPOINT_AP_NORTHEAST_1: "https://graphcool.aws-box.example.com/simple/v1"
      SCHEMA_MANAGER_ENDPOINT: "https://graphcool.aws-box.example.com/schema-manager"
      BACKEND_API_SIMPLE_V1_ADDR: "https://graphcool.aws-box.example.com/system"
            
    # replace them with your own secrets:
      MASTER_TOKEN: "my-master-token"
      SYSTEM_API_SECRET: "system-api-secret"
      JWT_SECRET: "my-jwt-secret"
      PRIVATE_CLIENT_API_SECRET: "my-client-api-secret"
      SCHEMA_MANAGER_SECRET: "evenmoresecretwow"
      
    # replace with your mysql client host connection here
    # for example from AWS RDS mysql database
    # replace also with your own database user and password
      SQL_CLIENT_HOST_CLIENT1: "mysqldb.casdfbasdf.eu-central-1.rds.amazonaws.com"
      SQL_CLIENT_HOST_READONLY_CLIENT1: "mysqldb.casdfbasdf.eu-central-1.rds.amazonaws.com"
      SQL_CLIENT_HOST: "mysqldb.casdfbasdf.eu-central-1.rds.amazonaws.com"
      SQL_CLIENT_PORT: "3306"
      SQL_CLIENT_USER: "graphcool"
      SQL_CLIENT_PASSWORD: "graphcool"
      SQL_CLIENT_CONNECTION_LIMIT: "10"
      
      SQL_INTERNAL_DATABASE: "graphcool"
      SQL_INTERNAL_HOST: "mysqldb.casdfbasdf.eu-central-1.rds.amazonaws.com"
      SQL_INTERNAL_PORT: "3306"
      SQL_INTERNAL_USER: "graphcool"
      SQL_INTERNAL_PASSWORD: "graphcool"
      SQL_INTERNAL_CONNECTION_LIMIT: "10"
      
      SQL_LOGS_DATABASE: "logs"
      SQL_LOGS_HOST: "mysqldb.casdfbasdf.eu-central-1.rds.amazonaws.com"
      SQL_LOGS_PORT: "3306"
      SQL_LOGS_USER: "graphcool"
      SQL_LOGS_PASSWORD: "graphcool"
      SQL_LOGS_CONNECTION_LIMIT: "10"
      
      SQL_CLIENT_HOST_EU_WEST_1_CLIENT1: "mysqldb.casdfbasdf.eu-central-1.rds.amazonaws.com"
      SQL_CLIENT_PORT_EU_WEST_1: "3306"
      SQL_CLIENT_USER_EU_WEST_1: "graphcool"
      SQL_CLIENT_PASSWORD_EU_WEST_1: "graphcool"
      
      SQL_CLIENT_HOST_US_WEST_2_CLIENT1: "mysqldb.casdfbasdf.eu-central-1.rds.amazonaws.com"
      SQL_CLIENT_PORT_US_WEST_2: "3306"
      SQL_CLIENT_USER_US_WEST_2: "graphcool"
      SQL_CLIENT_PASSWORD_US_WEST_2: "graphcool"
      
      SQL_CLIENT_HOST_AP_NORTHEAST_1_CLIENT1: "mysqldb.casdfbasdf.eu-central-1.rds.amazonaws.com"
      SQL_CLIENT_PORT_AP_NORTHEAST_1: "3306"
      SQL_CLIENT_USER_AP_NORTHEAST_1: "graphcool"
      SQL_CLIENT_PASSWORD_AP_NORTHEAST_1: "graphcool"

      # for compatibility reasons, will be removed in later releases:
      AWS_REGION: "eu-west-1"
      FILEUPLOAD_S3_AWS_ACCESS_KEY_ID: "notchecked"
      FILEUPLOAD_S3_AWS_SECRET_ACCESS_KEY: "notchecked"
      FILEUPLOAD_S3_ENDPOINT: "http://graphcool-aws-services:4572"
      FILEUPLOAD_S3_BUCKET: "files.graph.cool"
      FILEUPLOAD_AWS_REGION: "local"
      REDIS_HOST: "graphcool-redis-host"
      REDIS_PORT: "6379"
      KINESIS_ENDPOINT: "http://graphcool-aws-services:4568"
      KINESIS_STREAM_API_METRICS: "graphcool-aws-services"
      KINESIS_STREAM_ALGOLIA_SYNC_QUERY: "graphcool-aws-services"
      AWS_CBOR_DISABLE: "true"
      CLOUDWATCH_ENDPOINT: "http://graphcool-aws-services:4582"
      BUGSNAG_API_KEY: ""
      AWS_ACCESS_KEY_ID: "notchecked"
      AWS_SECRET_ACCESS_KEY: "notchecked"
      SNS_ENDPOINT_SYSTEM: "http://graphcool-aws-services:4572"
      SNS_FUNCTION_LOGS: ""
      AUTH0_DOMAIN: ""
      AUTH0_CLIENT_SECRET: ""
      AUTH0_API_TOKEN: ""
      DATA_EXPORT_S3_ENDPOINT: "http://graphcool-aws-services:4572"
      DATA_EXPORT_S3_ENDPOINT: "http://graphcool-aws-services:4572"
      DATA_EXPORT_S3_BUCKET: "graphcool-data-export"
      STRIPE_API_KEY: ""
      INITIAL_PRICING_PLAN: "2017-02-free"
      SNS_SEAT: "arn:aws:sns:local:123456789012:crm-loal-Infrastructure-sns-collaborator-signup"
      SNS_ENDPOINT: "http://graphcool-aws-services:4575"

networks:
  proxy:
    external: true
  internal:
    external: false
```

## Start Graphcool
Run in aws-box terminal:
```bash
sudo docker-compose up -d
sudo docker-compose logs -f
```
...and wait until the log tells you, your docker containers are up and running.

## Prepare Project
Exit from aws-box terminal and go to the terminal of your pc/notebook.
Dowload full-example from graphcool/framework on github.
Find latest instruction here: https://github.com/graphcool/framework/tree/master/examples/full-example
Now you need your stripe test secret key from above:
```
export STRIPE_TEST_KEY="sk_test..."
curl https://codeload.github.com/graphcool/framework/tar.gz/master | tar -xz --strip=2 framework-master/examples/full-example
cd full-example
npm install -g graphcool
npm install
```

Before we can deploy our project, we first need to prepare our local environment, that it is pointing to our new aws-box graphcool and localfaas services.

We need to have a global graphcoolrc file to add our box. Please go to your browser and go to https://www.graph.cool/ and Open Console and log into or sign-up if you haven't. After that, you should have a global ~/.graphcoolrc file with your account credentials for the central graphcool service on your pc/notebook.

Backup this file for security reasons, as we are going to change it now.
```bash
cp ~/.graphcoolrc ~/.graphcoolrc.backup
touch getToken.sh
chmod +x getToken.sh
```

But before we change this file, we first need a token, to access our aws-box without direct authentication:

Add the following lines to your getToken.sh file. 
Replace the string example.com with your domain your graphcool service is listening on.
Replace the string MASTER_TOKEN with your new MASTER_TOKEN out of your docker-compose.yml environment.
```bash
#!/bin/bash
curl 'https://graphcool.aws-box.example.com/system' -H 'Content-Type: application/json' -d '{"query":"mutation {authenticateCustomer(input:{auth0IdToken:\"MASTER_TOKEN\"}){token, user{id}}}"}' -sS
```
Save the file and run it with
```bash
./getToken.sh
```
If everything works, it'll output your new secret token to access your graphcool service in your aws-box. Copy only the token to your clipboard, we will add it now to your global .graphcoolrc file.

Or you can also execute the above mutation query in your browser url https://graphcool.example.com/system playground directly, to obtain your token.

Now, open ~/.graphcoolrc in your editor. Adjust it like so, and replace the example.com with your domain and replace the string YOUR-TOKEN-FROM-YOUR-CLIPBOARD with your new token. Please don't touch your platform token. This is for the graph.cool PAAS service.
```bash
clusters:
  default: shared-eu-west-1
  local:
    host: 'https://graphcool.aws-box.example.com'
    faasHost: 'https://graphcool.aws-box.example.com'
    clusterSecret: >-
     YOUR-TOKEN-FROM-YOUR-CLIPBOARD
platformToken: >-
  DON'T-TOUCH-THE-PLATFORM-TOKEN-HERE

```
Now you are ready to deploy to your new graphcool backend.

## Deploy Project
Run in your full-example folder:
```bash
graphcool deploy
```
Choose "local" deployment, "PROD" and "full-example" as input parameters.

It should deploy to your new aws-box backend.

If you get an error, switch to debug mode by setting the environment variable:
```bash
export DEBUG="*"
graphcool deploy
```

Now you should get much more information in which step you get an error.
You can also go to your aws-box and
```bash
cd /opt/graphcool
sudo docker-compose logs -f
```
to see, if your backend containers receive your calls from your frontend.

HINT: Take care about the authorizations of your database user.
It must have authorizations for the following databases:
* graphcool
* log
* cj*

With every deployment graphcool creates a project starting with cj*. During deployment it generates a new database called cj*. Your database user must have authorization on these also.

If it doesn't work, chances are high, that the error lies in one of your graphcool environment parameters in your docker-compose.yml file. Good luck!

## Test Run Project
Now run:
```bash
graphcool root-token seed-script
```
To get a new token for the seed-script. Copy this token into your clipboard.
Replace __ENDPOINT__ and __ROOT_TOKEN__ with the two copied values, and run the script to create a few product items:
```bash
node src/scripts/seed.js
```

It should have created a new product (iphone) in your full-example project.

## Shopping Workflow
* test a simple query
* signup as a new user
* login as a known user
* set authorization token and test token login
* create a new order
* add a couple of items
* pay the order

Get into your playground with:
```bash
graphcool playground
```
It should open a browser and start the GraphQL playground of your full-example shop project.
For example:
https://graphcool.aws-box.example.com/simple/v1/cj*

### Test a simple query
Execute the following query:
```graphql
{
  allProducts {
    id
    name
    description
    price
  }
}
```
You should see something like:
```graphql
{
  "data": {
    "allProducts": [
      {
        "id": "cjproductid",
        "name": "iPhone X",
        "description": "The new shiny iPhone",
        "price": 1200
      }
    ]
  }
}
```

### Signup as a new user
Execute the following mutation query:
```graphql
mutation signup {
  signupUser(
    email: "me@example.com"
    password: "password"
  ) {
    id
    token
  }
}
```
You should get your Authorization token for your next calls:
```graphql
{
  "data": {
    "signupUser": {
      "id": "cjuserid",
      "token": "long-token-string-with-many-characters-and-numbers"
    }
  }
}
```

### Login as a known user
Execute the following mutation query:
```graphql
mutation login {
  authenticateUser(
    email: "me@example.com"
    password: "my-secret-password"
  ) {
    token
  }
}
```
You should get your Authorization token for your next calls:
```graphql
{
  "data": {
    "authenticateUser": {
      "token": "long-token-string-with-many-characters-and-numbers"
    }
  }
}
```
### Set authorization token and test token login
In the GraphQL Playground, click at the bottom on: ```HTTP Headers``` and create a new Header entry with the key and value:

```bash
Authorization Bearer long-token-string-with-many-characters-and-numbers
```
While "Authorization" is the key, and the value is "Bearer long-token-string-with-many-characters-and-numbers". Please take care, that there is a space between Bearer and the login token. Don't use the quotes.

Use this header for all authenticated requests in future.

2. Execute the following query:
```graphql
{
  loggedInUser {
    id
  }
}
```
You should get your User id:
```graphql
{
  "data": {
    "loggedInUser": {
      "id": "cjuserid"
    }
  }
}
```

### Create a new order
Take care, that the Authorization header is set.
In the GraphQL Playground at the bottom, click onto "QUERY VARIABLES" to open the drawer for variable values. Add the variable in this drawer in json syntax:
```json
{"userId": "cjuserid"}
```
Execute the following mutation query (with Authorization header):
```graphql
mutation beginCheckout($userId: ID!) {
  createOrder(
    description: "Christmas Presents 2017"
    userId: $userId
  ) {
    id
  }
}
```
You'll get:
```graphql
{
  "data": {
    "createOrder": {
      "id": "cjorderid"
    }
  }
}
```

### Add an item to the order
Add variables:
```json
{ 
  "orderId": "cjorderid",
  "productId": "cjproductid",
  "amount": 1
}
```
Execute the query:
```graphql
mutation addOrderItem($orderId: ID!, $productId: ID!, $amount: Int!) {
  setOrderItem(
    orderId: $orderId
    productId: $productId
    amount: $amount
  ) {
    itemId
    amount
  }
}
```
You will get something like:
```graphql
{
  "data": {
    "setOrderItem": {
      "itemId": "cjitemid",
      "amount": 1
    }
  }
}
```

### Pay the order
Before we can pay, we need a one-time-payment token from stripe. Because we will need it more often, we create a shell script therefore:
```bash
touch getStripePaymentToken.sh
chmod +x getStripePaymentToken.sh
```
Add the following lines to the file getStripePaymentToken.sh. Replace the sk_test_keyasdfsdafsdaf with your own Stripe test secret API key from stripe.com. (Don't forget to switch to test data!
```bash
#!/bin/bash
curl https://api.stripe.com/v1/tokens \
   -u sk_test_keyasdfsdafsdaf: \
   -d card[number]=4242424242424242 \
   -d card[exp_month]=12 \
   -d card[exp_year]=2018 \
   -d card[cvc]=123
```
Run
```bash
./getStripePaymentToken.sh
```
You will get as output a new payment token: "tok_paymentToken"

Add the following variables and use your order id from earlier and your just generated one-time payment token.
```json
{ 
  "orderId": "cjorderid", 
  "stripeToken": "tok_paymentToken"
}
```
Execute the following mutation query:
```graphql
mutation pay($orderId: String!, $stripeToken: String!) {
  pay(
    orderId: $orderId
    stripeToken: $stripeToken
  ) {
    success
  }
}
```
You will get hopefully the following json as result:
```
{
  "data": {
    "pay": {
      "success": true
    }
  }
}
```

With these backend queries and mutations, you are able to develop a frontend app and run a simple webshop.
