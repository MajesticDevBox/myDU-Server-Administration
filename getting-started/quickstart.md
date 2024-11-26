---
description: >-
  This is the official Server Installation Guide from NQ, sourced from myDU
  Section of Dual Universe Website.
---

# myDU Server Installation Guide

[Original PDF](https://installer-prod.dualthegame.com/mydu-server/documentation.pdf)

<figure><img src="../.gitbook/assets/GetReady.png" alt=""><figcaption></figcaption></figure>

## 🚀 What you will need

A Linux server with docker and docker-compose installed, or a windows PC with docker-desktop installed. If you plan on hosting an instance on the public internet, having a domain name and going through the “securing your stack” steps a few sections below is highly recommended. In case you are using Windows, replace all '.sh' scripts invocation with '.bat' equivalent script in this whole documentation.

## 🌞 How big should my server be?

Default install requires 20Gb of disk space, and takes 3Gb of database size (mongodb + postgresql). This will grow with the number of constructs, blueprints and holes in the planets that are made. CPU-wise 4 to 8 cores should (no promises) work for a small bunch of players. Please note that industry units are quite CPU hungry, so you might need to beef up if you plan on making huge factories.

## 🏦 Architecture overview

{% hint style="warning" %}
All exposed services except the front which is a direct GRPC connection are routed through a nginx HTTP server, to expose only public routes. A player client first connects to "queueing", the queuing service, which validates login and redirects player to a "front" with rate limiting.
{% endhint %}

### <mark style="color:blue;">The client then connects over GRPC to the front. Most traffic is routed on that connection, except:</mark>

* voxel and mesh gets are sent directly to the voxel service
* construct elements are fetched directly from orleans
* some element properties are fetched directly over http (served by nginx)

### <mark style="color:blue;">The stack runs the following images from a docker-compose file:</mark>

* <mark style="color:yellow;">`nginx`</mark> with mounted conf: ensure only public routes are accessible
* <mark style="color:yellow;">`mongo`</mark> with mounted data dir: voxels and meshes are stored there
* <mark style="color:yellow;">`redis`</mark> used internally for caching and inter-service data exchange
* <mark style="color:yellow;">`postgres with mounted data dir`</mark> main databases for everything not voxel
* <mark style="color:yellow;">`rabbitmq`</mark> use for inter-service internal communication
* <mark style="color:yellow;">`zookeeper`</mark> needed by kafka
* <mark style="color:yellow;">`kafka`</mark> used for internal job queueing
* <mark style="color:yellow;">`front`</mark> endpoint for client, mainly just a router to other services
* <mark style="color:yellow;">`node`</mark> visibility service
* <mark style="color:yellow;">`orleans`</mark> runs the Microsoft Actor Framework with same name. Everything gameplay is in there
* <mark style="color:yellow;">`constructs`</mark> microservice that serves construct information
* <mark style="color:yellow;">`queueing`</mark> authenticates and queues clients before dispatching them to the front
* <mark style="color:yellow;">`voxel`</mark> everything voxel and mesh
* <mark style="color:yellow;">`market`</mark> market microservice handling market orders and transactions
* <mark style="color:yellow;">`nodemanager`</mark> used for proper stack initialisation
* <mark style="color:yellow;">`backoffice`</mark> the backoffice used by game masters and support team
* <mark style="color:yellow;">`sandbox`</mark> image containing various maintenance utilities

{% hint style="info" %}
All DU services use a YAML configuration file, mounted as <mark style="color:yellow;">**`config/dual.yaml`**</mark>. Additionally, some resources used at runtime are stored in <mark style="color:yellow;">**`data/`**</mark> subdirectories. Those can be moved to an AWS S3 bucket by changing the configuration.
{% endhint %}

### Runtime resource folders:

* <mark style="color:yellow;">`user_content(RW)`</mark> those are element properties that can be too big to store efficiently in postgres.
* <mark style="color:yellow;">`tutorials`</mark> startup choices, challenges and tutorials
* <mark style="color:yellow;">`pools`</mark> ore pool definitions for planets and alien space cores
* <mark style="color:yellow;">`wrecks`</mark> wreck models
* <mark style="color:yellow;">`asteroids`</mark> asteroid models\\
* <mark style="color:yellow;">`aliens`</mark> alien space core definitions

{% hint style="info" %}
Finally, database files (mongo and postgres) are stored in <mark style="color:yellow;">**`db-mongo`**</mark> and <mark style="color:yellow;">**`db-postgres`**</mark> locally. You can change the data and db path by editing the <mark style="color:yellow;">**`.env`**</mark> file.
{% endhint %}

## 🪟 QuickStart for Windows:

Install Docker Desktop from Docker Inc. It is free and does not require any signup. Start <mark style="color:yellow;">`dual-universe-server-installer.exe.`</mark> Do **not** pick a location in <mark style="color:yellow;">`Program Files`</mark> for the installation, the default is fine if you have enough disk space. The installer takes 20G of disk space and might need an hour to complete. Once it’s finished running, double-click on <mark style="color:yellow;">`Start DU`</mark> icon on your desktop to launch the server, and connect to it with the MyDU client, putting <mark style="color:yellow;">`localhost`</mark> in the <mark style="color:yellow;">`server URL`</mark> field, and the credentials you gave to the installer, using “admin” as login.

## 🦌 QuickStart without installer:

Download the compose and scripts package provided by Novaquark. This is achieved by running in the directory where you want to copy files:

```docker
docker run --rm -it -v ./:/output novaquark/dual-server-fas
tinstall:latest

```

All scripts must be run from this directory. On windows replace all '.sh' scripts with their '.bat' version with same name. Run the script that will edit the config/dual.yaml file by filling your domain name or IP address to the components that need absolute URLS:

```docker
cp config/dual.yaml config/dual.yaml.ori
python3 scripts/config-set-domain.py config/dual.yaml htt
p://<MY_IP_OR_DOMAIN> MY_IP
```

Second argument must be a URL, third argument must be an IP address.

Set password for your first admin user:

```docker
scripts/admin-set-password.sh admin MY_PASSWORD
```

Start the complete stack:

```docker
scripts/up.sh
```

And that's it, you should be able to log in as admin to the game and backoffice. Do NOT run <mark style="color:yellow;">`docker-compose up -d`</mark> as inter-service dependency will make some services fail to boot properly.

## ⬇️ Making your server accessible to others:

Note: <mark style="color:yellow;">**the command below is required if you didn’t set IP address in the windows installer, and you wish for any kind of non-local access to your server.**</mark>

If you wish to host a DU server for others on your home internet connection in insecure mode, you need to forward a few TCP ports from your Router to your computer. You can either use the UI of your Router, or a program like <mark style="color:yellow;">`UPnP Wizard`</mark>. You need to forward TCP ports <mark style="color:yellow;">8081</mark>, <mark style="color:yellow;">10111</mark>, <mark style="color:yellow;">9630</mark>, <mark style="color:yellow;">9210</mark> and <mark style="color:yellow;">10000</mark>. Finally, you need to edit <mark style="color:yellow;">`config/dual.yaml`</mark> so that it advertises your public IP address: first obtain your public ip address (ask any search engine), then run the below command, replacing the two occurrences of <mark style="color:yellow;">`MY_IP`</mark> with your actual IP address. And that's it, other people should be able to connect to your DU server by entering your IP address in the <mark style="color:yellow;">`server URL`</mark> field.

```docker
python3 scripts\\config-set-domain.py config/dual.yaml htt
p://MY_IP MY_IP
```

## 📀 Backing up your stack:

A backup script is provided as <mark style="color:yellow;">`scripts/backup.sh`</mark> (<mark style="color:yellow;">`backup.bat`</mark> for Windows). When run it will create a directory with current date, and dump postgres, mongo and user\_content in this directory, while the server stack is running.

## 🛠 Maintenance mode:

If you set in <mark style="color:yellow;">`dual.yaml`</mark> <mark style="color:yellow;">`queueing.start_maintenance`</mark> to `true`, the server will boot in maintenance mode, allowing only admins to connect. You can exit maintenance mode with:

```docker
scripts/maintenance-mode-off.sh
```

## 💥 Known Issues and Quirks:

Admin players (flagged as such in the backoffice) are intentionally invisible to non admin players. They can chose to be visible by toggling a setting in the “global settings” client menu.

## 👱 User management:

The oauth and community database management used by Novaquark has been replaced by a simple local auth system (with securely hashed passwords). Users are stored in the <mark style="color:yellow;">`dual.auth`</mark> table. Notable fields are username (login), password (hashed) and roles (comma-separated list of roles). The <mark style="color:yellow;">`config/dual.yaml`</mark> can be edited to tweak which role can do what, in the <mark style="color:yellow;">`auth.roles`</mark> section. `game_access` specifies the roles to login in game: a user must have one of these roles to login with the client and play DU. <mark style="color:yellow;">`high_priority`</mark> specifies roles which connects in priority from a separate wait queue <mark style="color:yellow;">`bypass_priority`</mark> roles bypass the waiting queue entirely. <mark style="color:yellow;">`impersonation`</mark> is allowed to impersonate other users by using the <mark style="color:yellow;">`login: impersonate_to`</mark> syntax. The role <mark style="color:yellow;">`bo`</mark> is required, but not sufficient, to connect to the Backoffice: the backoffice also limits which action a user can take based on its own role system.

Those roles can be tweaked with dual.yaml entries backoffice.administrator\_roles, <mark style="color:yellow;">`backoffice.customer_support_roles`</mark> and <mark style="color:yellow;">`backoffice.game_designer_roles`</mark> . Users can be managed on the backoffice <mark style="color:yellow;">`Users`</mark> page, which require the role `Administrator` to access.

Users can reset their own password by going on the <mark style="color:yellow;">`/guest`</mark> page of the backoffice. For mails to work properly you need to fill a few fields on the backoffice section of the dual.yaml config file:

* <mark style="color:yellow;">`public_url`</mark> this is the base public url at which your backoffice is reachable
* <mark style="color:yellow;">`smtp_sender_email`</mark> this is the 'from' email used to send the password reset email
* <mark style="color:yellow;">`smtp_host`</mark> host of your SMTP server. The default 'smtp' uses the built-in postfix.

Once this is setup you can also invite new users by email or invite link from a form on the <mark style="color:yellow;">`Users`</mark> page.

### 🧾 Automatic account creation:

If you set <mark style="color:yellow;">`auth.allow_account_creation`</mark> to true in <mark style="color:yellow;">`dual.yaml`</mark>, players will be allowed to create accounts on their own on your server without any supervision by going to the <mark style="color:yellow;">`/guest`</mark> backoffice page.

## 📰 Public Server Listing:

The myDU client has a <mark style="color:yellow;">`server list`</mark> feature that lists public myDU servers. You can opt-in to this system and have your server publicly listed from the backoffice by going to the <mark style="color:yellow;">`public listing`</mark> page on the left menu. You will be presented with a form to fill in to register your server and have it publicly listed. Once done you can return to this page to edit information associated with your server, such as name and description.The server list system remembers your server using the file <mark style="color:yellow;">`config/server_secret`</mark>. To unlist permanently one can simply delete this file. Please note that Novaquark requests that you adhere to the terms of service, and Novaquark reserves the right to remove your server listing for any reason.

## 🔐 Securing your stack: Enabling SSL:

{% hint style="danger" %}
This is an advanced use case that requires a bit of knowledge in DNS handling. By default, my-du uses multiple http connections which are unencrypted, which is not ideal. If you own a domain name, you can use the provided script <mark style="color:yellow;">`du-ssl.py`</mark> through its wrapper<mark style="color:yellow;">`/scripts ssl.sh`</mark> to:
{% endhint %}

* obtain SSL certificates through <mark style="color:yellow;">`letsencrypt`</mark>
* reconfigure <mark style="color:yellow;">`dual.yaml`</mark> to advertise the new https domain-based URLs
* reconfigure the <mark style="color:yellow;">`nginx`</mark> proxy to serve all ports with SSL

## 🌍 Creating DNS entries:

The first step is to create 5 <mark style="color:yellow;">`A Records`</mark> in your DNS zone pointing to the IP of your server. This is usually accomplished through connecting to your domain registrar website. By default they are named <mark style="color:yellow;">`du-orleans`</mark>, <mark style="color:yellow;">`du-queueing`</mark>, <mark style="color:yellow;">`du-usercontent`</mark>, <mark style="color:yellow;">`du-voxel`</mark> and <mark style="color:yellow;">`du-backoffice`</mark>,  but those can be renamed to something else (see next section).

## ✍️ Editing domains.json:

Open the file 'config/domains.json in a text editor and fill it according to the domain names you've created. The simplest way is to set <mark style="color:yellow;">`prefix`</mark> to <mark style="color:yellow;">`du-`</mark> and `tld` to your domain name.

Alternatively, you can set the full domain per service by creating a <mark style="color:yellow;">`services`</mark> dictionary entry and filling the values for <mark style="color:yellow;">`orleans`</mark>, <mark style="color:yellow;">`queueing`</mark>, <mark style="color:yellow;">`backoffice`</mark>, <mark style="color:yellow;">`voxel`</mark> and <mark style="color:yellow;">`usercontent`</mark>.

Example:

```json
{
  "tld": "yourdomain.com",
  "prefix": "du-",
  "services": {
    "queueing":"login",
    "backoffice":"bo"
  }
}
```

## ↗️Redirecting ports:

The ports <mark style="color:yellow;">`443`</mark>, and <mark style="color:yellow;">`80`</mark> (during install only) on those domain names must point to the computer running the my-du server. If you are behind a router such as a home connection, you probably need to configure your router to redirect those ports.

## 🔑 Creating SSL certificates:

Temporarily disable any service (apache, nginx) listening on port <mark style="color:yellow;">`80`</mark> of the computer running the my-du server, and type the following:

```docker
./scripts/down.sh
```

{% hint style="info" %}
Make sure that the `A Records` for DNS match the new FQDN

Make sure ports <mark style="color:yellow;">80</mark> and <mark style="color:yellow;">443</mark> are forwarded to the server.
{% endhint %}

**Create new certs:**

```powershell
./scripts/ssl.sh --create-certs
```

This will start certbot from letsencrypt to request the certificate for the 5 domain names defined in <mark style="color:yellow;">`config/domains.json`</mark> . It might ask you a few questions on the terminal. Note that it will emit a challenge by connecting to port <mark style="color:yellow;">`80`</mark> on all five domains, so you might have to wait for DNS propagation before this step. If all went well the keys and certificates are created in the <mark style="color:yellow;">`letsencrypt`</mark> folder.

#### &#x20;Wrapping Up

Once SSL is enabled your stack only uses two ports: <mark style="color:yellow;">443</mark> and <mark style="color:yellow;">9210</mark>. Your stack server address becomes <mark style="color:yellow;">https://du-queueing.MYDOMAIN:443</mark>. The certificates are valid for 3 months and can be renewed by running:

```docker
./scripts/ssl.sh --update-certs
```



## 🥞 Reconfiguring the stack:

Simply run the below command, replacing <mark style="color:yellow;">`MY_IP`</mark> with the public IP address of the server or computer running the my-du server.

```
./scripts/ssl.sh --config-dual YOUR-IP
```

{% hint style="info" %}
This must be IP Address not TLD
{% endhint %}

```
./scripts/ssl.sh --config-nginx
```

Now you can restart your Docker Stack:

```
./scripts/up.sh
```

These two commands will update <mark style="color:yellow;">`config/dual.yaml`</mark> and <mark style="color:yellow;">`nginx/conf.d`</mark> with the domain information.

## 🖋 Editing docker-compose to forward port 443:

Open docker-compose.yml in a text editor, locate the <mark style="color:yellow;">`nginx`</mark> section and uncomment the line <mark style="color:yellow;">`- 443:443`</mark> to forward port <mark style="color:yellow;">`443`</mark> to the nginx server. You can comment all other port lines as they are unused in SSL mode.

## 🔗Advanced: authentication webhook:

Should you wish to interfere with the queueing service authentication mechanism, a webhook can be enabled:

Set in dual.yaml “auth: auth\_hook\_url:” that url will be called with a json payload containing the login request without the password. Login will be denied if an HTTP error code (4xx 5xx) is returned. Note that the webhook is called by the queuing service itself, so from within the docker stack.

## 🕹 Game master guide:

All element properties and many gameplay features are tweakable through the <mark style="color:yellow;">`item hierachy`</mark>, accessible from the backoffice. The backoffice is accessible on port `12000` on your host over https. It uses by default a self-signed certificate that you can change, stored in <mark style="color:yellow;">`nginx`</mark> directory.

## 💿 Image management:

By default, the server will prevent your users from using images at external URLs. You as administrator can specify on which domains images are allowed to load through the <mark style="color:yellow;">`CoherentConfig`</mark> item bank entry, on field <mark style="color:yellow;">`imageCDNList`</mark>, that you can change on the backoffice. To allow all (at your own risks), use the wildcard <mark style="color:yellow;">`*`</mark>.

## ⚛️ Basic element properties:

Let's take an example: suppose you want to increase max HP of the container hub. To achieve this, login to the backoffice and go to the component hierarchy page through the left menu. Type <mark style="color:yellow;">`ContainerHub`</mark> in the search field, which should highlight a single entry in the hierarchy below. Clicking on the <mark style="color:yellow;">`ContainerHub`</mark> card should open a new page where you can edit properties.

The editor shows you the definition of a ContainerHub in YAML format. Change the value of <mark style="color:yellow;">`hitpoints`</mark> to your liking and hit <mark style="color:yellow;">`Save`</mark>. Et voila, all newly deployed container hubs will have your new value of hit points set!

Existing elements are not affected: since the <mark style="color:yellow;">`hitpoints`</mark> property can be affected by talents, the actual max HP value is stored per element at deployment time in the database in what is called a dynamic property. Fortunately, you can ask the system to recompute existing properties through an administrative command in the sandbox image:

```
/python/DualSQL/DualSQL --recompute-props DEF.json
```

<mark style="color:yellow;">`DEF.json`</mark> is a json dictionary of element names to list of property names that changed. You can use parent element names, and it will recurse. So, following our example, you'd create a DEF.json with the following content:

```json
{ 
"ContainerHub": ["hitpoints"] 
}
```

And execute the property recomputation by running:

```docker
docker-compose run --rm --entrypoint /python/DualSQL/DualSQ L --mount \\ type=bind,src=$PWD/DEF.json,dst=/DEF.json sandbox --reco mpute-props /DEF.json
```

This will update the hitpoints property of all deployed container hubs, by taking the new base value and re-applying all talents that were used.

{% hint style="warning" %}
It is highly recommended to backup your item hierarchy data when tinkering with it, by using the export button, which will download a YAML dump of the item hierarchy to your computer.
{% endhint %}

## 🚗 Exporting blueprints from the production server:

Should you wish to import a blueprint you own from the Dual Universe production server, you can use the meshexporter service here. Login with your NQ credentials. You will be presented with a list of all core blueprints in your inventory. Simply hit the “export” button to download a JSON representation of your blueprint. Then login to the backoffice of your my-du server, and go to the blueprint page. From there you can import the JSON file to generate the blueprint, and grant it to any player.

{% embed url="https://meshexporter.dualuniverse.game/" %}

## 🪨 Dynamic Asteroids:

Those are controlled by <mark style="color:yellow;">`AsteroidManagerConfig`</mark> in <mark style="color:yellow;">`FeaturesConfig`</mark>:

* <mark style="color:yellow;">`count`</mark> : how many asteroids per tier to maintain
* <mark style="color:yellow;">`lifetimeDays`</mark> : how long before destroying an asteroid
* <mark style="color:yellow;">`notBefore`</mark> : unix timestamp that prevents the whole feature from activating before given date
* <mark style="color:yellow;">`planets`</mark> : list of planet ids around which to spawn asteroids

Asteroids are spawned from a list of asteroid model fixtures (the construct JSON files) located in `data/asteroids` on the host. An unlinked backoffice page <mark style="color:yellow;">`v3/asteroidsmanager.html`</mark> can be accessed once logged in to show information about live asteroids.

## 🧠 Talent System:

All talents are defined in the item hierarchy as direct children of the <mark style="color:yellow;">`talent`</mark> node. Let's consider this example and break down the fields:

```
ContainerProficiency:
	parent: Talent
	displayName: Container proficiency
	group: InventoryManager
	description: +10 % storage volume on containers when depl oyed
	order: 3
	requiredTalents:
	- {name: "NanopackUpgrades", level: 3}
	  costs: [1800, 9000, 45000, 225000, 1125000]
	  effects:
	  - name: ContainerProficiency
	    target: PLAYER_ITEM
	    modifiers:
	    - name: ContainerProficiency
	      property: maxVolume
	      action: ADD_PERCENT
	      value: 10
	      itemId: ItemContainerBase
```

<mark style="color:yellow;">`Group`</mark> specifies in which group the talent will appear. Talent groups are defined in the item hierarchy too.

<mark style="color:yellow;">`description`</mark> is what is shown to the player on the client

<mark style="color:yellow;">`requiredTalents`</mark> sets up the prerequisites to acquire level 1 of this talent

<mark style="color:yellow;">`costs`</mark> are the XP cost per level

<mark style="color:yellow;">`effects`</mark> is a list of effects, each effect being a list of modifiers.

<mark style="color:yellow;">`target`</mark> specifies the effect type. Possible values are:

* <mark style="color:yellow;">`PLAYER_ITEM`</mark> element properties when put down
* <mark style="color:yellow;">`PLAYER`</mark> affects a property of the player (see Character in item hierarchy)
* <mark style="color:yellow;">`ELEMENT`</mark> affects a property of an element while using it

<mark style="color:yellow;">`property`</mark> is the affected property. <mark style="color:yellow;">`itemId`</mark> is the affected item name (this item and all of it's children). <mark style="color:yellow;">`action`</mark> can be <mark style="color:yellow;">`ADD`</mark> or <mark style="color:yellow;">`ADD_PERCENT`</mark> to be additive or multiplicative. <mark style="color:yellow;">`value`</mark> is the bonus value per level.

## 👽 Alien Space Constructs:

Alien space cores are reconciled dynamically if needed at stack boot, from the definition in file <mark style="color:yellow;">`data/aliens/alienspacecores.json`</mark>. The construct model used is <mark style="color:yellow;">`data/aliens/alien-station.json`</mark>.

The definition file looks like:

```
{
	"990001": {"name": "Alpha", "position": {"x": 33946000, "y": 71381990, "z": 28850000}}, 
}
```

The key is the construct id. You must use the <mark style="color:yellow;">`9900XX`</mark> range. The ore pools provided by the alien stations are in <mark style="color:yellow;">`data/pools/orepools-0.json`</mark>.

## 🏝 Territory Fees:

Territory fees are controlled by the <mark style="color:yellow;">`TerritoriesConfig`</mark> I.H. node. All fields should be self-explanatory.

## 🛥 Inactive Asset Requisition:

Being tied to the Novaquark Community database, it is recommended to disable the IAR for now as it will not work properly. This can be done by setting <mark style="color:yellow;">`constructGarbageCollection`</mark> to <mark style="color:yellow;">`false`</mark> in <mark style="color:yellow;">`FeaturesList`</mark>.

## 👥 Organization Construct Limit:

Can be disabled by setting <mark style="color:yellow;">`FeaturesList.orgConstructLimit`</mark> set to false. Can be set to warning only without enforcing any deletion by setting <mark style="color:yellow;">`orgConstructLimitWarnOnly`</mark> to true.

## ⛏️ Ore Pools:

Ore pools are defined in json files in <mark style="color:yellow;">`data/pools`</mark> named <mark style="color:yellow;">`orepools-PLANETID.json`</mark> (0 being for alien space cores). Those files are cached at runtime, so changes require a stack reboot.

## ⚔️ PvE:

The PVE mission definition files are located in <mark style="color:yellow;">`data/tutorials/sentinel`</mark>. <mark style="color:yellow;">`tiers.json`</mark> gives generic info about the mission tiers. <mark style="color:yellow;">`library.yaml`</mark> defines the ships, weapons, patrol areas The parameters of the dynamic mission generator are in the item bank under <mark style="color:yellow;">`PvEGenerationConfigTIER`</mark>. Finally, the construct models of NPC ships are in <mark style="color:yellow;">`ships`</mark> subdirectory.

## 💰 Missions and Jobs:

Two backoffice page can be used to list jobs and missions: <mark style="color:yellow;">`v3/missions.html`</mark> and <mark style="color:yellow;">`v3/formalmissions.html`</mark>. Aphelia missions are defined in a YAML file, which can be injected in the unlinked backoffice page <mark style="color:yellow;">`v3/haulingfixtures.html`</mark>. A sample file is provided in the <mark style="color:yellow;">`data/extra`</mark> directory. The ratio of visible aphelia missions is configurable in the item bank through the <mark style="color:yellow;">`MissionConfig`</mark> object.

## 📈 Market Bot Orders:

Those are defined in <mark style="color:yellow;">`data/market_orders`</mark>. For each market the system will recurse the construct hierarchy until it finds a csv with name the id of the construct. To trigger bot order generation, go to the backoffice markets page, and hit the <mark style="color:yellow;">`seed`</mark> button at the top.

## 🏢 Backoffice Extra Pages:

<mark style="color:yellow;">`v3/wallets.html`</mark> can be used to see wallet logs and notification logs.

<mark style="color:yellow;">`v3/audit.html`</mark> can be used to see the backoffice audit log, which shows all operations made through the backoffice that modified the state of the game.

## Credits:

Creator of original document

[NQ-Bearclaw](https://github.com/NQ-Bearclaw)
