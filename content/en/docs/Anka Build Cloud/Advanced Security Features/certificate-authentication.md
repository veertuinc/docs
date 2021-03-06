---
date: 2019-12-12T00:00:00-00:00
title: "Certificate Authentication"
linkTitle: "Certificate Authentication"
weight: 2
description: >
  How to set up certificate based authentication.
---

> **This guide requires an Anka Enterprise (or higher) license**


There are several different ways you can enable Certificate authentication:

1. With the combined (controller + registry) native macOS package: You'll edit the `/usr/local/bin/anka-controllerd`, enabling TLS/HTTPS (required) and then certificate authentication (with either ENVs or options/flags).
2. With the docker package: You'll edit the `docker-compose.yml`, enabling TLS/HTTPS and then certificate authentication (with ENVs).
3. With either the controller or registry standalone: You'll edit the proper config files, enabling TLS/HTTPS (required) and then certification authentication (with either ENVs or options/flags).

> If you're using the Enterprise Plus license, [you will need to setup authorization for your certificate]({{< relref "docs/Anka Build Cloud/Advanced Security Features/certificate-authentication.md#managing-usergroup-permissions-authorization" >}})

## Requirements

1. A Root CA certificate. For more information about CAs, see https://en.wikipedia.org/wiki/Certificate_authority. Usually provided by your organization or where you obtain your certificate signing. We will refer to this as **anka-ca-crt.pem** and **anka-ca-key.pem** throughout the guide.
2. A certificate (signed with the Root CA) for the Anka Build Cloud Controller & Registry.
3. Certificates (signed with the Root CA) for your Anka Build Nodes so they can connect/authenticate with the Anka Build Cloud Controller & Registry.

> **The guide will show you how to generate self-signed versions of these. If you already have certificates, you can skip the commands.**

---

## 1. Create a self-signed Root CA certificate

If you don’t have a, you can create it with openssl and add it to your keychain:

```shell
cd ~
openssl req -new -nodes -x509 -days 365 -keyout anka-ca-key.pem -out anka-ca-crt.pem -subj "/O=$ORGANIZATION/OU=$ORG_UNIT/CN=$CA_CN"
# Add the Root CA to the System keychain so the Root CA is trusted and you can avoid warnings when you go to access the Controller UI; if you have a better method to do this without using the system keychain, feel free to use it
sudo security add-trusted-cert -d -k /Library/Keychains/System.keychain anka-ca-crt.pem
```

## 2. Configuring TLS for Controller & Registry

> Required

> The **Controller TLS certificate** ("`server`" cert options) is not part of the authentication process and doesn't need to be derived from the CA you just generated. This means that you can use certificates supplied by your organization or a 3rd party for TLS/HTTPS

> Ensure your signed key is decrypted using `openssl rsa`

> For this guide, we're running the Controller & Registry locally, so we use 127.0.0.1. Update this depending on where you have things hosted

If you do not have TLS certificates for your Controller & Registry, you can create them now:

```shell
export CONTROLLER_ADDRESS="127.0.0.1"
export REGISTRY_ADDRESS=$CONTROLLER_ADDRESS
openssl genrsa -out anka-controller-key.pem 4096
openssl req -new -nodes -sha256 -key anka-controller-key.pem -out anka-controller-csr.pem -subj "/O=$ORGANIZATION/OU=$ORG_UNIT/CN=$CONTROLLER_CN" -reqexts SAN -extensions SAN -config <(cat /etc/ssl/openssl.cnf <(printf "[SAN]\nextendedKeyUsage = serverAuth\nsubjectAltName=IP:$CONTROLLER_ADDRESS"))
openssl x509 -req -days 365 -sha256 -in anka-controller-csr.pem -CA anka-ca-crt.pem -CAkey anka-ca-key.pem -CAcreateserial -out anka-controller-crt.pem -extfile <(echo subjectAltName = IP:$CONTROLLER_ADDRESS)
```

> You can use the same certificate for both the Controller and Registry

Ensure that the certificate has **Signature Algorithm: sha256WithRSAEncryption** using `openssl x509 -text -noout -in ~/anka-controller-crt.pem | grep Signature` (https://support.apple.com/en-us/HT210176)

> Beginning in Controller version 1.12.0, [you can control the allowed TLS Cipher Suites and minimum/maximum TLS versions]({{< relref "docs/Anka Build Cloud/configuration-reference.md#tls" >}})

### Native macOS Controller & Registry package

Edit `/usr/local/bin/anka-controllerd` in the following manner:

1. Change `LISTEN_ADDRESS=":80"` to `LISTEN_ADDRESS=":443"`

> SSL will work on any port you want

2. Append the following parameters on the end of the **$CONTROLLER_BIN** line:

    ```shell
    --use-https \
    --server-cert $CERT_FOLDER/anka-controller-crt.pem \
    --server-key $CERT_FOLDER/anka-controller-key.pem
    ```

3. Ensure https is in the registry URL:

    ```shell
    --anka-registry "https://$REGISTRY_ADDRESS:8089" \
    ```

> The Controller & Registry runs as root. This is why you need to specify the absolute path to the location where you generated the certs.

### Linux/Docker Controller & Registry

Within the `docker-compose.yml`:

1. Change the **anka-controller** ports from `80:80` to `443:80`. You can keep the **anka-registry** ports the same.
2. Under the **anka-controller**, modify or set **ANKA_REGISTRY_ADDR** to use `https://`.
3. Uncomment the highlighted lines shown below and modify `****EDIT_ME****` to the location you created your certificates in for both **anka-controller** and **anka-registry**:

{{< highlight dockerfile "hl_lines=11 13" >}}

. . .

  anka-controller:
    build:
       context: .
       dockerfile: anka-controller.docker
    ports:
       - "80:80"
    # To change the port, change the above line: - "CUSTOM_PORT:80"
    ######   EDIT HERE FOR TLS  ########
    # volumes:
      # Path to ssl certificates directory 
      # - ****EDIT_ME****:/mnt/cert
      
. . .

{{</ highlight >}}

Now let’s configure the Controller & Registry containers/services to use those certificates. Search for the environment variables **USE_HTTPS**, **SERVER_CERT** and **SERVER_KEY** in `docker-compose.yml`, uncomment the lines, and then modify the `****EDIT_ME****` with the name of your certificate for both **anka-controller** and **anka-registry**:

{{< highlight dockerfile "hl_lines=28 31 33" >}}

. . .

 anka-controller:
    build:
       context: .
       dockerfile: anka-controller.docker
    ports:
       - "443:80"
    # To change the port, change the above line: - "CUSTOM_PORT:80"
    ######   EDIT HERE FOR TLS  ########
    volumes:
      # Path to ssl certificates directory 
      - /home/ubuntu:/mnt/cert
    depends_on:
       - etcd
      #  - beanstalk
       - anka-registry
    restart: always
    environment:
      # Address of anka registry. this address will be passed to your build Nodes
      ANKA_REGISTRY_ADDR: https://<REGISTRY_ADDRESS>:8089

. . .

      ######   EDIT HERE FOR TLS ########

      # Use https, if enabled Controller will use tls for http communication
      # USE_HTTPS:              --use-https   

      # Server certificate pem
      # SERVER_CERT:            --server-cert /mnt/cert/****EDIT_ME**** 
      # Server private key pem
      # SERVER_KEY:             --server-key /mnt/cert/****EDIT_ME****
. . .

{{</ highlight >}}

> **Make sure to perform the same changes for the anka-registry container.**

### Test the Configuration

Start or restart your Controller and test the new TLS configuration using `https://`. You can also try using `curl -v https://$HOST/api/v1/status`.

If that doesn’t work, try to repeat the above steps and validate that the file names and paths are correct. If you are still having trouble, debug the system as explained in the Debugging Controller section.

## 3. Creating self-signed Node Certificates

The Controller's authentication module uses the Root CA (anka-ca-crt.pem) to authenticate the Node certificates. When the Node sends the requests to the Controller, it will present its certificates. Those certificates will then be validated against the configured CA.

You can use the following openssl commands to create Node certificates using the Root CA:

```shell
openssl genrsa -out node-$NODE_NAME-key.pem 4096
openssl req -new -sha256 -key node-$NODE_NAME-key.pem -out node-$NODE_NAME-csr.pem -subj "/O=$ORGANIZATION/OU=$ORG_UNIT/CN=$NODE_NAME"
openssl x509 -req -days 365 -sha256 -in node-$NODE_NAME-csr.pem -CA anka-ca-crt.pem -CAkey anka-ca-key.pem -CAcreateserial -out node-$NODE_NAME-crt.pem
```

## 4. Configuring the Controller & Registry with the CA Root certificate and enable authentication

> The CA_CERT is the authority that is used to validate the node certificates you'll pass in later.

### Native macOS Controller & Registry package

Edit the `/usr/local/bin/anka-controllerd` and add the following onto the end of the **$CONTROLLER_BIN** line:

```shell
--enable-auth \
--ca-cert $CERT_FOLDER/anka-ca-crt.pem \
--skip-tls-verification
```

### Linux/Docker Controller & Registry package

Within the `docker-compose.yml`, search for the environment variables **ENABLE_AUTH** and **CA_CERT**. Edit both **anka-controller** and **anka-registry** so they look like the configuration below (assuming that a certificate folder is already mounted at /mnt/cert).

{{< highlight dockerfile "hl_lines=26 27" >}}

. . .

anka-controller:
   build:
      context: .
      dockerfile: anka-controller.docker
   ports:
      - "443:80"
   # To change the port, change the above line: - "CUSTOM_PORT:80"
   volumes:
     # Path to ssl certificates directory
     - /home/ubuntu:/mnt/cert
   depends_on:
      - etcd
     #  - beanstalk
   restart: always
   environment:  

. . .

     USE_HTTPS:              --use-https  
     # Server certificate pem
     SERVER_CERT:            --server-cert /mnt/cert/anka-controller-crt.pem
     # Server private key pem
     SERVER_KEY:             --server-key /mnt/cert/anka-controller-key.pem
     ENABLE_AUTH:            --enable-auth 
     CA_CERT:                --ca-cert /mnt/cert/anka-ca-crt.pem


. . .

{{</ highlight >}}

> **Make sure to perform the same changes for the anka-registry container.**

> **Until you have a Node joined to the Controller, it won't see your Enterprise license and won't enable authentication.**

> If you're connecting the Anka CLI with the HTTPS Registy, you can use the Node certificates: `anka registry --cert /Users/$USER_WHERE_CERTS_ARE/node-$NODE_NAME-crt.pem --key /Users/$USER_WHERE_CERTS_ARE/node-$NODE_NAME-key.pem --cacert /Users/$USER_WHERE_CERTS_ARE/anka-ca-crt.pem add $REGISTRY_NAME https://$REGISTRY_ADDRESS:8089`

## 5. Joining your Node to the Controller with the Node certificate

> Certificates are cached, so if you update/renew them, you need to either: 1. disjoin and re-join them to the controller, issue `sudo pkill -9 anka_agent` on each node to restart the agent, or issue a `<controller>/v1/node/update` PUT to the controller API to forcefully update all nodes.

> If you're using a signed certificate for the controller dashboard, but self-signed certificates for your nodes and CI tools, you'll need to specify the `--cacert` for `ankacluster join` and `anka registry add` commands and point it to the signed CA certificate. You'll usually see `SSLError: ("bad handshake: Error([('SSL routines', 'tls_process_server_certificate', 'certificate verify failed')],)",)` if the wrong CA is being used.

> If you previously joined your Nodes to the Controller, you'll want to `sudo ankacluster disjoin` on each before proceeding (if it hangs, use `ps aux | grep anka_agent | awk '{print $2}' | xargs kill -9` and try disjoin again).

Copy both the Node certificates (node-$NODE_NAME-crt.pem, node-$NODE_NAME-key.pem) and the anka-ca-crt.pem to the Node.

Then, use the `ankacluster` command to connect it to the Controller in the following manner:

```shell
sudo ankacluster join https://$CONTROLLER_ADDRESS --skip-tls-verification --cert /Users/$USER_WHERE_CERTS_ARE/node-$NODE_NAME-crt.pem --cert-key /Users/$USER_WHERE_CERTS_ARE/node-$NODE_NAME-key.pem --cacert /Users/$USER_WHERE_CERTS_ARE/anka-ca-crt.pem
You should see output similar to the following:
Testing connection to Controller...: OK
Testing connection to registry….: OK
Ok
Cluster join success
```

### Testing Controller API Authentication

Restart your Controller & Registry and then test the status endpoint with curl:

```shell
curl -v https://$HOST/api/v1/status 
```

The response you should get is a **401 Authentication Required** similar to below:

```shell
> GET /api/v1/status HTTP/2
> Host: localhost:80
> User-Agent: curl/7.58.0
> Accept: */*
> 
* Connection state changed (MAX_CONCURRENT_STREAMS updated)!
< HTTP/2 401 
< content-type: application/json
< content-length: 54
< date: Thu, 28 Nov 2019 16:58:23 GMT
< 
{"status":"FAIL","message":"Authentication Required"}
```

**If this is the response you get, it means the authentication module is working.**

Let’s try to get a response using the Node certificate we created. Execute the same command, but now pass Node certificate and key:

```shell
curl -v https://$CONTROLLER_ADDRESS/api/v1/status --cert /Users/$USER_WHERE_CERTS_ARE/node-$NODE_NAME-crt.pem --key /Users/$USER_WHERE_CERTS_ARE/node-$NODE_NAME-key.pem
```

If everthing is configured correctly, you should see something like this (I used 127.0.0.1 to setup this example):

```shell
*   Trying 127.0.0.1...

. . .

> GET /api/v1/status HTTP/2
> Host: 127.0.0.1:80
> User-Agent: curl/7.64.1
> Accept: */*
> 
* Connection state changed (MAX_CONCURRENT_STREAMS == 250)!
< HTTP/2 200 
< cache-control: no-store
< content-type: application/json
< content-length: 184
< date: Sun, 12 Apr 2020 04:26:13 GMT
< 
{"status":"OK","message":"","body":{"status":"Running","version":"1.7.0-4e6617d3","registry_address":"https://127.0.0.1:8089","registry_status":"Running","license":"enterprise plus"}}
* Connection #0 to host 127.0.0.1 left intact
* Closing connection 0
```

## 6. Final Notes

- You may notice that the Controller UI doesn't load or acts strangely. You will need to enable [Root Token Authentication]({{< relref "docs/Anka Build Cloud/Advanced Security Features/root-token-authentication.md" >}}) to access the controller UI.
- If you get an invalid cert error from the Controller UI, make sure that you add the root CA you generated to your system keychain.

---

## Managing User/Group Permissions (Authorization)

When creating certificates, you'll want to specify CSR values using openssl's `-subj` option. For example, if we're going to generate a certificate so our Jenkins instance can access the Controller & Registry, you'll want to use something like this:

```shell
-subj "/O=MyOrgName/OU=$ORG_UNIT/CN=Jenkins"
```

> Required values are `O=` and `CN=`

> Spaces are supported in `O=` and Anka Build Cloud Controller version >= 1.10

Within the Controller, we use **`O=`** as the **permission group name** and **`CN=`** as the **username**. The **Group Name** will be `MyOrgName`, like we used in the `-subj` above.

{{< include file="shared/content/en/docs/Anka Build Cloud/Advanced Security Features/partials/_managing-permissions.md" >}}
