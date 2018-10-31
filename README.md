# Ionosphere
**This is a third-party effort, and is NOT a part of the official [Signal](https://signal.org) project or any other project of [Open Whisper Systems](https://whispersystems.org).**

**WARNING: This adapter is currently undergoing a security audit. It should not be regarded as secure, use at your own risk. I am not a cryptographer, I just encrypt a lot.**

A sample project using Hubot to bounce Signal messages around. This repository includes instructions and some configuration for deploying a chatbot that connects to the Signal messaging service. As an example of what is possible with such a tool, the bot includes functionality for creating simple notification lists that can be used to send broadcast messages. Directions are provided for installing the bot manually, using [Docker](https://docker.com), or using [Heroku](https://heroku.com).

## Components
This project uses [Hubot](https://hubot.github.com) and a few extensions for it:
* **[libsignal-service-javascript](https://github.com/throneless-tech/libsignal-service-javascript)** A standalone port of the "libtextsecure" library components of the [Signal-Desktop](https://github.com/WhisperSystems/Signal-Desktop) client. It can be found on NPM [here](https://npmjs.com/package/@throneless/libsignal-service).
* **[hubot-signal-service](https://github.com/throneless-tech/hubot-signal)**, a Hubot adapter for connecting to the Signal network that utilizes the library above. It can be found on NPM [here](https://www.npmjs.com/package/hubot-signal-service).
* **[hubot-list](https://github.com/throneless-tech/hubot-list)**, a simple Hubot script for providing a basic distribution list functionality. It can be found on NPM [here](https://npmjs.com/package/@throneless/hubot-list).

Additionally, the Docker installation uses the following custom container:
* **[chambana/hubot](https://github.com/chambana-net/docker-hubot)**, which provides automatic installation and configuration of a modern version of Hubot. It's available on the Docker Hub [here](https://hub.docker.com/r/chambana/hubot/).

## Installation
In order to create a functioning bot that connects to the Signal network, it is necessary to have a phone number associated with the bot for authentication purposes. This can be a real phone, a [Google Voice](https://voice.google.com) number, a [Twilio](https://twilio.com) number, or any other number where you can receive an SMS message. It is not currently recommended to associate the bot with a number that you also intend for normal Signal use, as you will most likely clobber your keys. Technically after you authenticate the bot you no longer need the number, but you may want to retain it so that someone else can't start using it.
### Standalone
These instructions were tested using [NodeJS](https://nodejs.org) v8.12.0. Some of the components are only compatible with Hubot v3.x and above. Hubot uses a persistant storage backend called the "Brain." For many adapters, having a persistent brain is not necessary, but for hubot-signal-service it is mandatory so that it can store keying information. Any available hubot-brain should work; the most commonly used is [hubot-redis-brain](https://www.npmjs.com/package/hubot-redis-brain), but installing standalone [Redis](https://redis.io) is outside the scope of these instructions.
1. The recommended way to install Hubot is to use [generator-hubot](https://www.npmjs.com/package/generator-hubot), a [Yeoman](https://yeoman.io) generator. As of 31-Oct-2018, the `latest` tag has not been updated to point at a version that installs Hubot v3.x or above, so it is necessary to install it with an explicit version:
`npm install -g yo generator-hubot@1.1.0`
2. Create a directory for the bot, and then run generator inside and answer the prompts. When it asks you for the adapter, enter `signal-service`:
```
mkdir hubot
cd hubot
yo hubot
```
3. Edit the `external-scripts.json` file to include hubot-list and any other scripts you would like to include. Do **not** add hubot-signal-service to this file.
```
[
  "hubot-diagnostics",
  "hubot-help",
  "hubot-redis-brain",
  "@throneless/hubot-list"
]
```
4. Hubot is primarily configured through environment variables. With the hubot-signal-service adapter and the `external-scripts.json` above, you will need to specify the following environment variables:
* `HUBOT_SIGNAL_NUMBER`: This is the phone number that this Hubot instance will listen on in [E.164 format](https://en.wikipedia.org/wiki/E.164), i.e. a US phone number would look like `+15555555555`.
* `HUBOT_SIGNAL_PASSWORD`: This is an arbitrary password string, but it must be the same between initializations.
* `HUBOT_LIST_ADMINS`: This is a comma-separate list of phone numbers that are allowed to create, manage, and send to the distribution lists provided by hubot-list.
* `REDIS_URL`: A URL for connecting to your Redis instance. See [here](https://www.npmjs.com/package/hubot-redis-brain) for valid values for `REDIS_URL`.
**NOTE:** Additionally, hubot-signal-service connects to the Signal staging server rather than the production server by default. This is suitable for testing, but you won't be able to send to or receive messages from the bot using standard clients. To connect to the live server, you must also pass the variable `NODE_ENV=production`.
`NODE_ENV=production HUBOT_SIGNAL_NUMBER=+15555555555 HUBOT_SIGNAL_PASSWORD=<password> HUBOT_LIST_ADMINS=+15555556666 node ./bin/hubot -a signal-service`
5. When you first start the bot with the above parameters, it will have the Signal server send you a numeric code via SMS at the number you specified, and then exit. On the next startup, in addition to the above parameters:
* `HUBOT_SIGNAL_CODE`: This is the code you receive via SMS at the number above (without the dash/hyphen). When you start up the bot for the second time with this code, it will authenticate that you have control over the number and Hubot will respond to Signal messages sent to the number above. The `HUBOT_SIGNAL_CODE` parameter will be ignored on subsequent startups and can be ommitted thereafter.
`NODE_ENV=production HUBOT_SIGNAL_NUMBER=+15555555555 HUBOT_SIGNAL_PASSWORD=<password> HUBOT_SIGNAL_CODE=<code> HUBOT_LIST_ADMINS=+15555556666 node ./bin/hubot -a signal-service`
### Docker
1. This repository provides a sample `docker-compose.yml` file for use with `docker-compose`. Install `docker-compose`, and then create a `.env` file in the same directory with the following configuration options, one per line in `KEY=VALUE` form:
* `HUBOT_SIGNAL_NUMBER`: This is the phone number that this Hubot instance will listen on in [E.164 format](https://en.wikipedia.org/wiki/E.164), i.e. a US phone number would look like `+15555555555`.
* `HUBOT_SIGNAL_PASSWORD`: This is an arbitrary password string, but it must be the same between initializations.
* `HUBOT_LIST_ADMINS`: This is a comma-separate list of phone numbers that are allowed to create, manage, and send to the distribution lists provided by hubot-list.
* `HUBOT_ADAPTER`: The adapter to use with Hubot. For the purposes of this tutorial, we're using `hubot-signal-service`.
* `HUBOT_EXTERNAL_SCRIPTS`: A comma-separated list of Hubot scripts to load with the bot. This should **not** include hubot-signal-service. For the purposes of this tutorial, we are using `hubot-help,hubot-diagnostics,hubot-redis-brain,@throneless/hubot-list`.
* `REDIS_URL`: A URL for connecting to your Redis instance. This Docker configuration includes a Redis container, so we can use `redis://redis:6379`.
* `HUBOT_NAME`: The name you would like for the bot. For example, `hubot` or `ionosphere`.
2. After filling out the `.env` file in the same directory as the provided `docker-compose.yml` file, run `docker-compose up` from the same directory. It will download and install the necessary Docker containers, and run them.
3. On first run, Hubot will exit with a message that it has sent an authentication code to the number you specified. If necessary hit `CTRL^c` to regain controll of the shell and add the variable `HUBOT_SIGNAL_CODE` to the `.env` file, set to the code you received via SMS (sans the hyphen). Run `docker-compose up` again, and it will reload Hubot and complete the registration process. If you wish, you may run `docker-compose up -d` instead to run it in the background.
