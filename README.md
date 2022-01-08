# Blog Ping Id

This project provides a sample to integrate PingAccess and PingFederate to be used as auth resource server and provider
with your services. This is using Java Spring Boot services but can be used with services written in any other language
like nodeJS.

# PingAccess and PingFederate

## Starting the containers

PingAccess and PingFederate both are provided via docker containers. In order to start them, run `docker-compose up -d`.
You will need to create a devops account on pingId to get the credentials. Once you have the credentials, you can either
use `ping-devops` utility to provide them OR pass them as `.env` file to `docker-compose up -d` command.

If using `ping-devops` utility, then spin up the containers
via, `docker-compose --env-file ${HOME}/.pingidentity/devops up -d`.

If using environment variables, then create a `.env.local` file and add the following content,

```
PING_IDENTITY_DEVOPS_USER=<user name>
PING_IDENTITY_DEVOPS_KEY=<user key>
```

and then run the command, `docker-compose --env-file .env.local up -d`

After the containers are started you can use the following credentials to login to the admin console. For pingaccess,
use `https://localhost:9000`. For pingfederate, use `https://localhost:9999/pingfederate/app`. The user credentials
are: `administrator/2FederateM0re`.

# Setup

So, both PingAccess and PingFederate comes pre-configured with some things. For our setup we will make some changes to
that and also add some of our own.

## PingFederate

⭐️ *Signing Certificate* -- First thing you will need is a signing certificate for your access token.

1. Go to Security > Signing & Decryption Keys & Certificates
2. Create New
    - Name: `Todo Cert`
    - Subject Alternate Names: `DNS Name -- host.docker.internal`
    - Organization: `any name`
    - Country: `US`
3. Create New
    - Name: `Tweet Cert`
    - Subject Alternate Names: `DNS Name -- host.docker.internal`
    - Organization: `any name`
    - Country: `US`
  
---

⭐️ *Access Token Manager* -- In order to generate token, you will need to define a access token manager.

1. Go to Applications > OAuth > Access Token Management
2. Create New Instance
    >*Enter an Access Token Management Instance Name and ID, select the plugin Access Token Management Type, and a parent if applicable. The types available are limited to the plugins currently installed on your server.*
    - a. Type
        - Instance Name: `Todo Token Management`
        - Instance Id: `todotokenmgmt`
        - Type: `JSON Web Tokens`<br><br>
        
    >*Complete the configuration necessary to issue and validate access tokens. This configuration was designed into, and is specific to, the selected Access Token Management plugin. <br>A JSON Web Token (JWT) Bearer Access Token Management Plug-in that enables PingFederate to issue (and optionally validate) cryptographically secure self-contained OAuth access tokens.*
    - b. Instance Configuration
        - Certificates > Add a new row to 'Certificates' *[A group of certificates and their corresponding public/private key pairs for use with signatures]*
            - Key ID: `todokey`
                >*An identifier for the given key.*
            - Certificate: `Todo Cert`
                >*Requires an EC key or RSA key length of at least 2048 bits.*
            - Click **Update**
        - JWS Algorithm: `RSA Using SHA-256`
            >*The HMAC or signing algorithm used to protect the integrity of the token. For HMAC, the active symmetric key must be selected below. **For RSA or EC, the active signing certificate must be selected.** Integrity protection can also be achieved using symmetric encryption, in which case this field can be left unselected.* 
        - Active Signing Certificate Key Id: `todokey`
            >*The Key ID of the key pair and certificate to use when producing JWTs using an RSA-based or EC-based algorithm.*
        - Click **Show Advanced Fields**
            - Issuer Claim Value: `https://pingfederate:9031`
                >*Indicates the value of the Issuer (iss) claim in the JWT (omitted, if blank).*
            - Audience Claim Value: `:8082`
                >*Indicates the value of the Audience (aud) claim in the JWT (omitted, if blank).*
            - JWKS Endpoint Path: `/todoauthtoken/jwks` -- This needs to be unique for each token manager.
                >*Path on the PingFederate server to publish a JSON Web Key Set with the keys/certificates that can be used for signature verification. Must include the initial slash (example: `/oauth/jwks`). The resulting URL will be `https://<pf_host>:<port>/ext/<JWKS Endpoint Path>`. If specified, the path must be unique across all plugin instances, including child instances.*

    <br>
    
    >*Provide the names of the attributes that will be carried in (or referenced by) the OAuth access token. For auditing purposes, an attribute may be chosen as the subject.*
    - c. Access Token Attribute Contract
        - Add a new attribute to extend the USER_KEY subject.

    <br>
    
    >*Specify a list of base resource URI's which can be used to select this Access Token Management instance.*
    - d. Under Resource URIs, add a base resource uri, `https://:3000`<br><br>

3. Create New Instance
    - a. Type
        - Instance Name: `Tweet Token Management`
        - Instance Id: `tweettokenmgmt`
        - Type: `JSON Web Tokens`<br><br>
    - b. Instance Configuration
        - Certificates > Add a new row to 'Certificates'
            - Key ID: `tweetkey`
            - Certificate: `Tweet Cert`
            - Click **Update**
        - JWS Algorithm: `RSA Using SHA-256`
        - Active Signing Certificate Key Id: `tweetkey`
        - Click **Show Advanced Fields**
            - Issuer Claim Value: `https://pingfederate:9031`
            - Audience Claim Value: `:8082`
            - JWKS Endpoint Path: `/tweetauthtoken/jwks` -- This needs to be unique for each token manager.

    <br>
    
    - c. Access Token Attribute Contract
        - Add a new attribute to extend the USER_KEY subject.

    <br>

    - d. Under Resource URIs, add a base resource uri, `https://:3000`

---

⭐️ *Access Token Mapping* -- This is needed to fulfill the access token attribute contracts.

>*Manage the attribute mapping(s) used to fulfill the access token attribute contracts. This configuration maps from a persistent grant or other sources into the access token attribute contract. For mappings involving a persistent grant, a default mapping should be configured for each access token manager. The default can be overridden based on the context of the authentication event of the original grant.*
1. Go to Applications > OAuth > Access Token Mappings
    - From Context dropdown, select Client Credentials and Access Token Manager as one of the token managers created above.
      - 1st time: 
          - Context: `Client Management`
          - Access Token Manager: `Todo Token Management`
          - Click **Add Mapping**
      - 2nd time: 
          - Context: `Client Management`
          - Access Token Manager: `Tweet Token Management`
          - Click **Add Mapping**
    - Under Contract Fulfillment, select Context and ClientId in the respective dropdowns.
      - 1st/2nd time: 
          - Source: `Context`
          - Vaule: `Client ID`

---

⭐️ *Clients* -- Now comes the client creation to be used by the services to get the access tokens.

1. Go to Applications > OAuth > Clients 
>*Manage OAuth clients (also known as applications)*
2. Add Client
    >Manage the configuration and policy information about a client.
    - Client Id: `todo_client`
    - Name: `Todo Client`
    - Client Authentication: `Client Secret`
    - Client Secret: Either use Generate Secret button OR enter one.
    - Request Object Signing Algorithm: `RSA Using SHA-256`
    - JWKS Url: `https://localhost:9031/ext/todoauthtoken/jwks`
    - Allowed Grant Types: `Client Credentials`
        - You can also select `Access Token Validation (Client is a resource server)` if you will be using this client id for pingaccess to access pingfederate.
    - Default Access Token Manager: `Todo Token Mgmt`
2. Add Client
    - Client Id: `tweet_client`
    - Name: `Tweet Client`
    - Client Authentication: `Client Secret`
    - Client Secret: Either use Generate Secret button OR enter one.
    - Request Object Signing Algorithm: `RSA Using SHA-256`
    - JWKS Url: `https://localhost:9031/ext/tweetauthtoken/jwks`
    - Allowed Grant Types: `Client Credentials`
        - You can also select `Access Token Validation (Client is a resource server)` if you will be using this client id for pingaccess to access pingfederate.
    - Default Access Token Manager: `Tweet Token Mgmt`

---
---
## Pingaccess

⭐️ *Token Provider* -- This is already set by default 
>*Configure connections between PingAccess and PingFederate*.

1. Go to Settings > System > Token Provider.
>Select to send the URI the user requested as the 'aud' OAuth parameter for PingAccess to use to select an Access Token Manager. If this option is enabled and no matching Access Token Manager exists, PingFederate returns a 401 error to the browser. If cleared, the default Access Token Manager is used.
2. Only change you will need to do is select `Send Audience` under `OAuth Resource Server`
    - If you want to use a different client id then update the id and secret.

---

<!-- ⭐️ *Virtual Host* -- This is needed to access the application via pingaccess.
>*These are the virtual host names and ports from which PingAccess accepts requests (e.g., myhost.com:80). A wildcard (\*) is used to accept any host with the specified port (e.g., *:3000).*
1. Go to Applications > Applications > Virtual Hosts
2. Add Virtual Host
    >Enter the host name for the Virtual Host, e.g. myHost.com. You can use a wildcard (*) to indicate that any host name is acceptable.
    - Host: Use the same one as used on pingfederate's *Access Token Manager*'s Resource URI tab. Ex. `https://`.
    >Enter the integer port number for the Virtual Host, e.g 1234.
    - Port: 3000 -- This is the default pingaccess application port. It can be configured on startup to a different one. -->

---

⭐️ *Sites* -- Add your API information.

1. Go to Applications > Sites
2. Add Site
    - Name: `Todo API`
    - Targets: `host.docker.internal:8082`
      - Enter one or more hostname:port pairs for the site.
    - Secure: `Yes`
      - Select if the site is expecting HTTPS connections.
3. Add Site
    - Name: `Tweet API`
    - Targets: `host.docker.internal:8082`
    - Secure: `Yes`

---

⭐️ *Application* -- Add your Application information.

1. Go to Applications > Applications
2. Add Application
3. Name: *Todo App*
4. Context Root: */todos*
5. Virtual Hosts: Use the one created under *Virtual Host* step.
6. Application Type: *API*
7. SPA Support: *Checked*
8. Access Validation: *Token Provider*
9. Destination: *Site*
10. Site: Select the site created under *Sites* step.
11. Require Https: *Checked*
12. Save
13. Repeat the same for Tweet App.

#### Certificate

Since both PA and PF are running inside the docker container over https, for the spring boot application to access these
we need to add the certificates to the java trust store. In order to do so, get the certificate by running the following
command,

```
// For PingAccess
openssl s_client -showcerts -connect localhost:3000
```

Copy the content between `-----BEGIN CERTIFICATE-----` and `-----END CERTIFICATE-----` inclusive in a file and then
import that cert file to java trust store by running the command,

```
sudo keytool -import -alias pingaccess-docker -keystore /path/to/java/security/cacerts -file /path/to/cert/file/created/above
```

Do the same for PingFederate certificate. To get the certificate,

```
// For PingFederate
openssl s_client -showcerts -connect localhost:9031
```

#### Host File

Open your hosts file (/etc/hosts) and add the following,

```
127.0.0.1 pingfederate <...virtual hosts defined on pingaccess>
```

#### Logs

The logs for PA and PF both are available under the same location `/opt/out/instance/log` in the respective containers.

## Local Certificate

In order to run the services over https, you will need to install certificate. For that you can make use of `mkcert`.

### Installing `mkcert`

```
brew install mkcert
```

### Creating certificate using `mkcert`
- `mkcert -install`
  ```
  Created a new local CA 💥
  Sudo password:
  The local CA is now installed in the system trust store! ⚡️
  The local CA is now installed in Java's trust store! ☕️
  ```
- `mkcert -pkcs12 localhost 127.0.0.1 ::1 host.docker.internal`
  ```
  Created a new certificate valid for the following names 📜
  - "localhost"
  - "127.0.0.1"
  - "::1"
  - "host.docker.internal"

  The PKCS#12 bundle is at "./localhost+3.p12" ✅

  The legacy PKCS#12 encryption password is the often hardcoded default "changeit" ℹ️

  It will expire on DAY MONTH YEAR 🗓
  ```

This will create and install the certificate in the trusted keystore.

### Using the certificate

Now create a directory in the root folder, `certs` and copy the generated certificate file to this location. Now when
you start any of the services in this project they will be able to run on `https`.
- `mdkir /Users/USER_NAME/certs && cp ./localhost+3.p12 /Users/USER_NAME/certs`

## Project Setup

This project contains 3 spring-boot services.

- *bff-service* -- This is an aggregator service like a client accessing the other services over pingaccess.
- *todo-service* -- Serivce providing APIs for Todo.
- *tweet-service* -- Serivce providing APIs for Tweet.

### BFF Service Config

The config can be found under `bff-service/src/main/resources/application.yml`. The `auth` segment contains the token
endpoint for pingfederate. The `audience`, `clientId` and `clientSecret` are provided under individual service config
segments like `todo` and `tweet`.

### Start the services

Before starting the todo and tweet service, add a file named `application-local.yml` under `src/main/resources` with
following contents,

```yaml
trust-store:
  path: <absolute path for java trust store>
  password: <trust store password>
```

Either you can IDE to start all the services OR can start them individually from command line using the
command `gradle bootRun`.

### Usage

Once the services have started, you can access it on, `https://localhost:8084/bff`. When accessing for the first time,
you will see a response like this,

```json
{
  "todos": [],
  "tweets": []
}
```

To add a new todo, make the request to `https://localhost:8084/bff/todo` and in the body provide a json string. Similar
API `/bff/tweet` to add a new tweet.
