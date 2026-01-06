---
layout: post
title:  "Discovering Vault MCP Server"
date:   2026-01-05 09:03:35 +0000
categories: ["hashicorp"]
---
I recently discovered that HashiCorp has released an [MCP Server][vault-mcp] for Vault. An MCP Server (Model Context Protocol server) acts as a bridge, enabling AI agents to interact with external systems by translating natural language prompts into specific API calls. This means that instead of manually writing commands or scripts, you can describe what you want in plain English, and the MCP server will handle the translation into Vault operations. It opens up new possibilities for developers, DevOps engineers, and security teams who want to experiment with AI-assisted workflows.

So I figured, why not give it a try and see how an MCP server would integrate with Vault in practice?

In the following sections, I’ll walk through setting up a Vault cluster on a Debian/Ubuntu machine and then demonstrate how to use the Google Gemini CLI as an AI agent to interact with Vault via the MCP server.

### Installing Vault
I am referring [Hashicorp's guide][install-vault] for installing Vault on Linux. I select the package manager option for Debain/Ubuntu and then copy and run the commands as it is. Once installed, it sets up the directories */opt/vault/* and */etc/vault.d/*, and also creates a local *vault* user.
```bash
wget -O - https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(grep -oP '(?<=UBUNTU_CODENAME=).*' /etc/os-release || lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install vault
```
<script src="https://asciinema.org/a/VWgd9nA26bNaB1pBA5zn2Yr8t.js" id="asciicast-VWgd9nA26bNaB1pBA5zn2Yr8t" async="true"></script>

### Configuring Vault
To configure vault with TLS, I will install *[mkcert][mkcert]*. It provides a convenient way to generate locally trusted self signed TLS certificates for development scenarios.

```bash
wget -O mkcert \
https://github.com/FiloSottile/mkcert/releases/download/v1.4.4/mkcert-v1.4.4-linux-amd64

sudo install --mode=0755 ./mkcert /usr/local/bin/ && rm mkcert
```
<script src="https://asciinema.org/a/ir1k2MyINzP4SPY5TArTiryNS.js" id="asciicast-ir1k2MyINzP4SPY5TArTiryNS" async="true"></script>

I will use `mkcert` to generate a certificate for a local domain name `vault.local` and then make this domain resolve to localhost by adding an entry in /etc/hosts file. This way I can use the certificates and configure Vault to be reachable on https://vault.local:8200/

```bash
# Creates a new CA root certificate and adds it your system’s trust store.
# Certificates you generate with mkcert will be recognized as valid and trusted locally.
mkcert -install
# Creates certificate for vault.local
mkcert vault.local
# Copies the certificate and key to /opt/vault/tls with right owner and permissions
sudo install --owner=vault --group=vault --mode=0600 *.pem /opt/vault/tls/

# Resolve vault.local to localhost
echo '127.0.0.1 vault.local' | sudo tee -a /etc/hosts

# Vault configuration file
cat <<EOF | sudo tee /etc/vault.d/vault.hcl
ui = true

storage "file" {
  path = "/opt/vault/data"
}


# HTTPS listener
listener "tcp" {
  address       = "0.0.0.0:8200"
  tls_cert_file = "/opt/vault/tls/vault.local.pem"
  tls_key_file  = "/opt/vault/tls/vault.local-key.pem"
}
EOF
```
<script src="https://asciinema.org/a/9POIiN7DhYRPCbKPt7HgVoIsK.js" id="asciicast-9POIiN7DhYRPCbKPt7HgVoIsK" async="true"></script>

### Initializing Vault
Next, I startup vault and initialize its storage backend by running `vault operator init`. During initialization, vault generates a root key which is split into 5 shards known as `unseal keys`. I then pass the unseal keys by running `vault operator unseal` thrice using a different unseal key each time. This is the default init process where a minimum of 3 unseal keys is required to decrypt and access vault data. Note also the `Initial Root Token` generated during initialization. I will use it later to authenticate the MCP Server with Vault.

```bash
sudo systemctl start vault && systemctl status vault
# This sets the context for the vault CLI
export VAULT_ADDR=https://vault.local:8200
vault status
vault operator init
vault operator unseal
```
<script src="https://asciinema.org/a/8RghT6TMG8n9upeVbqc9SEtVX.js" id="asciicast-8RghT6TMG8n9upeVbqc9SEtVX" async="true"></script>

Next, I do simple test to verify vault works as expected
```bash
export VAULT_TOKEN="<VAULT-TOKEN>"   # Replace <VAULT-TOKEN> with root token
vault token lookup
```
<script src="https://asciinema.org/a/EOTo0oefQhthRWbNtKxJPmrk3.js" id="asciicast-EOTo0oefQhthRWbNtKxJPmrk3" async="true"></script>

### Installing Vault MCP Server
Next, I install the [Vault MCP Server][vault-mcp]
```bash
wget -O vault-mcp-server.zip \
https://releases.hashicorp.com/vault-mcp-server/0.2.0/vault-mcp-server_0.2.0_linux_amd64.zip
sudo apt-get update && sudo apt-get install -y unzip
unzip vault-mcp-server.zip
sudo install -m 755 ./vault-mcp-server /usr/local/bin/
rm -rf vault-mcp-server.zip vault-mcp-server LICENSE.txt
```
<script src="https://asciinema.org/a/1lGaYczlvJIOA6BB6AiBSYFln.js" id="asciicast-1lGaYczlvJIOA6BB6AiBSYFln" async="true"></script>

### Install and configure Gemini CLI
Next, I install Google's gemini-cli along with [NodeJS][install-nodejs] which is a pre-requisite for the CLI. 
```bash
# Download and install nvm:
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.3/install.sh | bash
# in lieu of restarting the shell
\. "$HOME/.nvm/nvm.sh"
# Download and install Node.js:
nvm install 24
# Verify the Node.js version:
node -v
# Verify npm version:
npm -v
#Install Gemini CLI
npm install -g @google/gemini-cli@latest
```
<script src="https://asciinema.org/a/bnOJ2Hvdp7tC0yr7ifQreFM2e.js" id="asciicast-bnOJ2Hvdp7tC0yr7ifQreFM2e" async="true"></script>

Next I go to [Google AI Studio][ai-studio] and get a Gemini API Key. Google AI Studio has a free-tier option with restricted usage limits. However it should be enough for now.

I then configure the CLI to use it by creating a .gemini/.env file with the `GEMINI_API_KEY` variable. I also load the MCP server into the CLI by executing `gemini mcp add` command. When the CLI is started, it will spawn the MCP server automatically and connect with it over stdio. Note also I have added the `VAULT_ADDR` and `VAULT_TOKEN` variables in the .env file. The MCP server will use it to authenticate and make API calls to the Vault server.
```bash
# Directory for storing gemini CLI context and MCP logs
mkdir -p vault-mcp-server/{.gemini,logs}
cd vault-mcp-server

# Set context for the CLI by defining required env variables
echo 'VAULT_ADDR="https://vault.local:8200"' >> ./.gemini/.env
echo 'VAULT_TOKEN="<VAULT-TOKEN>"' >> ./.gemini/.env   # Replace <VAULT-TOKEN>
echo 'GEMINI_API_KEY="<API_KEY>"' >> ./.gemini/.env  # Replace <API_KEY> with your Gemini API Key

# Add MCP Server
gemini mcp add vault-mcp-server \
vault-mcp-server -- stdio --log-file=./logs/mcp.log
```
<script src="https://asciinema.org/a/CUumwzNImnVQaIcEsAgPjDbHf.js" id="asciicast-CUumwzNImnVQaIcEsAgPjDbHf" async="true"></script>

### Agent in action

I enter `gemini` to start the console and run `/mcp list` which lists the tools from the MCP server. Tools provide specific capabilities that an AI agent can access through MCP. 

<script src="https://asciinema.org/a/JHvNYAc2ktoZjyKeZukX4qfMg.js" id="asciicast-JHvNYAc2ktoZjyKeZukX4qfMg" async="true"></script>
Following tools are available in the beta release 0.2.0

  - create_mount                                                                                    
  - create_pki_issuer                                                                               
  - create_pki_role                                                                                 
  - delete_mount                                                                                    
  - delete_pki_role                                                                                 
  - delete_secret                                                                                   
  - enable_pki                                                                                      
  - issue_pki_certificate                                                                           
  - list_mounts                                                                                     
  - list_pki_issuers                                                                                
  - list_pki_roles                                                                                  
  - list_secrets                                                                                    
  - read_pki_issuer                                                                                 
  - read_pki_role                                                                                   
  - read_secret                                                                                     
  - write_secret

I enter prompts to setup a KV secret engine and add/retrieve secrets

**prompt-1:** *Create KV V2 secret engine mount at path kv/gemini using vault-mcp-server. Then create a secret named foo with key bar and value baz*

**prompt-2:** *List all secrets and their values under the mount kv/gemini*

Notice, how it uses the tools to carry out actions in response to my prompts. For the first prompt it used the `create_mount` tool to create the KV secret engine and then used the `write_secret` to create the secret. For the second prompt it used the `list_secrets` and `read_secret` to read the secret back. 

Upon review, the agent’s actions were aligned with my expectations.

```bash
vault secrets list
vault kv list -mount=kv/gemini
vault kv get -mount=kv/gemini foo
```
<script src="https://asciinema.org/a/9CT9YgVNqUANERTj4MqEjcg1o.js" id="asciicast-9CT9YgVNqUANERTj4MqEjcg1o" async="true"></script>

### Conclusion

- An AI agent connected to an MCP server can only perform the actions that are exposed to it through the server’s available tools.
- If the server does not expose a capability the agent cannot execute that action.
- MCP can be inefficient as it tries to load all tool definition into the LLM context window consuming substantial tokens and increasing costs. Below is the usage gragh for the prompts I made in this demo.

![Gemini token usage]({{ "/assets/images/gemini-token-usage.png" | relative_url }})


[vault-mcp]: https://developer.hashicorp.com/vault/docs/mcp-server/overview
[install-vault]: https://developer.hashicorp.com/vault/install#linux
[mkcert]: https://github.com/FiloSottile/mkcert/releases
[install-nodejs]: https://nodejs.org/en/download
[ai-studio]: https://aistudio.google.com/
