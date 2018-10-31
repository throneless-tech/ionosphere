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
**NOTE:** Additionally, hubot-signal-service connects to the Signal staging server rather than the production server by default. This is suitable for testing, but you won't be able to send to or receive messages from the bot using standard clients. To connect to the live server, you must also pass the variable `NODE_ENV=production`.
2. After filling out the `.env` file in the same directory as the provided `docker-compose.yml` file, run `docker-compose up` from the same directory. It will download and install the necessary Docker containers, and run them.
3. On first run, Hubot will exit with a message that it has sent an authentication code to the number you specified. If necessary hit `CTRL^c` to regain controll of the shell and add the variable `HUBOT_SIGNAL_CODE` to the `.env` file, set to the code you received via SMS (sans the hyphen). Run `docker-compose up` again, and it will reload Hubot and complete the registration process. If you wish, you may run `docker-compose up -d` instead to run it in the background.
### Heroku
This assumes that you have the Heroku CLI installed and that you have set up a Heroku account. **NOTE:** The Heroku free plan will not keep Hubot awake 24/7. You can partially work around that by including the hubot-heroku-keepalive script (instructions [here](https://github.com/hubot-scripts/hubot-heroku-keepalive)) or by upgrading to a paid tier.
1. Follow steps 1 - 3 of the standalone installation instructions above.
2. Follow the instructions here: https://hubot.github.com/docs/deploying/heroku/. When you get to the part about setting environment variables, instead of the examples given, set the following environment variables in Heroku:
* `HUBOT_SIGNAL_NUMBER`: This is the phone number that this Hubot instance will listen on in [E.164 format](https://en.wikipedia.org/wiki/E.164), i.e. a US phone number would look like `+15555555555`.
* `HUBOT_SIGNAL_PASSWORD`: This is an arbitrary password string, but it must be the same between initializations.
* `HUBOT_LIST_ADMINS`: This is a comma-separate list of phone numbers that are allowed to create, manage, and send to the distribution lists provided by hubot-list.
**NOTE:** Additionally, hubot-signal-service connects to the Signal staging server rather than the production server by default. This is suitable for testing, but you won't be able to send to or receive messages from the bot using standard clients. To connect to the live server, you must also pass the variable `NODE_ENV=production`.
3. Before pushing your repository to Heroku, install Redis into your Heroku instance:

`heroku addons:create rediscloud`

4. Continue with the tutorial and push the repository to Heroku. Hubot will exit at first, but will send an SMS message with an authentication code to the number you specified. Add that code (sans hyphen) to your Heroku environment:

`heroku config:set HUBOT_SIGNAL_CODE=<code>`

5. Hubot should pick up the code and complete registration with the Signal server on its next restart. If necessary, you can restart the dyno manually.

### Usage
From another Signal device, add the bot's number to your contacts. You may need to back out of and re-open Signal, but when you open a compose window to write a message to that contact it should have a blue 'send' button, indicating the messages are going over Signal's encrypted messaging service. The bot can be loaded with any available Hubot script, but the distribution lists outlined in this example project use the following commands (only available when requested via a number specified in `HUBOT_LIST_ADMINS` above):
```
list lists - list all list names
list dump - list all list names and members
list create <list> - create a new list
list destroy <list> - destroy a list
list rename <old> <new> - rename a list
list add <list> <number> - add a number to a list
list remove <list> <number> - remove a number from a list
list info <list> - list members in list
list membership <number> - list lists that name is in
```
**NOTE:** Hubot's text processing doesn't like `+` signs, so when adding a number you should use the same [E.164 format](https://en.wikipedia.org/wiki/E.164) as specified above, but omit the leading `+`.

To send a message to a list you've created, simply send a message to the bot as below:

`@<listname> <message>`

So for instance, in order to send the message "Hello world!" to the list "notifications", message the bot:

`@notifications Hello world!`

And every number you added to the notifications list will receive the message "Hello world!" coming from the bot's phone number.

## License
[<img src="https://www.gnu.org/graphics/agplv3-155x51.png" alt="AGPLv3" >](http://www.gnu.org/licenses/agpl-3.0.html)

Ionosphere is a free software project licensed under the GNU Affero General Public License v3.0 (AGPLv3) by [Throneless Tech](https://throneless.tech).
