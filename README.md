# Jupyterhub and JKG2AT

Table of Contents
-----------------

* [Prerequisites](#prerequisites)
  * [Setup LDAP Authentication](#setup-ldap-authentication)
  * [Build the JupyterHub Docker image](#build-the-jupyterhub-docker-image)
  * [Prepare the Jupyter Notebook Image](#prepare-the-jupyter-notebook-image)
     * [Configure Jupyter Notebook Image](#configure-jupyter-notebook-image)
  * [Run JupyterHub](#run-jupyterhub)
     * [Setup SSL/TLS Client Authentication on the nb2kg/hub-notebook Images](#setup-ssltls-client-authentication-on-the-nb2kghub-notebook-images)
  * [Setup Jupyter Kernel Gateway](#setup-jupyter-kernel-gateway)
     * [Jupyter Kernel Gateway SSL/TLS and Client Auth](#jupyter-kernel-gateway-ssltls-and-client-auth)
        * [z/OS Specific Option (AT-TLS)](#zos-specific-option-at-tls)
           * [Set up AT-TLS on z/OS](#set-up-at-tls-on-zos)
           * [Define certificates for Jupyter Kernel Gateway](#define-certificates-for-jupyter-kernel-gateway)
           * [Define AT-TLS policy for Jupyter Kernel Gateway](#define-at-tls-policy-for-jupyter-kernel-gateway)
           * [Give Jupyter Kernel Gateway the proper authority](#give-jupyter-kernel-gateway-the-proper-authority)
        * [Export User Certs for Use by Jupyter Notebook](#export-user-certs-for-use-by-jupyter-notebook)
           * [Export the end user certificate](#export-the-end-user-certificate)
           * [Transfer the certificate package from z/OS to the remote Linux system that will host the end user.](#transfer-the-certificate-package-from-zos-to-the-remote-linux-system-that-will-host-the-end-user)
           * [From the remote linux system, validate the contents of the package.](#from-the-remote-linux-system-validate-the-contents-of-the-package)
           * [Extract the client certificate, the private key and the CA certificate from the p12 package to three separate files, all in PEM format.](#extract-the-client-certificate-the-private-key-and-the-ca-certificate-from-the-p12-package-to-three-separate-files-all-in-pem-format)
              * [To extract the client certificate into a pem-encoded file, issue:](#to-extract-the-client-certificate-into-a-pem-encoded-file-issue)
              * [To extract the CA certificate into a pem-encoded file, issue:](#to-extract-the-ca-certificate-into-a-pem-encoded-file-issue)
              * [To extract the private key into a pem-encoded file, issue  (you will be prompted for the password twice) :](#to-extract-the-private-key-into-a-pem-encoded-file-issue--you-will-be-prompted-for-the-password-twice-)
              * [To verify the end user certificate (once CA certificate is extracted), issue:](#to-verify-the-end-user-certificate-once-ca-certificate-is-extracted-issue)
  * [Behind the scenes](#behind-the-scenes)
     * [Create a Docker Network](#create-a-docker-network)
     * [Create a JupyterHub Data Volume](#create-a-jupyterhub-data-volume)
  * [FAQ](#faq)
     * [How can I view the logs for JupyterHub or users' Notebook servers?](#how-can-i-view-the-logs-for-jupyterhub-or-users-notebook-servers)
     * [How do I specify the Notebook server image to spawn for users?](#how-do-i-specify-the-notebook-server-image-to-spawn-for-users)
     * [If I change the name of the Notebook server image to spawn, do I need to restart JupyterHub?](#if-i-change-the-name-of-the-notebook-server-image-to-spawn-do-i-need-to-restart-jupyterhub)
     * [How can I backup a user's notebook directory?](#how-can-i-backup-a-users-notebook-directory)

-----------------

This repository provides a reference deployment of [JupyterHub](https://github.com/jupyter/jupyterhub), a multi-user [Jupyter Notebook](http://jupyter.org/) environment, on a **single host** using [Docker](https://docs.docker.com).  

This deployment:

* Runs the [JupyterHub components](https://jupyterhub.readthedocs.org/en/latest/getting-started.html#overview) in a Docker container on the host
* Uses [DockerSpawner](https://github.com/jupyter/dockerspawner) to spawn single-user nb2kg Jupyter Notebook servers in separate Docker containers on the same host
* Persists JupyterHub data in a Docker volume on the host
* Persists user notebook directories in Docker volumes on the host
* Uses [LDAPAuthenticator](https://github.com/jupyterhub/ldapauthenticator) to authenticate users


## Prerequisites

* This deployment uses Docker for all the things, via  [Docker Compose](https://docs.docker.com/compose/overview/).
  It requires [Docker Engine](https://docs.docker.com/engine) 1.12.0 or higher.
  See the [installation instructions](https://docs.docker.com/engine/installation/) for your environment.
* This example configures JupyterHub for HTTPS connections (the default).
   As such, you must provide TLS certificate chain and key files to the JupyterHub server.
   If you do not have your own certificate chain and key, you can either
   [create self-signed versions](https://jupyter-notebook.readthedocs.org/en/latest/public_server.html#using-ssl-for-encrypted-communication),
   or obtain real ones from [Let's Encrypt](https://letsencrypt.org)
   (see the [letsencrypt example](examples/letsencrypt/README.md) for instructions).
* Migrating from Jupyter notebook with nb2kg to Jupyterhub
   This process is pretty straight forward, and there aren't really any tricks to the process.
   What you do is download the files on your old Jupyter notebook server to your workstation.
   Next you start up Jupyterhub, login to your user's server, then upload your files into that environment.
   When that process is done, make sure your old Jupyter notebook server is down.

From here on, we'll assume you are set up with docker,
via a local installation or [docker-machine](./docs/docker-machine.md).
At this point,

    docker ps

should work.


## Setup LDAP Authentication

This deployment uses LDAPAuth to authenticate users.
You will need to specify a LDAP server IP, Port, Bind DN in the `.env` file.

```
# LDAP Settings
# The IP of your LDAP Server (NO QUOTES!)
LDAP_SERVER_HOST=<Hostname or IP>
# The port your LDAP Server is running on (NO QUOTES!)
LDAP_SERVER_PORT=<Port>
# The LDAP user bind
LDAP_BIND_DN=<ldap_bind>  # example 'cn={username},profiletype=USER,cn=example,cn=org'
```

**Note:** If LDAP server lives on the same machine hosting the Docker container then it can not be reached at 127.0.0.1 because that will point to the docker container. You need to use an external IP address.

**Note:** The `.env` file is a special file that Docker Compose uses to lookup environment variables.
If you choose to place the LDAP information in this file,
you should ensure that this file remains private
(e.g., do not commit the sensative information to source control).

**You may run into issues if your LDAP username contains special characters.**

## Build the JupyterHub Docker image

Configure JupyterHub and build it into a Docker image.

1. Copy the TLS certificate chain and key files for the JupyterHub server to a directory named `secrets` within this repository directory. These will be added to the JupyterHub Docker image at build time.  If you do not have a certificate chain and key, you can either [create self-signed versions](https://jupyter-notebook.readthedocs.org/en/latest/public_server.html#using-ssl-for-encrypted-communication), or obtain real ones from [Let's Encrypt](https://letsencrypt.org) (see the [letsencrypt example](examples/letsencrypt/README.md) for instructions).

    ```
    mkdir -p secrets
    cp jupyterhub.crt jupyterhub.key secrets/
    ```

**Note:** Here, the names are required to be jupyterhub, but the default is mykey.


1. Create a `userlist` file with a list of authorized users.  At a minimum, this file should contain a single admin user.  For example:

   ```
   exampleuser admin
   ```

   The admin user will have the ability to add more users in the JupyterHub admin console.

   **Note:** The userlist is added to the jupyterhub image at Docker build time.

1. Use [docker-compose](https://docs.docker.com/compose/reference/) to build the
   JupyterHub Docker image on the active Docker machine host:

    ```
    make hub_image
    ```

## Prepare the Jupyter Notebook Image

You can configure JupyterHub to spawn Notebook servers from any Docker image, as
long as the image's `ENTRYPOINT` and/or `CMD` starts a single-user instance of
Jupyter Notebook server that is compatible with JupyterHub.

To specify which Notebook image to spawn for users, you set the value of the  
`DOCKER_NOTEBOOK_IMAGE` environment variable to the desired container image.
You can set this variable in the `.env` file, or alternatively, you can
override the value in this file by setting `DOCKER_NOTEBOOK_IMAGE` in the
environment where you launch JupyterHub.

To build the default nb2kg/notebook image,

```
make notebook_image
```

**Note:** This step and the JupyterHub step can be done all at once using `make all`


### Configure Jupyter Notebook Image

In previous iterations of Jupyter/nb2kg notebooks, you configured the notebook image through a config file.  Now, because of Jupyterhub, you need to configure things at a Jupyterhub level instead of at an nb2kg image level.  This can be done by editting the `.env` file.  The following things need to be set:

```
# The following Environment Variables are used by nb2kg notebook
# if you are using any other notebook image, you may ignore these.
# Kernel Gateway Host URL:Port (NO QUOTES!)
KG_URL=https://<URL>:<Port>
# Kernel Gateway Authentication Token String
KG_AUTH_TOKEN=<Token>
# Set false if using a signed certificate
VALIDATE_KG_CERT=true
# The following are used for Jupyter Kernel Gateway/nb2kg to use
# client authentication.  These certificates must be saved in
# the individual nb2kg/hub notebook images in /etc/jupyterpki/
# Kernel Gateway Client Key Filename (NO QUOTES!)
CLIENT_KEY=key.pem
# Kernel Gateway Client Certificate Filename (NO QUOTES!)
CLIENT_CERT=cert.pem
# Kernel Gateway Certificate Authority Certificate Filename (NO QUOTES!)
CLIENT_CA=cacert.pem
```

## Run JupyterHub

Run the JupyterHub container on the host.

To run the JupyterHub container in detached mode:

```
docker-compose up -d
```

Once the container is running, you should be able to access the JupyterHub console at

```
https://myhost.mydomain
```

To bring down the JupyterHub container:

```
docker-compose down
```

**Note:** If changes are made to the .env file, you must redeploy the docker container, but a complete rebuild is not required.

### Setup SSL/TLS Client Authentication on the nb2kg/hub-notebook Images
If you are connecting to a Jupyter Kernel Gateway server that requires client authentication, you must setup `CLIENT_KEY`, `CLIENT_CERT`, and `CLIENT_CA`.  These files must be copied into the individual nb2kg/hub-notebook images into their `/etc/jupyterpki/` directories.

Before you copy the certificates into these directories, you must start the images.  This can be done in a few ways.  The first way to start a notebook image is to login as the user whose image you want started.  Another option is to login as an admin user, go into the admin interface, and then start the user notebook you want started.  Once the image is started, find the image corresponding to the user you are trying to access.  The image should have a name like jupyter-<username>.

```
docker ps
```

Now you need to copy the SSL files into that image's `/etc/jupyterpki/` folder.

```
docker cp <local_file> <container_id>:/etc/jupyterpki/
```

If you do not want to use `key.pem`, `cert.pem`, and `cacert.pem`, you can change these in the `.env` file before starting Jupyterhub.  That being said, you can't change those file names between different notebook images.  All nb2kg/hub-notebook images in the Jupyterhub must have the same client authentication file names.

Once the files have been copied, you can shutdown the user notebook image if you'd like.  This can be done as an individual user by clicking Control Panel > Stop My Server or as an admin using the admin interface.


## Setup Jupyter Kernel Gateway
Setting up Jupyter Kernel Gateway can be done simply or in very complex ways.  Lets start off with this simplest setup that can be used.  You simply install Jupyter Kernel Gateway, create a file called `~/.jupyter/jupyter_kernel_gateway_config.py` on the Jupyter Kernel Gateway host system.  The file should look like the following:

```
# Configuration file for jupyter-kernel-gateway.
c.KernelGatewayApp.allow_origin = '*'
c.JupyterWebsocketPersonality.list_kernels = True
c.KernelGatewayApp.ip = '<ip of host>'
```

### Jupyter Kernel Gateway SSL/TLS and Client Auth
If you would like to add SSL/TLS to this setup, add the following lines to the `~/.jupyter/jupyter_kernel_gateway_config.py` file.

```
c.KernelGatewayApp.certfile = "<path_to>/<cert_file>"
c.KernelGatewayApp.keyfile = "<path_to>/<key_file>"
```

With this setup, you don't need to worry about setting up cient authentication on the nb2kg/hub-notebook image, but you get the advantage of having an encrypted session between the notebook image and the Jupyter Kernel Gateway host.

To add client authentication into the mix, add the following line to that same file

```
c.KernelGatewayApp.client_ca = "<path_to>/<rootCa_file>"
```
If you are using this option, you MUST setup client authentication with the nb2kg/hub-notebook image.

#### z/OS Specific Option (AT-TLS)
z/OS has a special form of SSL/TLS called Application Transparent Transport Layer Security (AT-TLS). As the name suggests, AT-TLS provides SSL/TLS at the networking layer instead of the application layer. This means you do not need to set the `c.KernelGatewayApp.certfile`, `c.KernelGatewayApp.keyfile`, or the `c.KernelGatewayApp.client_ca` in `~/.jupyter/jupyter_kernel_gateway_config.py` to attain the same results.  In fact there are additional benefits to using AT-TLS on z/OS with Jupyter Kernel Gateway.  If you have everything configured correctly, you can run Jupyter kernels as the remote notebook user.  This concept may be a bit confusing, so let me try to explain.

On a z/OS system, just like any others, we have users.  One such user will run the Jupyter Kernel Gateway server.  In a traditional Jupyter Kernel Gateway model, the notebook kernels would run under the same user ID as the one that started Jupyter Kernel Gateway.  When you have AT-TLS set up, the notebook kernels would run under the user IDs of the notebook users, instead of the user ID that started Jupyter Kernel Gateway.  This provides additional benefits of granular access and resource usage control for different notebook users.

##### Set up AT-TLS on z/OS
If the z/OS system doesn't already have AT-TLS set up, follow the instructions in the "Application Transparent Transport Layer Security data protection" section in [z/OS Communication Server: IP Configuration Guide](https://www.ibm.com/support/knowledgecenter/en/SSLTBW_2.1.0/com.ibm.zos.v2r1.halz002/attls.htm) to set up AT-TLS for the system.

The steps include
* Update TCP/IP profile
* Set up Policy Agent
* Define AT-TLS policy

**Note:** When moving to an AT-TLS enabled setup be sure to change KG_URL in .env to https://

##### Define certificates for Jupyter Kernel Gateway
On z/OS, digital certificates can be managed through RACF, PKI Services, or other security products. RACF is often used in a simple environment, whereas PKI Services is recommended when a large (50 or more) number of certificates are needed.

For more information about digital certificates in RACF, see "Planning your certificate environment" and "Setting up your certificate environment" in [z/OS Security Server RACF Security Administrator's Guide](https://www.ibm.com/support/knowledgecenter/en/SSLTBW_2.1.0/com.ibm.zos.v2r1.icha700/icha700_Planning_your_certificate_environment.htm).

For more information about PKI Services, see "Introducing PKI Services" in [z/OS Cryptographic Services PKI Services Guide and Reference](https://www.ibm.com/support/knowledgecenter/en/SSLTBW_2.1.0/com.ibm.zos.v2r1.ikya100/int.htm).

The following steps show examples using RACF commands:
1. Define a CA certificate, if you don't already have one. For example:
```
RACDCERT GENCERT CERTAUTH SUBJECTSDN(OU(’Local CA’) O(’IBM’) C(’US’))
     WITHLABEL(’My Local CA’) NOTAFTER(DATE(2030/01/01)) SIZE(1024)
```
2. Define a key ring for the Jupyter Kernel Gateway server. For example:
```
RACDCERT ADDRING(JKGRing) ID(JKGID)
```
3. Define a certificate for the Jupyter Kernel Gateway server that is signed by the CA. For example:
```
RACDCERT GENCERT ID(JKGID) SIGNWITH(CERTAUTH LABEL(’My Local CA’))
     KEYUSAGE(HANDSHAKE) WITHLABEL(’JKG Server Cert’)
     SUBJECTSDN(CN(’JKG Server’) O(’IBM’) L(’Poughkeepsie’)
     SP(’New York’) C(’US’)) NOTAFTER(DATE(2030/01/01))
```
4. Connect both the CA certificate and the server certificate to the Jupyter Kernel Gateway's key ring. For example:
```
RACDCERT ID(JKGID) CONNECT(CERTAUTH LABEL(’My Local CA’)
     RING(JKGRing))
RACDCERT ID(JKGID) CONNECT(ID(JKGID) LABEL(’JKG Server Cert’)
     RING(JKGRing) USAGE(PERSONAL) DEFAULT)
```
5. Give the Jupyter Kernel Gateway server access to its key ring. For example:
```
PERMIT IRR.DIGTCERT.LISTRING CLASS(FACILITY) ID(JKGID) ACCESS(READ)
```

##### Define AT-TLS policy for Jupyter Kernel Gateway
Your AT-TLS policy needs to have a "rule" covering the port Jupyter Kernel Gateway listens to.

Below is a sample AT-TLS policy that assumes Jupyter Kernel Gateway listens on port 8099 and uses certificates on RACF keyring JKGRing:

```
####################################################################################
# Sample AT-TLS policy for Jupyter Kernel Gateway on z/OS
####################################################################################

# This rule covers all inbound connection to the Jupyter Kernel Gateway port (8099).
# It also references the gAction1 and serverEnv sections.
TTLSRule                          JKG_Server
{
  Direction                       Inbound
  LocalPortRange                  8099
  TTLSGroupActionRef              gAction1
  TTLSEnvironmentActionRef        serverEnv
}

# This Group Action simply indicates TLS is to be enabled for the connection.
TTLSGroupAction                   gAction1
{
  TTLSEnabled                     On
}

# This Environment Action indicates Jupyter Kernel Gateway acts as the server during
# TLS handshake and uses certificates on keyring JKGRing, and that client
# authentication is required. It also references gAdvAction1 for further configuration.
TTLSEnvironmentAction             serverEnv
{
  HandshakeRole                   ServerWithClientAuth
  EnvironmentUserInstance         0
  TTLSKeyRingParms
  {
    Keyring                       JKGRing
  }
  Trace                           7
  TTLSEnvironmentAdvancedParmsRef gAdvAction1
}

# This Environment Advanced Parameters section indicates the client certificate
# must map to a valid z/OS user ID. It also enforces that TLS v1.2 to be used.
TTLSEnvironmentAdvancedParms      gAdvAction1
{
 ClientAuthType		                SAFCheck
 TLSv1 Off
 TLSv1.1 Off
 TLSv1.2 On
}
```

##### Give Jupyter Kernel Gateway the proper authority
As mentioned earlier, with client authentication (via AT-TLS) enabled on z/OS, the notebook kernels would run under the user IDs of the notebook users, instead of that of the Jupyter Kernel Gateway. In order for this happen, the user ID that Jupyter Kernel Gateway runs under (JKGID in the previous examples) requires additional authority:

* READ authority to the BPX.SRV.*userid* SURROGAT class profile, where *userid* is the user ID of the notebook user. This allows the Jupyter Kernel Gateway to spawn the notebook kernels under *userid*. You can also define a generic BPX.SRV.** profile that covers all user IDs. For example:
```
SETROPTS CLASSACT(SURROGAT) RACLIST(SURROGAT) GENERIC(SURROGAT)
RDEFINE SURROGAT BPX.SRV.** UACC(NONE)
PERMIT BPX.SRV.** CLASS(SURROGAT) ACCESS(READ) ID(JKGID)
SETROPTS GENERIC(SURROGAT) RACLIST(SURROGAT) REFRESH
```
Issue the following command to verify that the profile setup is successful:
```
RLIST SURROGAT BPX.SRV.** AUTHUSER
```

* Configure the z/OS system that hosts the Jupyter Kernel Gateway to honor ACLs that will be set by the Gateway. For example:
```
SETROPTS CLASSACT(FSSEC)
```
* FIXME directory permissions



#### Export User Certs for Use by Jupyter Notebook

##### Export the end user certificate

Once you have created your end user certificate in RACF, signed by the certificate authority, export the client certificate (public and private key) and CA certificate (public key) into a PKCS#12 certificate package (.p12).   Export the client certificate to transfer to the remote system that will host the end user.
	
From TSO, issue the following command: ('password' is the password used to access the contents of the package.)

```
		RACDCERT EXPORT(LABEL('Spark Client Cert'))
		ID(JDOE)
		DSN('JDOE.USERA.P12')
		FORMAT(PKCS12DER)
		PASSWORD('password')
```
		
This creates a p12 package in the JDOE.USERA.P12 dataset.
	
For more information on exporting certificates, see [https://www.ibm.com/support/knowledgecenter/SSLTBW_2.1.0/com.ibm.zos.v2r1.icha400/le-export.htm?view=embed](https://www.ibm.com/support/knowledgecenter/SSLTBW_2.1.0/com.ibm.zos.v2r1.icha400/le-export.htm?view=embed).
	
##### Transfer the certificate package from z/OS to the remote Linux system that will host the end user.
Transfer (e.g. FTP) the p12 package as binary to the remote Linux system.

##### From the remote linux system, validate the contents of the package.

Issue the openssl command to display the contents of the package.  You will be prompted twice for the password used during the export.

```
openssl pkcs12 -info -in JDOE.USERA.P12 -passin pass:"password"
```
	
This will display the end user (USERA) certificate, the private key, and the public Certificate Authority certificate.
	
If you do not want to display the private key, you can use the "-nokeys" option.  For example:

```
openssl pkcs12 -info -in JDOE.USERA.P12 -passin pass:"password" -nokeys
```


##### Extract the client certificate, the private key and the CA certificate from the p12 package to three separate files, all in PEM format.

These files should be used in the `Setup SSL/TLS Client Authentication on the nb2kg/hub-notebook Images` section.

###### To extract the client certificate into a pem-encoded file, issue:

```
openssl pkcs12 -in JDOE.USERA.P12 -passin pass:"password" -out usera.orig.pem -clcerts -nokeys
```
		
This creates a file "usera.orig.pem".   This file cannot be used until some informational lines are removed.  
		
Specifically, lines outside the BEGIN and END lines should be removed.  Here is an example of what these look may like:

```
		Bag Attributes
		    friendlyName: Spark Client Cert
		    localKeyID: 00 00 00 01
		subject=/C=US/ST=New York/L=Poughkeepsie/O=IBM/CN=Spark Client
		issuer=/C=US/O=IBM/OU=SPARK Local CA
```		

To remove these lines, you may edit the file directly, or issue the following command to convert it:

```
openssl x509 -in usera.orig.pem -out usera.pem
```
		
usera.pem should now contain only BEGIN and END lines with the base64 encoded certificate inside.  
For example:


```
-----BEGIN CERTIFICATE-----
(base 64 encoded certificate contents)
-----END CERTIFICATE-----
```		
		
###### To extract the CA certificate into a pem-encoded file, issue:
		
```		
openssl pkcs12 -in JDOE.USERA.P12 -passin pass:"password" -out ca.orig.pem -cacerts -nokeys
```
		
To remove the informational lines, either edit the file, or issue the following commands:

```
openssl x509 -in ca.orig.pem -out ca.pem
```
		
***NOTE: if you have a chain of certificate authorities, you may need to edit the file directly to remove each instance instead of issuing the openssl x509 command.***
		
		
###### To extract the private key into a pem-encoded file, issue  (you will be prompted for the password twice) :

```
openssl pkcs12 -in JDOE.USERA.P12 -passin pass:"password" -out userakey.pem -nocerts -nodes
```
		
Edit userakey.pem to remove the informational lines.
Here is an example of what these look may like:

```
		Bag Attributes
		    friendlyName: Spark Client Cert
		    localKeyID: 00 00 00 01
		subject=/C=US/ST=New York/L=Poughkeepsie/O=IBM/CN=Spark Client
		issuer=/C=US/O=IBM/OU=SPARK Local CA
```
	
		
###### To verify the end user certificate (once CA certificate is extracted), issue:

```
openssl verify -CAfile ca.pem usera.pem
```
		
You should see output similar to:

```
usera.pem: OK
		
```


## Behind the scenes

`make hub_image` does a few things behind the scenes, to set up the environment for JupyterHub:

### Create a Docker Network

Create a Docker network for inter-container communication.  The benefits of using a Docker network are:

* container isolation - only the containers on the network can access one another
* name resolution - Docker daemon runs an embedded DNS server to provide automatic service discovery for containers connected to user-defined networks.  This allows us to access containers on the same network by name.

Here we create a Docker network named `jupyterhub-network`.  Later, we will configure the JupyterHub and single-user Jupyter Notebook containers to run attached to this network.

```
docker network create jupyterhub-network
```

### Create a JupyterHub Data Volume

Create a Docker volume to persist JupyterHub data.   This volume will reside on the host machine.  Using a volume allows user lists, cookies, etc., to persist across JupyterHub container restarts.

```
docker volume create --name jupyterhub-data
```

## FAQ

### How can I view the logs for JupyterHub or users' Notebook servers?

Use `docker logs <container>`.  For example, to view the logs of the `jupyterhub` container

```
docker logs jupyterhub
```

### How do I specify the Notebook server image to spawn for users?

In this deployment, JupyterHub uses DockerSpawner to spawn single-user
Notebook servers. You set the desired Notebook server image in a
`DOCKER_NOTEBOOK_IMAGE` environment variable.

JupyterHub reads the Notebook image name from `jupyterhub_config.py`, which
reads the Notebook image name from the `DOCKER_NOTEBOOK_IMAGE` environment
variable:

```
# DockerSpawner setting in jupyterhub_config.py
c.DockerSpawner.container_image = os.environ['DOCKER_NOTEBOOK_IMAGE']
```

By default, the`DOCKER_NOTEBOOK_IMAGE` environment variable is set in the
`.env` file.

```
# Setting in the .env file
DOCKER_NOTEBOOK_IMAGE=jupyter/scipy-notebook:2d878db5cbff
```

To use a different notebook server image, you can either change the desired
container image value in the `.env` file, or you can override it
by setting the `DOCKER_NOTEBOOK_IMAGE` variable to a different Notebook
image in the environment where you launch JupyterHub. For example, the
following setting would be used to spawn single-user `pyspark` notebook servers:

```
export DOCKER_NOTEBOOK_IMAGE=jupyterhub/pyspark-notebook:2d878db5cbff

docker-compose up -d
```

### If I change the name of the Notebook server image to spawn, do I need to restart JupyterHub?

Yes. JupyterHub reads its configuration which includes the container image
name for DockerSpawner. JupyterHub uses this configuration to determine the
Notebook server image to spawn during startup.

If you change DockerSpawner's name of the Docker image to spawn, you will
need to restart the JupyterHub container for changes to occur.

In this reference deployment, cookies are persisted to a Docker volume on the
Hub's host. Restarting JupyterHub might cause a temporary blip in user
service as the JupyterHub container restarts. Users will not have to login
again to their individual notebook servers. However, users may need to
refresh their browser to re-establish connections to the running Notebook
kernels.

### How can I backup a user's notebook directory?

There are multiple ways to [backup and restore](https://docs.docker.com/engine/userguide/containers/dockervolumes/#backup-restore-or-migrate-data-volumes) data in Docker containers.  

Suppose you have the following running containers:

```
docker ps --format "table {{.ID}}\t{{.Image}}\t{{.Names}}"

CONTAINER ID        IMAGE                    NAMES
bc02dd6bb91b        jupyter/minimal-notebook jupyter-jtyberg
7b48a0b33389        jupyterhub               jupyterhub
```

In this deployment, the user's notebook directories (`/home/jovyan/work`) are backed by Docker volumes.

```
docker inspect -f '{{ .Mounts }}' jupyter-jtyberg

[{jtyberg /var/lib/docker/volumes/jtyberg/_data /home/jovyan/work local rw true rprivate}]
```

We can backup the user's notebook directory by running a separate container that mounts the user's volume and creates a tarball of the directory.  

```
docker run --rm \
  -u root \
  -v /tmp:/backups \
  -v jtyberg:/notebooks \
  jupyter/minimal-notebook \
  tar cvf /backups/jtyberg-backup.tar /notebooks
```

The above command creates a tarball in the `/tmp` directory on the host.
