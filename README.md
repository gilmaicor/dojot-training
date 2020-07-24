# dojot-training

## About
This repository is the workspace for the dojot's training.

Here, you'll find instructions and recipes to support you.

> This documentation contains some tags that are replaced when you run the setup.sh to set your workspace. So, after running this script, don't forget to reload the documentation.

## Table of Contents

* [dojot-training](#dojot-training)
  * [About](#about)
  * [Table of Contents](#table-of-contents)
  * [Prerequisites](#prerequisites)
  * [Setting up your Ubuntu machine](#setting-up-your-ubuntu-machine)
      * [Docker](#docker)
      * [Allowing an insecure registry](#allowing-an-insecure-registry)
      * [Docker Compose](#docker-compose)
      * [Git](#git)
      * [JQ](#jq)
      * [MQTT Client](#mqtt-client)
      * [HTTP Client](#http-client)
      * [Javascript Editor](#javascript-editor)
  * [<strong>Hands-on</strong>](#hands-on)
  * [Use Cases](#use-cases)
      * [Cold-chain monitoring](#cold-chain-monitoring)
      * [Water quality monitoring](#water-quality-monitoring)
  * [Tasks](#tasks)
      * [Task 1: Start dojot's microservices for the cold-chain monitoring use case](#task-1-start-dojots-microservices-for-the-cold-chain-monitoring-use-case)
      * [Task 2: Configure and simulate the cold-chain monitoring use case](#task-2-configure-and-simulate-the-cold-chain-monitoring-use-case)
      * [Task 3: Develop an iot-agent for the water quality monitoring use case](#task-3-develop-an-iot-agent-for-the-water-quality-monitoring-use-case)
        * [Step 1: Start a dummy iotagent-http](#step-1-start-a-dummy-iotagent-http)
        * [Step 2: Implement your iotagent-http for the water quality monitoring use case](#step-2-implement-your-iotagent-http-for-the-water-quality-monitoring-use-case)
        * [Step 3: Test your iotagent-http](#step-3-test-your-iotagent-http)
      * [Task 4: Develop a function node for the water quality monitoring use case](#task-4-develop-a-function-node-for-the-water-quality-monitoring-use-case)
        * [Step 1: Load a stub of the decoder node into flowbroker](#step-1-load-a-stub-of-the-decoder-node-into-flowbroker)
        * [Step 2: Implement the decoder logic](#step-2-implement-the-decoder-logic)
        * [Step 3: Test your decoder node](#step-3-test-your-decoder-node)

## Prerequisites

To do the tasks, you will need:

- A machine with Ubuntu 18.04 with at least 4GB RAM

- User with sudo permissions

- Connection with the Docker Hub or that contains dojot's docker images.

- Docker > 17.12

- Docker Compose > 1.18

- Git

- HTTP Client

- JavaScript Editor

## Setting up your Ubuntu machine

### Git

To install git:

``` sh
sudo apt-get install git
```

### HTTP Client

Our suggestion is to use curl, but if you are familiar with other tools like postman, feel free to use them. To install curl:

``` sh
sudo apt-get install curl
```

### Javascript Editor

Some of the hands-on will require to develop Javascript code. Our suggestion is to use Visual Studio Code, but feel free to choose a code editor of your preference.

Instructions to install vs code can be found at [VSCode Installation](https://code.visualstudio.com/docs/setup/linux). Basically, you need to run:

``` sh
curl https://packages.microsoft.com/keys/microsoft.asc | gpg --dearmor > microsoft.gpg
sudo install -o root -g root -m 644 microsoft.gpg /etc/apt/trusted.gpg.d/
sudo sh -c 'echo "deb [arch=amd64] https://packages.microsoft.com/repos/vscode stable main" > /etc/apt/sources.list.d/vscode.list'

sudo apt-get install apt-transport-https
sudo apt-get update
sudo apt-get install code # or code-insiders
```

## **Hands-on**

## Use Cases

### Water quality monitoring

A company produces an IoT device with pollutant and oxygenation level detectors. It will install several of them along a river near a factory to assess the water treatment. Due to the simplicity of the device, it sends the data corresponding to the two measurements (pollutant and oxygenation) in a 16-bit message. Each value is encoded in 8 bits. The data is sent hourly to a gateway that relays it to dojot via HTTP protocol.

A sample message is given bellow:

```
POST /chemsen/readings
{
  "timestamp": 1543449992,            - unix timestamp: 11/29/2018 - 12:06am
   "data": 656,                       - pollutant = 00000010b = 2 | oxygenation = 10010000b = 144
   "device": "PINHR_003"              - unique device identifier
}
```

## Tasks

### Task 1: Start dojot's microservices for the cold-chain monitoring use case

First of all, you need clone the docker-compose repository:

``` sh
git https://github.com/dojot/docker-compose
git checkout v0.4.2 -b v0.4.2
cd -
```

### Task 3: Develop an iot-agent for the water quality monitoring use case

The goal of doing this task is to learn how to develop an iot-agent microservice. This is REQUIRED to integrate a device not supported by dojot.

Before starting, go to https://github.com/dojot/iotagent-nodejs and take a look at the documentation.

If you dont't know what to do, don't panic; just follow the steps listed bellow.

#### Step 1: Start a dummy iotagent-http

There is a dummy iotagent-http at samples/iotagent-base. There, you will find:

``` sh
.
├── Dockerfile        - file to build the docker container
├── package.json      - javascript dependencies
├── README.md         - README file
└── src
    └── index.js       - the microservice code

```

First, let's start this microservice:

``` sh
cd samples/iotagent-base
sudo docker build -t dojot-training/iotagent-http .
cd -
```

Then you need to add the text above to docker-compose.yml, to include this new microservice, and
start it:

iotagent-http:
    image: dojot-training/iotagent-http
    depends_on:
      - kafka
      - data-broker
      - auth
    ports:
      - 3124:3124
    restart: always
    environment:
      SERVER_PORT: 3124
    logging:
      driver: json-file
      options:
        max-size: 100m



``` sh
cd docker-compose
pico docker-compose.yaml
sudo docker-compose up --remove-orphans -d
cd -
```

Wait some seconds and check its log:

``` sh
cd docker-compose
sudo docker-compose logs -f iotagent-http
cd -
```

You should see some messages that the microservice is running.

To send a HTTP message to this agent, run:

``` sh
DOJOT_HOST="127.0.0.1"
IOTAGENT_PORT=3124
curl -X POST ${DOJOT_HOST}:${IOTAGENT_PORT}/test/data \
-H 'Content-Type:application/json' \
-d '{ "timestamp": 1543449992, "data": 646, "device": "PINHR_003" }'
```

if you check the logs again, you should see that the message has been received.

#### Step 2: Implement your iotagent-http for the water quality monitoring use case

Now, you need to customize the dummy iotagent-http for handling the use case.
Openning the file iotagent-http/src/index.js, you  will see a list of TODOs.
You just need to implement them.

Once you've finished, you need to rebuild and restart the service:

``` sh
cd samples/iotagent-base
sudo docker build -t dojot-training/iotagent-http .
cd -
cd docker-compose
sudo docker-compose up -d
cd -
```

Check the logs to see if it's running.

#### Step 3: Test your iotagent-http

Create the template and devices for the water quality monitoring use case. Then, generate some
HTTP messages and validate if they are associated with the corresponding devices.

### Task 4: Develop a function node for the water quality monitoring use case

The goal here is to learn how to develop function nodes to extend the flowbroker microservice.

You can find instructions how to do this from scratch at https://github.com/dojot/flowbroker/tree/master/lib. So, go there and take a look.

Try to develop the decoder function by yourself. If you get stuck, follow the steps bellow.

#### Step 1: Load a stub of the decoder node into flowbroker

There is a stub for this microservice at samples/decoder-node-base. There, you will find:

``` sh
.
├── Dockerfile              - file to build the docker container
├── package.json            - javascript dependencies
├── README.md               - README file
└── src
    ├── decoder-node.html   - the node html
    └── index.js            - the decoder logic
```

First, build the container:

``` sh
cd samples/decoder-node-base
sudo docker build -t <flowbroker-node-docker-registry>dojot-training/decoder-node<unique-id> .
cd -
```

Then, push it to the docker registry:

``` sh
cd samples/decoder-node-base
sudo docker push <flowbroker-node-docker-registry>dojot-training/decoder-node<unique-id>
cd -
```

Now, load the node into the flowbroker:

```sh
DOJOT_HOST="http://localhost:8000"
JWT=$(curl -s -X POST ${DOJOT_HOST}/auth -H 'Content-Type:application/json' -d '{"username": "admin", "passwd" : "admin"}' | jq -r ".jwt")
curl -X POST -H "Authorization: Bearer ${JWT}" ${DOJOT_HOST}/flows/v1/node -H "content-type: application/json" -d '{"image": "<flowbroker-node-docker-registry>dojot-training/decoder-node<unique-id>", "id":"decoder-node"}'
```
Make sure you have the `jq` installed on your system, otherwise set the `JWT` variable manually.

If you want to remove the node, run:

```sh
DOJOT_HOST="http://localhost:8000"
JWT=$(curl -s -X POST ${DOJOT_HOST}/auth -H 'Content-Type:application/json' -d '{"username": "admin", "passwd" : "admin"}' | jq -r ".jwt")
curl -X DELETE -H "Authorization: Bearer ${JWT}" ${DOJOT_HOST}/flows/v1/node/decoder-node
```
Now, you should be able to access the decoder node at the dojot's gui.

#### Step 2: Implement the decoder logic

Openning the file decoder-node-base/src/index.js, you  will see a TODO.
You just need to implement it.

Once you've finished, you need to rebuild the container and push it to the registry. 
So, run:

``` sh
cd samples/decoder-node-base
sudo docker build -t <flowbroker-node-docker-registry>dojot-training/decoder-node<unique-id> .
sudo docker push <flowbroker-node-docker-registry>dojot-training/decoder-node<unique-id>
cd -
```

Now, reloads the node into the flowbroker:

``` sh
DOJOT_HOST="http://localhost:8000"
JWT=$(curl -s -X POST ${DOJOT_HOST}/auth -H 'Content-Type:application/json' -d '{"username": "admin", "passwd" : "admin"}' | jq -r ".jwt")
curl -X DELETE -H "Authorization: Bearer ${JWT}" ${DOJOT_HOST}/flows/v1/node/decoder-node
curl -X POST -H "Authorization: Bearer ${JWT}" ${DOJOT_HOST}/flows/v1/node -H "content-type: application/json" -d '{"image": "<flowbroker-node-docker-registry>dojot-training/decoder-node<unique-id>", "id":"decoder-node"}'
```

#### Step 3: Test your decoder node

Create virtual devices to receive the decoded values and a processing flow for executing the decoder. Then, generate some HTTP messages for the devices and look at the virtual ones to see if the values appeared decoded.
