# echo-sonos-apple-music
All of the pieces for an Amazon Echo (Alexa) <-> Sonos Apple Music integration. This code is live in my home with a high Wife Acceptance Factor (WAF).

# Usage

Global commands (no rooms required):

* Artists: "Alexa, tell Sonos to play Grimes in the Office"
* Artists: "Alexa, tell Sonos to play Grimes radio in the Office"
* Pause all: "Alexa, tell Sonos to pause all"
* Resume all: "Alexa, tell Sonos to resume all"

Room-specific commands, where "ROOM" could be any of your Sonos room names (eg Kitchen, Master Bedroom, and so on):

* Next: "Alexa, tell Sonos go to the next track in the ROOM"
* Previous: "Alexa, tell Sonos to go back in the ROOM"
* What's playing: "Alexa, ask Sonos what's playing in the ROOM"
* Pause: "Alexa, tell Sonos to pause in the ROOM"
* Resume: "Alexa, tell Sonos to resume in the ROOM"
* Mute: "Alexa, tell Sonos to mute in the ROOM"
* Unmute: "Alexa, tell Sonos to unmute in the ROOM"
* Repeat: "Alexa, tell Sonos to turn repeat [on,off] in the ROOM"
* Shuffle: "Alexa, tell Sonos to turn shuffle [on,off] in the ROOM"
* Crossfade: "Alexa, tell Sonos to turn crossfade [on,off] in the ROOM"
* Clear queue: "Alexa, tell Sonos to clear the queue in the ROOM"
* Volume up or down (single room): "Alexa, tell Sonos to turn it [up,down] in the ROOM"
* Volume up or down (all in group): "Alexa, tell Sonos to turn it [up,down] in the ROOM group"
* Set volume (single room): "Alexa, tell Sonos to change the volume to 22 in the ROOM"
* Set volume (all in group): "Alexa, tell Sonos to change the volume to 22 in the ROOM group"
* Add room to the group: "Alexa, tell Sonos to join NEW_ROOM to the ROOM"
* Remove room from the group: "Alexa, tell Sonos to ungroup ROOM"
* Many other natural phrasings are supported for each command. The file "echo/utterances.txt" has all of the examples.

Everything's dynamic - rooms and playlists are taken dynamically from your speech. There are common room names in utterances.txt to help the Alexa engine recognize valid entries, but it's not necessary to add more. 

The service is also smart enough to control your whole group when given only a room name, even if that room isn't the Sonos coordinator, so you can change the volume in an entire group without having to remember which speaker is the coordinator.

# How it works

1. When you say the command to Alexa, it triggers the Alexa skill with invocation name Sonos. 
2. The Alexa skill calls a web service running on AWS Lambda, passing it the channel name ("rock" in the example). AWS Lambda is either free or ridiculously cheap (think 10 cents a month. If you can't afford Lambda how did you afford a Sonos?).
3. Lambda then fires an HTTP request to a node.js server running node-sonos-http-api on your local network. 
4. node-sonos-http-api interprets the command and relays to Sonos over your local network.
5. If you're using an OSX/MacOS server in your house, check the servers folder for com.sonos.server.plist which is a Launchctl  plist - you will want to:
```
ln -ls /usr/local/lib/node_modules/sonos-http-api/server.js /usr/local/bin/sonos
```
You'll need to add the following as the first line of that file so it can be executed without calling node:
```
#!/usr/bin/env node
``` 
Included here are the Alexa API definitions, the Lambda AWS service that catches the Alexa requests, and an example preset configuration for jishi's node-sonos-http-api to actually play the music.

To set it up, you need to do the following:

# Get jishi's node-sonos-http-api working
1. Install node.js on a server on the same network as your Sonos. 
2. Grab https://github.com/jishi/node-sonos-http-api and run it on that server.  On Mac, it's "npm install https://github.com/jishi/node-sonos-http-api", then go to the directory created and "npm start".
3. Test it by hitting http://yourserverip:5005/zones
4. If you get a response, great! Now try playing a channel based radio station: http://yourserverip:5005/Office/applemusic/radio/radio:[your_channel_name]. Or, play an artists music or Favorite (example: http://yourserverip:5005/applemusic/queue/name:U2). To stop, use /pauseall.
5. If you have problems, make sure you are on the same network as your Sonos AND make sure you don't have a Sonos client running on the same machine. The client can interfere with the node.js server.

# Expose your server to the outside world
1. You need some way for Lambda to contact your server consistently. Services like DynDns and yDNS.eu will give you a consistent hostname for the outside world to use.  If you have an Asus router like I do, then dynamic DNS is actually a built-in / free feature.
2. On your local router, find the "Port Forwarding" configuration. Forward all inbound requests to 5005 (or configure some other port) to your node server.
3. Make sure your server has a locally static IP, so port forwarding doesn't lose track of it.
4. Setup your server to auto-start or daemonize the node-sonos-http-api server.
5. Test it by hitting http://yourdyndnsaddress:5005/zones.

# Create the Alexa Skill that will send events to AWS Lambda
1. Create a new Skill in the Alexa Skills control panel on Amazon. You need a developer account to do this.
2. Name can be whatever you want. "Invocation" is what you say (I used "Sonos").
3. Check Custom Interaction Model if it is not already checked. Click Next
4. Click Next, taking you to Interaction Model. Create a Custom Slot Type ("Add Slot Type"). Add a new type for PRESETS, another for ROOMS, and a final one for TOGGLES. Into each, copy/paste the contents of echo/custom_slots/PRESETS.slot.txt, echo/custom_slots/CHANNEL.slot.txt, echo/custom_slots/ROOMS.slot.txt and echo/custom_slots/TOGGLES.slot.txt
5. Still in Interaction Model, copy this repo's echo/intents.json into the "Intent Schema" field, and echo/utterances.txt into "Sample Utterances".
6. Don't test yet, just save. Click back to "Skill Information" and copy the "Application ID". You'll need this for Lambda.

# Configure the AWS Lambda service that will trigger your node-sonos-http-api server
1. Create an AWS Lambda account if you don't have one already. It's free!
2. In the Lambda console, look to the upper right. Make sure "N. Virginia" is selected, because not every zone supports Alexa yet.
3. Create a new Lambda function. Skip the blueprint. 
4. Pick any name you want, and choose runtime Node.js.
5. Go into this repo's [lambda/src](lamda/src) directory and copy options.example.js to options.js. Edit options.js to have your DynDNS hostname, your port, and the Alexa App ID you just copied.
6. In lambda/src, zip up everything. On Mac/Linux, `cd src; chmod a+r *.js; zip index.zip index.js`.  Make sure you don't capture the folder, just the files.
7. Choose to upload the zip file for src.zip.
8. The default handler is fine. Create a new role of type Basic Execution Role. Pick smallest possible memory and so on.
9. Click Next to proceed. Once created, click "Event Sources".
10. Add a source.  Choose "Alexa Skills Kit".
11. Test it out. I included a test blueprint in this repo. Click "Test" and choose an Alexa Services test. You should see some JSON in the console containing "What should I tell sonos to do?"

# Connect Alexa Skill to AWS Lambda
1. In the Lambda console, copy the long "ARN" string in the upper right.  
2. Go back into the Alexa Skill console, open your skill, click "Skill Information", choose Lambda ARN and paste that ARN string in.
3. Now you're ready to put it all together. Try "Alexa, use Sonos to play test"

# Optional Security Features
echo-sonos supports both HTTPS and basic auth, for those concerned with opening their Sonos server to the Internet.  options.example.js has flags and configuration information for both features. For HTTPS support, set "useHttps" to "true" and check to make sure the port is still correct.  For basic auth, change the "auth" variable and replace the username and password with your own.

## Configuring node-sonos-http-api
Securing node-sonos-http-api, HTTPS, and your home server are outside the bounds of this project. The below documentation is provided for convenience because so many people have asked me about it.  You could certainly do much more to secure your home server.  For starters, you could pin certificates to the client or put more effort behind secure key and credential storage.  This is a DIY hobbyist project, and it's up to your discretion to determine how much effort to put into security.

Both HTTPS and basic auth need to be configured on node-sonos-http-api before echo-sonos can use them.  You can configure one without the other, but they are recommended together.  In your node-sonos-http-api directory, create a file called settings.json according to the node-sonos-http-api documentation.  

I HIGHLY recommend you password protect and use SSL or you run the very real risk of some random person port scanning your network and having fun playing music in your house.

An example might look like this:
```
    {
        "port": 5005,
        "securePort": 5006,

        "https": {
            "key": "/path_to/server.key",
            "cert": "/path_to/server.crt"
        },

        "auth": {
            "username": "MAKE_UP_A_USERNAME",
            "password": "MAKE_UP_A_PASSWORD"
        }
    }
```
## Certificate creation
In the above example, HTTPS is configured using "yourserver.key" and "yourserver.crt".  You can buy a certificate but it's quite honestly just as secure to generate your own free ones. On Mac OSX just copy and paste the following into a terminal window. When the openssl req command asks for a “challenge password”, just press return, leaving the password empty.

```
openssl genrsa -des3 -passout pass:x -out server.pass.key 2048
openssl rsa -passin pass:x -in server.pass.key -out server.key
rm -f server.pass.key
openssl req -new -key server.key -out server.csr
```

At this point you need to answer some questions on the interactive command line. Most important is the "common name". If you used Dynamic DNS your hostname may be something like "myhomenetwork.dyndns.org". That hostname without the quotes is your common name.

Then paste:
```
openssl x509 -req -sha256 -days 3650 -in server.csr -signkey server.key -out server.crt
```

This creates a key and certificate that will last for 10 years. I'm guessing your server won't last that long.

# Troubleshooting
1. If you have trouble with your node server not triggering the music even when you hit it on localhost, it probably can't find Sonos. If it crashes with a message about "discovery" being "null" then that's definitely the case. Usually you're on the wrong wifi network, you forgot to close your Sonos application (which screws everything up), or your server died for some reason and you didn't notice.
2. If your Lambda test doesn't work, then you might have a case mis-match between the preset name in presets.json and the value in the Lambda test. It's also possible Lambda can't reach your host because your DynDNS setup isn't working, or a firewall is blocking it. If you're unsure, try the Maker plugin on IFTTT, to see if you can get it working externally from someplace else.
3. If Alexa says something about not being able to use your skill, then Lambda is probably returning an error. Check the Lambda logs. Alexa will say this if she doesn't see a proper response from the Lambda server.
4. If you are asking it to play something and Alexa keeps getting it wrong, enter the utterance into the CHANNEL slot. Alexa voice recognition is exceptional but at times it may mess up names.

# Contributing & Support
This repository comes with absolutely NO support.

Fork it, steal it, repurpose it as commercial code, just don't ask for my help :) 

This originally came from the echo-sonos project but it was largely focused on the presets and accessing Sonos favorites which I found to be annoying. In theory you can hack this code to work for Pandora or Spotify instead of Apple Music by just changing a few lines.
