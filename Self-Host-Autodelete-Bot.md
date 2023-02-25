If you run a Debian server and don't want to/can't setup Docker but want to run the AutoDelete bot, you can follow the instructions below that I just composed.

Save the following script to a new file on your server and run it (with sudo or root):

```
#!/bin/bash

# Change this if you want to install the bot in a different folder
INSTALL_DIR=/srv/autodelete

# This will be the linux user that will run the program.
# You shouldn't need to change this parameter, but if you want, you can.
LINUX_USER=autodeletebot

# Install the prerequisistes
# build-essential may be optional.
# We install nano so that most users will be able to edit the config.yml file.
apt update
apt install golang git build-essential nano

# Create our Linux user
useradd -r $LINUX_USER -s /bin/false

# Prepare the various folders
mkdir $INSTALL_DIR
mkdir $INSTALL_DIR/data
chown $LINUX_USER:root -R $INSTALL_DIR/data

# We also delete the previously downloaded repository, so that this script can also update your existing installation
rm -r $INSTALL_DIR/repository

# Download the updated repository
git clone https://github.com/riking/AutoDelete.git $INSTALL_DIR/repository

# Enter the repository, build the main package, copy the compiled file to our target folder
cd $INSTALL_DIR/repository
go build cmd/autodelete/main.go
mv main $INSTALL_DIR/autodelete
chmod +x $INSTALL_DIR/autodelete

# If the config.yml file does not exist, create it from the config.example.yml template.
cp -n config.example.yml $INSTALL_DIR/config.yml

# Create a systemd service file.
# This is a simple service that will start and monitor our bot, restarting it whenever it fails.

cat <<EOT > /etc/systemd/system/autodelete.service
[Unit]
Description=Discord AutoDelete Bot
After=network-online.target remote-fs.target nss-lookup.target
Wants=network-online.target

[Service]
Type=simple
User=$LINUX_USER
Group=$LINUX_USER
Restart=on-failure
RestartSec=3

ExecStart=$INSTALL_DIR/autodelete
WorkingDirectory=$INSTALL_DIR/


[Install]
WantedBy=multi-user.target
EOT

# After installing the service, we need to refresh systemd.
systemctl daemon-reload

# Open the editor for config.yml
EDITOR=nano editor $INSTALL_DIR/config.yml
```

At the end of the procedure, an editor will pop up.
Do not close it.
Before proceeding, you need to set up your Discord environment.

- Open the Applications portal ( https://discord.com/developers/applications )
- Create a new application
- Within OAUTH2, note your "CLIENT ID"
- Within OAUTH2, reset your "CLIENT SECRET" and note it down
- Within Redirects, add your URL ( http://your.domain.here/discord_auto_delete/oauth/callback )
- Default Auth Link can stay on None.
- Within Bot, create a new Bot.
- Reset the "BOT TOKEN" and note it down
- Choose if you want your bot to be private/public (so that only you can add this bot to a server)
- Everything else stays unchecked.

You should also look up your own Discord User ID.
It seems to be needed, and you should be able to find it by enabling Developer mode on Discord, then right clicking your profile within a server's user list, then "Copy ID".
For the sake of completing this procedure I'll let you troubleshoot this step through google.

Now you should have these informations.
- CLIENT ID
- CLIENT SECRET
- BOT TOKEN
- YOUR OWN USER ID

Within the editor that popped up earlier, put your informations in the appropriate fields.
Make sure to change /srv/autodelete if you also changed it in the script above.

```
# see config.go "type Config" for more fields
clientid: YOUR-CLIENT-ID-HERE
clientsecret: YOUR-CLIENT-SECRET-HERE
bottoken: YOUR-BOT-TOKEN-HERE
adminuser: YOUR-DISCORD-USER-ID-HERE
http:
  listen: "0.0.0.0:2202"
  public: "http://your.domain.here/"
backlog_limit: 200
errorlog: "/srv/autodelete/data/error.log"
statusmessage: "in the garbage"
```

After this, you can run these commands to control the bot:

```
# Start the bot
systemctl start autodelete.service

# Stop the bot
systemctl stop autodelete.service

# Monitor the bot logs
systemctl status autodelete.service

# Configure the bot to start at system boot
systemctl enable autodelete.service

# Configure the bot to NOT start at system boot
systemctl disable autodelete.service
```