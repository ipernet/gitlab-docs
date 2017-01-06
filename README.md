Installing a Docker private registry for GitLab
===================

This article provides detailed instructions to install a Docker Registry on your **GitLab from sources** setup.

Indeed, as of now, the [official documentation](https://docs.gitlab.com/ce/administration/container_registry.html#enable-the-container-registry) for this case needs more love :

> **Installations from source**

> If you have installed GitLab from source:

> - You will have to install [Registry](https://docs.docker.com/registry/deploying/) by yourself.

> ...

> At the absolute minimum, make sure your Registry configuration has *container_registry* as the service and *https://gitlab.example.com/jwt/auth* as the realm:
> ```
> auth:
>   token:
>     realm: https://gitlab.example.com/jwt/auth
>     service: container_registry
>     issuer: gitlab-issuer
>     rootcertbundle: /root/certs/certbundle
> ```

Ok.

¯\_(ツ)_/¯

So here what we are gonna do to make things better:

> 1. **Install a secure Docker Private Registry running on a separate domain.**
> - Running on the same box than your GitLab Instance.
> 2. **Understand what are the above settings**.
> - We will dig about the **Token Authentication** method used by **Docker Registry v2** where your GitLab instance acts as the **Authorization Service** and **token issuer**.

Installing the Registry
-------------

First, let's define a couple of prerequisites, constants and goals here:

> **GitLab instance**
>
>  - Assumed up and running at https://gitlab.example.com
>  - Up to date ([v8.8+](https://gitlab.com/gitlab-org/gitlab-ce/merge_requests/4040))
>  - Using [HTTPS](https://gitlab.com/gitlab-org/gitlab-ce/blob/master/doc/install/installation.md#using-https)
 
 And
 
> **Registry**
>
>- To be Installed at https://registry.example.com.
>  - Docker needs to be [installed](https://docs.docker.com/engine/installation/) on the running box.
>- **Storage** : Basic. 
>  - Your registry data will be persisted thanks to a Docker Volume on the host filesystem.
>  - You can later use more [advanced storage drivers](https://docs.docker.com/registry/storage-drivers/) such as an [Amazon S3 bucket](https://docs.docker.com/registry/storage-drivers/s3/).
>- **Authentication Mecanism**: [Token based](https://docs.docker.com/registry/spec/auth/token/). 
>  - This is one of the autentication mecanism available in Docker Registry and the one GitLab  [supports](https://gitlab.com/gitlab-org/gitlab-ce/blob/v8.15.2/spec/features/container_registry_spec.rb#L14).
>- **HTTPS** enabled.

 

> **Notes:** 
> 
>- A **Docker Registry** is a system that stores **Docker Images**.
>-  It provides a Web API to authenticate and manage the images (so **HTTP(s) protocol**).
 
### Authentication mecanism : Tokens

First thing first, we need to understand the basics of the authentication mechanism that will be used between your **GitLab instance** and your *soon-to-be-installed* **Docker Registry**. 

A Docker Registry supports very few [authentication mecanisms](https://docs.docker.com/registry/configuration/#auth), the first one beeing the [HTTP Basic auth](https://docs.docker.com/registry/configuration/#htpasswd) mecanism, centralized, and the other one [Token-based authentication mecanism](https://docs.docker.com/registry/spec/auth/token/), a decentralized system.

Let's introduce the latter more properly:

> The **Token-based authentication** allows you to **decouple** the authentication system **from** the registry. 
> It is an established authentication paradigm with a high degree of security.[^token-auth]

This nice intro gives **important clue** on how the system works: 

<i class="icon-right-circled"></i> The system responsible of the authentication part **is not** the registry itself, but an **external system**.

And in our case, this system will be *your* **GitLab instance**.

<i class="icon-angle-right"></i>It makes sense, because this is where is managed all the accounts that will access the registry.


> **Note:**
>
In the case of the [HTTP basic auth](https://docs.docker.com/registry/deploying/#/native-basic-auth) authentication mecanism, the Registry will **itself** checks if there is a valid *username/password* provided in the incoming HTTP requests against a provided *password file*. There is no external authentication system in this mecanism, which makes it irrelevant to use with GitLab.

#### A dialog with the Registry

Before going any further in the requirements setting up the token-based mecanism between GitLab and the Registry, let's see what is going to happen **once you will have a working** GitLab and Registry set up.

For that, read the [announcement of GitLab Registry](https://about.gitlab.com/2016/05/23/gitlab-container-registry/) up to the **Start using it** section below:

> **Start using it**
>
> First, ask your system administrator to enable GitLab Container Registry following the [administration documentation](http://docs.gitlab.com/ce/administration/container_registry.html).

> After that, you will be allowed to enable Container Registry for your project.

>- To start using your brand new Container Registry you first have to login:
> ```
> docker login registry.example.com
> ```
>- Then you can simply build and push images to GitLab:
> ```
> docker build -t registry.example.com/group/project .
> docker push registry.example.com/group/project
> ```

Let's dig here into the dialog between the user typing the command, the Registry and GitLab.

First, let's start with a user directly issuing the command push, without initial **login**.
```
$ docker push registry.example.com/group/project

The push refers to a repository [registry.example.com/group/project]
a02596fdd012: Preparing
denied: access forbidden
```

As one may expects, this fails. Let's have a look at the (simplified) behind the scene dialog of what happened here:

> **Docker utility speaking:**

>- There is no previous **login data** saved (like a *Bearer Token* or *HTTP Basic Auth* credentials).
>- I have no authentication data to provide to the Registry the push refers to.
>- I issue a HTTP call with no **Authorization: header** to know more about the authentication needed by the registry:
> ```
> GET https://registry.example.com/v2/
> ```

> **The Registry speaking:**
> 
>- Ah, an incoming HTTP request.
>- Wait, I'm setup to use a **Token based Authentication**, but there is no **Authorization request header** in this request.
>  - I was indeed expecting the following request header in the request, but found nothing:
> ```
> Authorization: Bearer <token>
> ```
> - Let's reply with a **401 Unauthorized**, but be kind enough to indicate how one needs to authenticate with me next time: 
> ```
> HTTP/1.1 401 Unauthorized

> Www-Authenticate:  Bearer    realm="https://gitlab.example.com/jwt/auth",service="container_registry"

> Content-Type:      application/json; charset=utf-8

> {
>     "errors": [
>         {
>             "code": "UNAUTHORIZED",
>             "message": "authentication required"
>         }
>     ]
> }
> ```
> 
> **Docker utility speaking:**
> 
>- Booo, a **HTTP/1.1 401 Unauthorized** response from the Registry.
>- ah, wait, there is an **Authentication challenge** in the response.
>  - The  ```Www-Authenticate: Bearer realm="" ...``` response header above.
>- This header gives me more info now:
>  - The registry uses a **Token Authentication** method (```Bearer``` scheme) 
>   - It wants a token issued by the https://gitlab.example.com/jwt/auth server (```realm=```)
>- Let's call this token server, I want a token to *push* and *pull* any image of the project *group/project* (= the *scope*) to the registry (this is what my user commands me):
> 
> ```
> GET https://gitlab.example.com/jwt/auth?scope=repository:group/project:push,pull&service=container_registry
> ```
> **GitLab speaking:**
> 
>- Ah, an incoming HTTP request requesting a **Bearer token** for a specific **scope**.
>- Wait, it's [missing](https://docs.docker.com/registry/spec/auth/token/#requesting-a-token) **account** and **client_id** params and a proper **Authorization:    Basic** request header. 
>- Who is this GitLab user? No idea, stop here.
> ```
> HTTP/1.1 403 Forbidden

> Content-Type:    application/json; charset=utf-8
> {
>     "errors": [
>         {
>             "code": "DENIED",
>             "message": "access forbidden"
>         }
>     ]
> }
> ```
> 
> **Docker utility speaking:**
> 
> - Not my lucky day, a **HTTP/1.1 403 Forbidden** from the Authorization server.
>  - There is no more I can do, I have no saved credentials of any sort, my human needs to feed me properly.


That's why the user gets:

```
$ docker push registry.example.com/group/project

The push refers to a repository [registry.example.com/group/project]
a02596fdd012: Preparing
denied: access forbidden
```

And that's why it is required to *login* to the registry first:

```
$ docker login registry.example.com
Username:
Password:
```

This time, the server issuing tokens (GitLab) will receive some account information about **the GitLab user** trying to get a token for use with the registry, thanks to a username and password provided, sent in a **Authorization: basic** request header to https://gitlab.example.com/jwt/auth.

If the provided credentials are those of a valid GitLab account, GitLab will reply to the request with a **Bearer token** that the client (here the Docker login utility) should [store for a future usage](https://docs.docker.com/engine/reference/commandline/login/#/credentials-store):
```
{
    "token": "eyJ0eXAiOiJKV1QiLCJhbGciOiJSUzI1NiIsImtpZCI6IldVT006VFpRWTpRTE1LOjRET1Y6VVhIRDpRVVE0OktaVEw6NElGVzpTSzVLOk5BQlo6TlJDSDpJQVlCIn0.YYYYYYYYYYY.XXXXXXXXX"
}
```

> At that moment, any further use of **docker push/pull** command related to the registry will include the stored token in the HTTP requests issued to the registry, as an **Authorisation: Bearer HTTP header**:
> ```
> Authorization: Bearer <the_token>
> ```

Now that the roles of everyone should be clearer, let's examine the token GitLab has issued.

This is a [JSON Web Token](https://www.sitepoint.com/introduction-to-using-jwt-in-rails/) (hint: *jwt/auth*  realm), so let's inspect the **[header](https://www.sitepoint.com/introduction-to-using-jwt-in-rails/)** of this token.

```
base64URLDecode("ey...n0") :

{"typ":"JWT","alg":"RS256","kid":"ZZZZZZ"}
```

This JWT has a signature (the XXXXXXX part above) created using the [RSA-SHA256 algorithm](https://jwt.io/#debugger-io) . 

> **It means 4 things:**
> 
>- There is no *secret*, but a **RSA key pair** shared between GitLab and the Registry to "sign/verify" the tokens.
>  - Otherwise the [HS156](https://jwt.io/#debugger-io) algo would have been used.
> - GitLab, the issuer of the tokens, can only **sign** those with a given **RSA private key**.
> - The **Registry** must have access to the **RSA public key** to **verify** signatures of the issued tokens.
> - We will thus need to generate a **RSA key pair** in our *GitLab+Registry* setup, sooner or later.

We now have all the information needed to understand how GitLab and the Registry will work together.


### Installing the Registry
####  On the same box than GitLab, on a separate domain.

> **What you need to be ready:**
> 
>- Registry domain DNS and HTTPS up: 
>  - [https://](https://gitlab.com/gitlab-org/gitlab-ce/blob/master/lib/support/nginx/registry-ssl)registry.example.com 
>- The **TLS termination point** of your registry is the **host NGINX ** (shared with GitLab).
>  - Use the official [registry-ssl](https://gitlab.com/gitlab-org/gitlab-ce/blob/master/lib/support/nginx/registry-ssl) config, nothing to change here.

> **What the registry-ssl config implies:**

> - A **single entry point** for your registry, using **HTTPS** only.
>  - This entry point is  https://registry.example.com. 
>       - It flows through your host NGINX .
>       - Your registry container HTTP server **will not** be the TLS termination point. 
>          - So  it will use plain [HTTP](https://gitlab.com/gitlab-org/gitlab-ce/blob/master/lib/support/nginx/registry-ssl#L50), only **accessible locally** at **localhost:5000**.

> **What the config doesn't say:**

>- Your registry SHOULD have no direct access from the outside other than through the host NGINX at https://registry.example.com
>- No bypass should be possible at http://registry.example.com:5000 or whatever.

**Setup time, on the Gitlab Box:**

```
$ sudo mkdir ~/docker-registry
$ cd docker-registry
```

- **Generate a new key pair that will be needed for the authentication mecanism:**
 >**Reminder**: 
 
 >- We generate a key pair for GitLab to **sign tokens** with the **private key** and for the Registry to **verify** those signatures with the **public key**.
 >- It does not matter if the certificate is **self-signed** here!
```
$ sudo openssl req -new -newkey rsa:4096 -subj "/CN=gitlab-issuer" -nodes -x509 -keyout auth.key -out auth.crt

$ sudo chmod 400 auth.key
```
- **Create a basic configuration file for your Registry**
 - As said we will use a basic **storage driver** for now, the **host file system**.
 - There are of course [many extra options](https://docs.docker.com/registry/configuration/) possible for your Registry.
```
$ sudo touch server-registry.yml
```

```
version: 0.1
log:
  level: info
  formatter: text
  fields:
    service: registry
storage:
  cache:
    blobdescriptor: inmemory
  filesystem:
    rootdirectory: /var/lib/registry
http:
  addr: localhost:5000
  headers:
    X-Content-Type-Options: [nosniff]
auth:
  token:
    realm: https://gitlab.example.com/jwt/auth
    service: container_registry
    issuer: gitlab-issuer
    rootcertbundle: /root/certs/auth.crt
```

We configure the **Authentication mecanism** for the registry to be **Token based**  (```auth: token:```) with the [GitLab parameters](https://gitlab.com/gitlab-org/gitlab-ce/blob/8-15-stable//app/services/auth/container_registry_authentication_service.rb#L20-24) [needed](https://docs.docker.com/registry/configuration/#token) for this mecanism to work.

Among the settings there is:
> **rootcertbundle**: The absolute path to the root certificate bundle. This bundle contains the public part of the certificates used to sign authentication tokens[^token-auth].

<i class="icon-right-circled"></i> **This is here the openssl-generated** *auth.crt* file. 

The path */root/certs/auth.crt* is relative to the Registry container so we are gonna map our host file to the expected */root/certs/auth.crt* container file thanks to a Docker Volume:

```
# docker run -d -p 127.0.0.1:5000:5000 --restart=always --name registry \
  -v /root/docker-registry/server-registry.yml:/etc/docker/registry/config.yml \
  -v /root/docker-registry/conf/auth.crt:/root/certs/auth.crt \
  -v /home/git/gitlab/shared/registry/:/var/lib/registry \
  registry:2
```

> **Notes**

>- We ensure the Registry will not be accessible on port 5000 from the outside :
> ```
> -p 127.0.0.1:5000:5000
> ```
>- We map the configuration file of the Registry that we created on the host to the expected */etc/docker/registry/config.yml* file in the container:
> ```
> -v /root/docker-registry/server-registry.yml:/etc/docker/registry/config.yml
> ```
> - GitLab and the Registry must share the [container registry storage path](http://docs.gitlab.com/ce/administration/container_registry.html#container-registry-storage-path) when the host filesystem is used so we add this extra volume:
> ```
> -v /home/git/gitlab/shared/registry/:/var/lib/registry
> ```
 
 - **Check the Registry has properly started:**
```
# docker logs registry
```


### Configure GitLab

Follow the [Official documentation](http://docs.gitlab.com/ce/administration/container_registry.html#configure-container-registry-under-its-own-domain), which I would complete as follow:

```
  registry:
    enabled: true
    host: registry.example.com
    api_url: http://localhost:5000/
    key: config/registry-auth.key
    path: shared/registry
    issuer: gitlab-issuer
```



-  ```api_url: http://localhost:5000/```

> Internal address to the registry, will be used by GitLab to directly communicate with API.
> 
> <i class="icon-right-circled"></i> Your registry is indeed accessible locally at **http://localhost:5000**, so there is no need for GitLab to connect to the TLS enabled URL at https://registry.example.com. See it as a shortcut.

- ```key: config/registry-auth.key```

> This is the location relative to GitLab directory where is the RSA private key. 
> 
> <i class="icon-right-circled"></i>  You thus need to copy the previously created key to where GitLab expects it:
> ```
> $ sudo cp -p /root/docker-registry/auth.key /home/git/gitlab/config/registry-auth.key
> 
> $ sudo chown git:git /home/git/gitlab/config/registry-auth.key
>```

You can now **restart GitLab**.

### Test
- **Try to login to your registry using one of your GitLab account:**

The following [commands lead to the same](https://github.com/docker/docker/blob/8874f80e67c560f44322233bfc22ecd86b85e9e2/registry/auth.go#L212):
```
$ sudo docker login https://registry.example.com
$ sudo docker login registry.example.com
```
 
 And this should work too:
```
$ sudo docker login http://localhost:5000
```

If you get a **Login succeeded**, you know have a working GitLab Registry setup.



----------
  [^token-auth]: https://docs.docker.com/registry/configuration/#/token
