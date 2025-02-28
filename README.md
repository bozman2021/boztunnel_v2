With BozTunnel, you can publish your local LLM APIs and apps on the internet so that you can use your local LLMs and LLM apps on the go and share them with friends.

The services are also published locally with TLS certificates for the same domain name. When you configure a local DNS server to resolve the domain name to the local IP address of the machine running the services, you can use the services at home within your local network without changing the domain name.

**Learn how to set it up on Youtube: Publish Ollama and OpenWebUI on the Internet with Cloudflare Tunnels**

**It's a Docker Compose Stack.**

If you don't see a copier.yml here, then this is the actual Docker Compose stack that publishes your local LLM services on the internet. This project was created with the llamatunnel copier template.

To learn how to set up the stack, please read the Installation section below.

# Installation

**Prerequisites**

- **Docker Desktop**
- **Ollama** installed on your machine and running on http://localhost:11434
- **A Cloudflare account.**
- **A domain mangaged with Cloudflare DNS.**
cloudflared installed on your machine. You can install it with **brew install cloudflare/cloudflare/cloudflared** on macOS or **winget install --id Cloudflare.cloudflared** on Windows. For linux, use the package manager of your distribution.
- **Python 3.8** or newer
- **Git 2.27** or newer

**Install Copier**

On Linux:
```
python3 -m pip install --user pipx
python3 -m pipx ensurepath
pipx install copier
```

Note that you probably need to start another terminal after installing pipx to make the pipx command available.

**Create a Cloudflare API Token**

Create a Cloudflare API token with the permissions to modify the DNS zone. This is needed for the Caddy DNS challenge. Go to the Cloudflare dashboard and create a token as follows:

My Profile > API Tokens > Create Token > Edit Zone DNS > Zone:DNS:Edit > Include:Specific Zone:<DOMAIN_NAME> > Continue to Summary > Create Token

Login with cloudflared on Your Local Machine

On you local machine, login with cloudflared:
````
cloudflared tunnel login
````
**Create a Folder for the Docker Stack**

If you keep all you docker stacks in a directory like $HOME/docker-stacks, you can use the following commands to create the Llama Tunnel stack in a new sub-directory.

macOS and Linux:
````
STACK_DIR="$HOME/docker-stacks/boztunnel"
mkdir -p $STACK_DIR
cd $STACK_DIR
````

**Create a Data Directory for cloudflared**

````
mkdir -p ./data/cloudflared
````

**Create a Tunnel and DNS Routes**

Create a tunnel and DNS routes for the services. cloudflared tunnel create returns a tunnel id. Please note it as you will need it later. This assumes, you keep the data directory at the default location which is ./data in the project directory. If you want to change location of the data directory, you need to create the directory structure and adjust the path accordingly.

macOS and Linux:
````
TUNNEL_NAME="boztunnel"
DOMAIN_NAME="askboz.se"
API_SUBDOMAIN="minime"
APP_SUBDOMAIN="app"
cloudflared tunnel create --credentials-file ./data/cloudflared/credentials.json $TUNNEL_NAME
cloudflared tunnel route dns $TUNNEL_NAME $API_SUBDOMAIN.$DOMAIN_NAME
cloudflared tunnel route dns $TUNNEL_NAME $APP_SUBDOMAIN.$DOMAIN_NAME
````

Note that you cannot manage this tunnel on the Cloudflare Dashboard. Instead, you need to use the cloudflared CLI tool to manage the tunnel and the DNS routing, otherwise you won't get the credentials.json file which is required to authenticate the cloudflared service.

**Create the Project from the Template**

Create the project from the template with copier and answer the questions. If you forgot the tunnel id, you can find it in the data/cloudflared/credentials.json file or see it with cloudflared tunnel list.
````
copier copy gh:bozman2021/boztunnel .
````

This will create a new directory in the STACK_DIR with the all the files necessary to run the tunnel.

**Change Permission**
````
chmod 755 ./data/cloudflared/credentials.json
````

**Start the Services**

Change into the project directory and start the services in the foreground:

````
docker compose up --build
````

Now you can use Ollama at https://api.example.com and OpenWebUI https://app.example.com (assuming you use app and api as subdomains). You can find the API key for Ollama in the **.env** file.

**Commit the Project to Git**

You need to keep this project in git if you want to use the update feature of copier. See below for more info abut this. This is how you can do it locally:
````
git init
git add .
git commit -m "Initial commit"

````
Usage

## Start/Stop

****Start in detached mode****

````
docker compose up --build -d
````

****Stop****
````
docker compose down
````

****Restart the services with detached mode****

````
docker compose up --build -d
````
****Restart the services with full output****

````
docker compose up --build
````
## Update code

**Update or Change the Configuration**

You can update the project with copier. This way, you can get new features, updated docker images, or bug fixes in the template and you can also change the configuration of the services.

Change into the project directory.
Stop the services, for example with docker-compose down.

Take a backup, for example with **zip -r -X "../llamatunnel.zip" .**
Gather the existing secret configurations (you can find them in the **.env** file).

Run copier update in the project directory.
Resolve any conflicts, if necessary.
Review the changes with git diff and commit them with 
````
git commit -am "Update".
````

> [!CAUTION]
> ***Caution: Make sure to provide the same answers for the secrets, as before if you want to keep the secrets. Again, you can find the secrets in the .env file.***

You can just acknowledge the other, non-secret options if you don't want to change anything.

If you just want to change the configuration, while keeping the current version, get the _commit value from the .copier-answers.yml file, and use it as the value of the --vcs-ref option. Usually, this is a git tag, in which case you can call copier update for example like this:
````
copier update --vcs-ref tags/0.1.7
````

> [!CAUTION]
> ****Security Notes****
> ****The .env file contains secrets, namely the CLOUDFLARE_API_TOKEN, the OLLAMA_API_KEY, and the WEBUI_SECRET_KEY. Don't commit the .env file to version control and store the secrets somewhere secure, so that you can enter them again on updates or reinstalls.****
>****The data directory also contains secrets, for example the credentials.json file and the private key of the TLS certificate. Take backups of the data directory, and don't commit it to version control.****

> [!NOTE]
> 
> If you think you've found a security issue, please follow GitHub's general instructions to privately report a security vulnerability.

### Solution Overview

This project uses Docker Compose to start cloudflared, Caddy, and OpenWebUI and Cloudflare Tunnels to route traffic from certain domains on the internet to Ollama and OpenWebUI on your local machine.

### DNS

**api.example.com** points to your local Ollama (which should already be installed on your machine)
**app.example.com** points to your local OpenWebUI

### Cloudflared

Creates an outbound-only connection to Cloudflare’s global network.
Forwards traffic only to the local Caddy service. Ollama and OpenWebUI are not directly exposed to the internet.

### Caddy

Acts as a reverse proxy.
Protects the Ollama API with a configurable API key.
Serves on https with SSL certificates for the same domain name. Thus, you can use a local DNS server to access the services from your local network without going over the internet.

### OpenWebUI

Is a chat app that uses the OpenAI API to create a chat experience.
Directly talks to the local Ollama, without going over the internet or Caddy.
### Ollama

Uses llama.cpp to run large language models on your local machine and expose them with an OpenAI compatible API.
Should already be installed directly on your machine and listen on http://localhost:11434 (the default configuration).

### Docker Compose

Manages multiple Docker containers in a single yaml file andå with a single command.
Encapsulates the services in a user-defined network, so that they can communicate with each other over an internal DNS.
Maps the ports of the Caddy and OpenWebUI services to the host system, so that you can also access the services from your local machine or your local network.
Requires that Docker Desktop or something similar is installed on your machine.

## FAQ

What are the Benefits of Using Cloudflare Tunnels to publish my local LLM Services on the Internet?

In principle, you can publish your local services on the internet in three different ways:

### Method	Pros	Cons
VPN with Dynamic DNS	Secure, private, and encrypted connection	gives access to your network, you need to open the tunnel to use the services
Open ports with Dynamic DNS	Easy to set up, no need for a VPN	you need to open ports on your router, needs additional security measures in your network, could be blocked by your internet service provider
Outbound tunnel to a gateway service on the internet	no need to open ports on your home network, can be shared with friends without handing out VPN access to your network, you can authenticate/authorize outside of your network	you need a service on the internet
An outbound tunnel is a good compromise between security, convenience, and shareability.

Cloudflare Tunnel is just one of many services you can use for outbound tunnels. The benefit is that it's free if you already have a domain you can use for this purpose. The main drawbacks of Cloudflare Tunnels are the complex setup, and some limitations of the service.

It would be more secure to perform pre-authentication of API clients and users outside of your network. Cloudflare supports this with the Zero Trust Access service, but this can get quite expensive.

Why Do You Install cloudflared as CLI Tool?

I use cloudflared as cli tool for administrative tasks and don't run them in the docker container defined in docker-compose.yaml. This way, I don't need to make my Cloudflare admin credentials available to the cloudflared service.

What if I have a Firewall?

If you have a firewall, you need to allow outbound connections to the Cloudflare network form the machine you run the solution on. You can find the DNS names and ports in the Cloudflare Tunnel with firewall documentation.

Why and How Do You Build a Custom Docker Image for Caddy?

The custom Caddy image adds the Cloudflare module for the DNS challenge to Caddy which is not built into Caddy by default.

Caddy uses the DNS challenge to create TLS certificates with Let's Encrypt for the public domains. Trusted TLS certificates for this domain are now both on Cloudflare and Caddy. This way, you can use the services on your local network at same domain name. You just need to configure your local DNS server to point them to the machine Caddy is running on.

Creating a custom Caddy binary and package it in an image can be done with a Docker Multi-stage build:

The first stage uses Caddy's builder image and runs the xcaddy command to build caddy with the Cloudflare module.
The second stage create a new image and copies the resulting binary from the builder into it.
Here is the Dockerfile, you can find it in  ./images/caddy/Dockerfile:

FROM caddy:2.7.6-builder AS builder
RUN xcaddy build --with github.com/caddy-dns/cloudflare
FROM caddy:2.7.6
COPY --from=builder /usr/bin/caddy /usr/bin/caddy
If you want to learn more about this, You can find a description of the procedure in the Caddy Image Overview on Docker Hub (scroll down to the "Adding custom Caddy modules" section).

What Do You Use to Edit the Caddyfile?

I use Visual Studio Code with the Caddyfile Support extension for Visual Studio Code to get syntax highlighting, suggestions, and documentation hints for Caddyfiles.

Can I Publish the Services on Deeper Level Subdomain like ollama.home.example.com?

This is possible, but you need a paid feature from Cloudflare. If you want to proxy a wildcard DNS record on a deeper level like *.local.example.com you can subscribe to Cloudflare Advanced Certificate Manager. For more information, see the Cloudflare Blog post Wildcard proxy for everyone by Hannes Gerhart.

How Can I Add Monitoring and Logging?

You can start all services with a monitoring endpoint that exposes Prometheus metrics. You can use the Prometheus and Grafana Docker images to monitor the services. Here are some links to get started:

Cloudflare Tunnel Metrics
Monitoring Caddy with Prometheus Metrics
Thibault Debatty: Use Loki to monitor the logs of your docker compose application
For cloudflared, you can use the --loglevel debug option to get more detailed logs.

You can also use the Cloudflare Tunnel logs to monitor the tunnel and the DNS routing.

Why Don't You Use DNSSEC?

I don't use DNSSEC in this setup because I want to use the same domain name for local and public access with a local DNS server that resolves the domain to the local IP address. While this is possible to implement with DNSSEC, I have not tried it and I believe it is also quite hard to do.

If you don't want to use the services locally with the same domain name, you can probably activate DNSSEC on your domain. If you do, I would like to hear about your experience, as I haven't tried it yet.

Can I Control the Geographic Region of the Cloudflare Tunnel?

While you can theoretically use the --region option to specify the region to which the tunnel connects, the only available value is us which routes all connections through data centers in the United States.

When you don't select this option or leave it empty you will connect to Cloudflare's global region. Thus, in effect, you cannot control in which region your tunnel will run, except for the global region or the us region.

Reference: Cloudflare Tunnel Run Parameters

How Can I Troubleshoot the Cloudflare Tunnel?

You can set a verbose log level and run tunnels with test commands. See the Cloudflare docs for Locally Managed Tunnels for more information.


# Create Personal Access Token on GitHub

From your GitHub account, go to **Settings → Developer Settings → Personal Access Token → Tokens (classic) → Generate New Token (Give your password) → Fillup the form → click Generate token → Copy the generated Token, it will be something like ghp_sFhFsSHhTzMDreGRLjmks4Tzuzgthdvfsrta**
Now follow the below method based on your machine:

## For Windows OS ⤴

Go to Credential Manager from Control Panel → Windows Credentials → find git:https://github.com → Edit → On Password replace with with your GitHub Personal Access Token → You are Done

If you don’t find **git:https://github.com** → Click on **Add a generic credential** → Internet address will be **git:https://github.com** and you need to type in your username and password will be your GitHub Personal Access Token → **Click Ok** and you are done

## For macOS ⤴

Click on the Spotlight icon (magnifying glass) on the right side of the menu bar. Type Keychain access then press the Enter key to launch the app → In Keychain Access, search for github.com → Find the internet password entry for github.com → Edit or delete the entry accordingly → You are done

## For a Linux-based OS ⤴

For Linux, you need to configure the local GIT client with a username and email address,
````
$ git config --global user.name "your_github_username"
$ git config --global user.email "your_github_email"
$ git config -l
````

Once GIT is configured, we can begin using it to access GitHub. Example:
````
$ git clone https://github.com/YOUR-USERNAME/YOUR-REPOSITORY
````

> Cloning into `YOUR-REPOSITORY`...
Username: <type your username>
Password: <type your password or personal access token (GitHub)
Now cache the given record in your computer to remembers the token:
````
$ git config --global credential.helper cache
````

If needed, anytime you can delete the cache record by:

````
$ git config --global --unset credential.helper
$ git config --system --unset credential.helper
````
Now try to pull with -v to verify

````
$ git pull -v
````
Linux/Debian (Clone as follows):

````
git clone https://<tokenhere>@github.com/<user>/<repo>.git
````
