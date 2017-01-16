GitLab Auto Deploy: OpenShift Origin introduction 
===================


This guide is the ***from sources*** documentation of the [Auto-Deploy video released by GitLab earlier this month](https://about.gitlab.com/2016/12/22/gitlab-8-15-released/).

We indeed already have a documentation from installing [GitLab from sources](https://gitlab.com/gitlab-org/gitlab-ce/blob/master/doc/install/installation.md), [another one](https://m42.sh/gitlab-registry.html) for setting up [Container Registry](https://gitlab.com/help/administration/container_registry.md), now let's focus on how to install a simple **OpenShift Origin** instance as used in the demo.

**We will cover what is shown in the video and a bit more as well:**

- (*Demo*) Install OpenShift Origin on https://openshift.example.com:8443/
  - (*Demo*) Use a valid TLS certificate for your OpenShift Origin console.

- (*More*) Setup users accesses to OpenShift Origin using GitLab as  [Identity Provider](https://docs.openshift.org/latest/install_config/configuring_authentication.html#identity-providers).

- (*More*) Support **non-public GitLab projects** and secure deployments to your OpenShift Origin when running GitLab CI auto-deploy.

The setup will be done on Debian Jessie on a dedicated server, directly connected to the internet.

***Notes:***
> You can use the GNU/Linux distributions of your choice.

**OpenShift Origin cluster:**

We will, as a demo and introduction to OpenShift Origin, install a **all-in-one** cluster with a master and a node **running on the same machine**.

**Prerequisites:**
> - A Linux Machine
>   - Accessible on public IP X.X.X.X
> - A working **GitLab 8.15+ installation**
>   - For the demo, refered to as https://gitlab.example.com
> - A working, secure, **GitLab Container Registry** installation
>  - For the demo, refered to as https://registry.example.com
>   - A [GitLab Runner](https://gitlab.com/gitlab-examples/openshift-deploy#requirements) using [**Docker executor**](https://docs.gitlab.com/ce/ci/docker/using_docker_build.html#use-docker-in-docker-executor) with **privileged mode** enabled.

##Requirements
###Kernel

The GitLab Auto Deploy demo uses a [specific Docker image](https://gitlab.com/gitlab-examples/openshift-deploy) to build and deploy a simple ruby on rails application.

Running this image requires your Kernel to support the [Overlay file system](https://gitlab.com/gitlab-examples/openshift-deploy/blob/master/.gitlab-ci.yml#L5) introduced in Kernel **3.18**. If you are not running **Kernel >= 3.18**, upgrade your kernel first.

> $ uname -r
> 3.16.0-4-amd64

One way of [doing this](https://wiki.debian.org/HowToUpgradeKernel) on Debian Jessie is to add the Jessie-backports repository and use a recent Kernel from there:

> ```$ echo "deb http://httpredir.debian.org/debian jessie-backports main" >> /etc/apt/sources.list```

> ```$ sudo apt update```
> ```$ sudo apt-get install -t jessie-backports linux-image-amd64```
> ```$ sudo reboot```

Once rebooted, load the `overlay` kernel module:
> ```$ sudo modprobe overlay```

###Docker

- You need to have Docker running on the host to run, as the single node of our cluster, [OpenShift Pods](https://docs.openshift.com/enterprise/3.0/architecture/core_concepts/pods_and_services.html#pods).

- You do **not need** to start the Docker deamon with specific settings.

> ***Notes:***

> - We will not run the OpenShift daemon in a Docker container, as [Debian is unsupported](https://docs.openshift.org/latest/getting_started/administrators.html#running-in-a-docker-container).
> - We are not using here the [Integrated OpenShift Origin Registry](https://docs.openshift.org/latest/install_config/registry/index.html#about-the-registry) but the GitLab Registry.

###OpenShift configuration



- Download the latest stable release of [OpenShift Origin](https://github.com/openshift/origin/releases):

> ```$ sudo mkdir /usr/local/openshift-origin``` 
>  

> ```$ mkdir {bin,etc}```  
> ```$ wget https://github.com/openshift/origin/releases/download/v1.3.2/openshift-origin-server-v1.3.2-ac1d579-linux-64bit.tar.gz```
> 
> ```$ echo 'd84852af7cc8c2de21b566286667c7850415d23f1d007e612c73c04f276c8bc4  openshift-origin-server-v1.3.2-ac1d579-linux-64bit.tar.gz' | shasum -a256 -c - && sudo tar -C /usr/local/openshift-origin/bin --strip-components 1 -xzf openshift-origin-server-v1.3.2-ac1d579-linux-64bit.tar.gz```

**Notes:** 

> - There are 2 spaces between "`...76c8bc4  openshift-...`"

- Remove the downloaded release:

> ```$ rm openshift-origin-server-v1.3.2-ac1d579-linux-64bit.tar.gz ```
 
- Add the directory you untarred the release into to your path:

> ```$ export PATH="$(pwd)/bin":$PATH```

A manual configuration of OpenShift is a tedious process, so we are going to generate a default configuration for a  [all-in-one](https://docs.openshift.org/latest/getting_started/administrators.html#installation-methods) server and work from it:

> ```$ sudo openshift start --write-config=/usr/local/openshift-origin/etc```

- Generate the `master-config.yaml` file too:

> ```$ ./bin/openshift start master --write-config=/usr/local/openshift-origin/etc/master
> Wrote master config to: usr/local/openshift-origin/etc/master```

You now have a complete [all-in-one](https://docs.openshift.org/latest/getting_started/administrators.html#installation-methods) basic configuration in `/usr/local/openshift-origin/etc`.

### Custom TLS certificate

- **Setup your DNS** so that `openshift.example.com` (change accordingly) points to the public IP of your server (X.X.X.X).

> ***Notes:***
> 
>- We do keep things simple here: no round robin DNS or load balancing layers.


- Let's now ensure that the **OpenShift console** will be accessible at https://openshift.example.com:8443.
 - By default, access to the console  is at https://X.X.X.X:8443, using a self-signed certificate generated by the config.

> `$ cd /usr/local/openshift-origin`
> `$ editor ./etc/master/master-config.yml`

- **Only** [change](https://docs.openshift.org/latest/install_config/certificate_customization.html#configuring-custom-certificates):

```
assetConfig:
  masterPublicURL: https://X.X.X.X:8443
  publicURL: https://X.X.X.X:8443/console/

masterPublicURL: https://X.X.X.X:8443

oauthConfig:
  assetPublicURL: https://X.X.X.X:8443/console/
  masterPublicURL: https://X.X.X.X:8443
```

to:

```
assetConfig:
  masterPublicURL: https://openshift.example.com:8443
  publicURL: https://openshift.example.com/console/
  
masterPublicURL: https://openshift.example.com:8443

oauthConfig:
  assetPublicURL: https://openshift.example.com:8443/console/
  masterPublicURL: https://openshift.example.com:8443
```

**Notes:** 

> - Use your own domain.
> - **Do not** change  the `masterUrl` setting [used internally](https://docs.openshift.org/latest/install_config/certificate_customization.html#configuring-custom-certificates) by the master and its nodes to communicate.

- **Generate a Trusted TLS certificate**

You need to generate a TLS certificate issued by a trusted authority for accessing the console at https://openshift.example.com.

> You can use [Let's encrypt](https://letsencrypt.org/) to generate one:

>  > `$ sudo apt install certbot`
>  
> > `# Ensure nothing is running on X.X.X.X:443 (stop nginx, apache2...)`
>
>  > `$ certbot certonly`
>  > `# choose "spin up a temporary web server"`
>  > `# enter "openshift.example.com" (change as needed)`

Then move the **private key** to:
 `/usr/local/openshift-origin/etc/master/openshift.example.com.key.pem` 

 and the **issued certificate** to:
`/usr/local/openshift-origin/etc/master/openshift.example.com.crt.pem`

> **Notes:**

>  Ensure your private key has correct permissions:
>  > `chmod 400 ./etc/master/openshift.example.com.key.pem`

- Now, add the **RSA Key pair** info to the master config:

> ```$ editor ./etc/master/master-config.yml```

```
assetConfig:
  servingInfo:
    namedCertificates: null # remove this
    # At the end of the section, add:
    namedCertificates:
    - certFile: openshift.example.com.crt.pem
      keyFile: openshift.example.com.key.pem
      names:
      - "openshift.example.com"

corsAllowedOrigins:
# At the end of the list, add:
- openshift.example.com:8443

oauthConfig:
  namedCertificates: null # remove this
  # At the end of the section, add:
  namedCertificates:
    - certFile: openshift.example.com.crt.pem
      keyFile: openshift.example.com.key.pem
      names:
      - "openshift.example.com"

servingInfo:
  namedCertificates: null # remove this
  # At the end of the section, add:
  namedCertificates:
    - certFile: openshift.example.com.crt.pem
      keyFile: openshift.example.com.key.pem
      names:
      - "openshift.example.com"
```

**Notes:** 

> - Use your own domain everywhere.

### GitLab Identity Provider

You don't want your OpenShift console access in the [wild](https://docs.openshift.org/latest/install_config/configuring_authentication.html#AllowAllPasswordIdentityProvider), so let's change the default Identity Provider to GitLab for conveniency.

As a **GitLab admin**, create a new Oauth Application at https://gitlab.example.com/admin/applications/new:

> **Name: ** OpenShift Origin
> **Redirect URI: ** https://openshift.example.com:8443/oauth2callback/gitlab
> **Scope:** api

Submit to get access to an **Application Id** and **secret** to use below.

- **Now edit the OpenShift Origin master config again:**

> ```$ editor ./etc/master/master-config.yml```

- **Remove the default Identity Provider**:

```
oauthConfig:
  #...
  identityProviders:
  - challenge: true
    login: true
    mappingMethod: claim
    name: anypassword
    provider:
      apiVersion: v1
      kind: AllowAllPasswordIdentityProvider
```

- **Add the [GitLab Identity Provider](https://docs.openshift.org/latest/install_config/configuring_authentication.html#GitLab) with the below info**:

```
oauthConfig:
  #...
  identityProviders:
  - name: gitlab
    challenge: true
    login: true
    mappingMethod: claim
    provider:
      apiVersion: v1
      kind: GitLabIdentityProvider
      url: https://gitlab.example.com # change as needed
      clientID: <APPLICATION_ID> # change as needed
      clientSecret: <SECRET> # change as needed
```

**Notes:** 

> - GitLab is only used here as an **Identity Provider**: 
>  - **OpenShift Origin** manages accesses and roles to a OpenShift Origin project independently of what is set on GitLab for the said project.
>  - By default, only the GitLab user whose OpenShift Origin token has been added to the **Kubernetes service** of the project on GitLab will be **admin** of this project once created by Gitlab CI in OpenShift Origin.


## OpenShift Origin Apps domain

You need a **domain** where your **OpenShift applications** will be deployed and accessible.


Let's decide  here on `*.apps.openshift.example.com`

> ***Notes:***
> 
>- We do keep things simple here: no round robin DNS or load balancing layers.

- **Define a wildcard DNS** on `*.apps.openshift.example.com` to your server public IP `X.X.X.X`

We run a `all in one` setup, so the master, console and the only node run on the same machine.


- **Now edit the master config again:**

> `$ cd /usr/local/openshift-origin`
> ```$ editor ./etc/master/master-config.yml```

- **Change:**
```
routingConfig:
  subdomain: router.default.svc.cluster.local
```

to:

```
routingConfig:
  subdomain: apps.openshift.example.com
```

**Notes:** 

> - Use your own domain everywhere.
> 
### Start OpenShift Origin
- It's now time to start your `all-in-one` OpenShift Origin cluster:

```
    $ sudo openshift start \
    	--master-config=/usr/local/openshift-origin/etc/master/master-config.yaml \
    	--node-config=/usr/local/openshift-origin/etc/node-rigel/node-config.yaml \
    	--loglevel=5
```
  

- **Open your browser and navigate to https://openshift.example.com:8443/**
  - You should have **no TLS certificate issue**.
  - You should be **redirected to GitLab** at https://gitlab.example.com/users/sign_in
 
- **Authorize OpenShift Origin** to use your GitLab account.
  - You should now be redirected to https://openshift.example.com:8443/console/


### OpenShift Origin Router

Your OpenShift applications will be at accessible at http://MY-APP.apps.openshift.example.com.
>That is to say on port `80` of IP `X.X.X.X`.

You may also want to access them using **https**, so on port `443` of IP `X.X.X.X`.

**Problem: ** 

Because we are running here a demo infrastructure in `all-in-one` mode with a single node that will be running on the machine than the master and probably lots of others things, you may have already something listening on **:80** and **:443** on your server (e.g. an **nginx, haproxy...**).

**Notes: **
>- In a production setup, you have **dedicated instances** for running OpenShift nodes.

Let's create an OpenShift router that won't conflict with your existing software stack, if any.

- First, we need to manage our running OpenShift Origin cluster:

> ```$ export KUBECONFIG=/usr/local/openshift-origin/etc/master/admin.kubeconfig```

- Let's create a [custom](https://docs.openshift.org/latest/install_config/router/default_haproxy_router.html) router not using the default port `80` and `443` of the host running its pod:

> ```$ sudo -E oadm policy add-scc-to-user hostnetwork -z router```
> 
> ```$ sudo -E oadm router router --replicas=0 \
	> --ports='10080:10080,10443:10443' \
	>--service-account=/usr/local/openshift-origin/etc/master/openshift-router.kubeconfig \
	>--service-account=router```
>   
> ```$ sudo -E ./bin/oc env dc/router \
	> ROUTER_SERVICE_HTTP_PORT=10080 \
	> ROUTER_SERVICE_HTTPS_PORT=10443```
>   
> ```$ sudo -E ./bin/oc scale dc/router --replicas=1```

**Notes: **
> We assume here that port `10080` and `10443` are available on the host, change if needed.
> 
> If you have nothing running on port `80` and `443` on your machine, just run the commands without any `--ports`/ `ROUTER_SERVICE_HTTP(s)_PORT` settings.


### Run the GitLab Auto Deploy demo

####Enable Images to Run on OpenShift Origin with USER in the Dockerfile

The demo creates a [Docker Image](https://gitlab.com/gitlab-examples/openshift-deploy/blob/master/build#L57) using [herokuish](https://github.com/gliderlabs/herokuish) that will run as a specific, [random](https://github.com/gliderlabs/herokuish/blob/5564778505b885077cfb1e421d3c0debbe0c8ecc/include/herokuish.bash#L70) **USER**, so you need to enable [this setting](https://docs.openshift.org/latest/admin_guide/manage_scc.html#enable-images-to-run-with-user-in-the-dockerfile) on your cluster.

> ```$ sudo -E oadm policy add-scc-to-group anyuid system:authenticated```

**Notes:**

> Failing to do so will prevent your ruby-demo image to be started with the following error:
> `mkdir /.basher: permission denied1`

Now, let's start the [video](https://www.youtube.com/watch?v=m0nYHPue5RU).


- At [\[0:31\]](https://youtu.be/m0nYHPue5RU?t=31), do not create a **public project**, create an **internal one**.

Continue the demo as shown, using your own **OpenShift Origin URLs**, up to [1:52](https://youtu.be/m0nYHPue5RU?t=111). 

- At [\[1:52\]](https://youtu.be/m0nYHPue5RU?t=111):

First line of `.gitlab-ci.yml`, change:

> ```image: registry.gitlab.com/gitlab-examples/openshift-deploy/```

with:

> ```image: ipernet/openshift-deploy```

Then add the following variable, below `KUBE_DOMAIN`, using your own domain:

> ```  REGISTRY_REALM: https://gitlab.example.com/jwt/auth```

**Notes:**
> Your `KUBE_DOMAIN` is `apps.openshift.example.com`

Change `example.com` with your domain.

**Why the changes?**

The project created is now an **internal** project, not a **public** one, so it creates a new requirement:

> When OpenShift Origin will have to [pull the image from your GitLab container registry](https://gitlab.com/gitlab-examples/openshift-deploy/blob/master/deploy#L41), it will faill because the image is not **public** anymore. Thus it is required to authenticate against your registry server to be able to **pull** and **deploy** images of your app.

The image `ipernet/openshift-deploy` adds [such authentication](https://gitlab.com/gitlab-examples/openshift-deploy/merge_requests/2/diffs).

**Your project should now build and be deployed on your OpenShift Origin cluster.**

It should be available at:

- http://staging.apps.openshift.example.com:10080 (if custom ports are used for router)
- http://staging.apps.openshift.example.com


### TLS

- If your machine is not dedicated for running a OpenShift Origin node as explained above and you have used specific ports for your OpenShift Origin Router (**10080**, **10443**), you may want to remove the port specification (**:10080**) for accessing your OpenShift Origin apps. Plus you may also want to support TLS on `*.apps.openshift.example.com`.

> This is not supposed to be how things are done for running a proper OpenShift Origin cluster, but in such case, you will have to setup a **reverse proxy** with TLS termination using `nginx` or `haproxy`.

- If you have started a standard OpenShift Origin router directly listening on `:80` and `:443`, check how to create [secure routes](https://docs.openshift.org/latest/dev_guide/routes.html#creating-routes).
