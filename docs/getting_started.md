# Getting started

The ads.cert suite of security protocols provide cryptographic security mechanisms suitable for use with the advertising ecosystem.  This open source software suite provides off-the-shelf tools for working with these standards.

This guide will walk you thorough the steps required to implement ads.cert in your company using our open source software suite.

# What's included

The ads.cert open source software suite consists of a single application that provides various ads.cert protocol utilities.

* Cryptographic keyring creation
* Key rotation management
* Commercial key management server integration
* DNS record update automation
* Signing operations
* Signature verification operations
* Integration API client library for ad servers
* Implementation validation and diagnostic tools

We designed the application to be installable as a local command line utility and deployed within containerized production environments.

Most users can simply download and install one of the prebuilt application images, although you have the option to build the application from scratch if desired.

# What you'll set up

To summarize, setting up ads.cert consists of:

* Selecting and registering an Internet domain name which will serve as your ads.cert Call Sign
* Creating an `adscert_config.yaml` file containing your ads.cert Call Sign and other configuration preferences
* Running the ads.cert key rotation tool to generate your ads.cert private key, encrypted and written to a `generated_adscert_keyring.json` file that you'll check into your source control system.
* Running the ads.cert DNS publication tool to create a `generated_adscert_dns_publication.json` file that can be subsequently fed into automation for updating DNS APIs or zone files.  We provide common adaptors that make use of this format, such as Hashicorp Terraform scripts.  (You can alternatively apply these DNS updates manually, but we recommend automation of key rotation.)
* If receiving signed HTTP requests, setting up a DNS record that indicates the ads.cert Call Sign domain associated with the server receiving the HTTP requests.

With these keys and DNS configurations established, you can then integrate ads.cert signing and verification software into your infrastructure.  This requires:

* Setting up the ads.cert signatory RPC server within your ad serving environment to generate or verify signatures, capable of integration into various ad server software languages.  (Implementers of smaller scale deployments using software natively written in Go may instead choose to integrate the signatory software in-process, although this is the less-preferred option.)
* Modifying ad server application code to request signatures or verifications for HTTP requests.
* Integrating API responses into your logging and monitoring systems.

We provide tooling for verifying implementations and integrating testing into your CI/CD environment.

# Tutorial: your first ads.cert deployment

Let's walk through the steps to create a real, working ads.cert Call Sign domain, keyring, DNS configuration, and practice with signing/verification to show that it is working.

This tutorial will demonstrate some steps using certain Google Cloud products, but you can apply these concepts to other environments.

## Step 1: download and install the ads.cert open source software

To use ads.cert tools, you'll need to either fetch the prebuilt Docker image containing the all-in-one application, or you will need to build the application from source.  Both options are easy if you have the required prerequisites (Docker and/or Go SDK) installed.

The Google Cloud Shell Editor provides a free IDE, console, and virtual machine image that you can use for this tutorial.  It has Docker and Go preinstalled.

### Option A: run using a Docker container

With Docker installed, the simplest and most convenient way to run the ads.cert application is by using the `docker run` command:

```
$ docker run ghcr.io/iabtechlab/adscert:latest
```

This downloads the application image, if required, and runs the application in a self-contained Linux VM.

The `docker run` command lets users further specify the application to run and arguments to pass to the application:

```
$ docker run
"docker run" requires at least 1 argument.
See 'docker run --help'.

Usage:  docker run [OPTIONS] IMAGE [COMMAND] [ARG...]

Run a command in a new container
```

Using this command line format, you can run the `adscert` executable within the container and pass it command line arguments.  For example, the following executes the command with the `version` argument:

```
$ docker run ghcr.io/iabtechlab/adscert:latest adscert version
IAB Tech Lab ads.cert open source toolkit
vX.X.X (official release) Built 2022-XX-XXTXX:XX:XXZ
```

If using Linux or Mac OS, you may wish to provide a shortcut to this tool for interactive use by creating an alias:

```
alias adscert='docker run ghcr.io/iabtechlab/adscert:latest adscert'
```

This will let you run most adscert commands as if the application were installed locally.

```
$ adscert version
IAB Tech Lab ads.cert open source toolkit
vX.X.X (official release) Built 2022-XX-XXTXX:XX:XXZ
```

### Option B: install using Go SDK

Verify that you have a recent version of the Go SDK installed locally.

```
$ go version
```

If present, the installed version will be output:

```
go version go1.17.2 linux/amd64
```

Run the following command to download, build, and install the ads.cert application locally:

```
$ go install github.com/IABTechLab/adscert@latest
```

Once complete, you should be able to run the `adscert` command from the command line:

```
$ adscert
IAB Tech Lab ads.cert open source toolkit
vX.X.X (local build) Built 2022-XX-XXTXX:XX:XXZ
```

## Step 2: establish an ads.cert Call Sign Internet domain

For this tutorial, you'll need to register an Internet domain name or use one currently owned by your business.  To test with no cost, consider registering a free domain using [Freenom](https://www.freenom.com/) under a free top level domain option such as .tk/.ml/.ga/.cf/.gq.  This tutorial will use `signer.tk` as the domain name, but you can choose any name choice you'd like for this throwaway exercise.

Freenom lets you choose between using their servers for DNS versus selecting your own DNS servers.  The ads.cert solutions will work fine with their DNS servers when setting up ads.cert DNS records manually, but ideally you'll use automation to manage DNS records.  For now, we'll use Freenom's DNS servers as the simpler option and then later modify this to use a separate, automatable DNS provider.

## Step 3: create your ads.cert configuration file

To simulate deployment in a real environment, we'll create a separate source repository containing the version controlled deployment configuration files.  For this demo we wil use Github, although you should use your business' preferred source control repository for managing your real deployments.

For example, after manually creating the Github repository github.com/cmlight/adscert-deploy-example, I can clone it locally:

```
$ git clone https://github.com/cmlight/adscert-deploy-example.git
$ cd adscert-deploy-example/
```

You'll create an ads.cert configuration file.  To generate a starter file from a template, run:

```
$ adscert init
```

This will generate an `adscert_config.yaml` starter file for you to edit using your favorite code editor.  Note that this action also generates an empty `generated_adscert_keyring.json` and `generated_adscert_dns_publication.json` file.  These will be discussed later.

For the first exercise, add your ads.cert Call Sign domain to the `callsigns` section of the `adscert_config.yaml` file:

```
callsigns:
  - domain: signer.tk
    realm: delivery
```

We also need to configure how the tools will generate and secure private keys used by the ads.cert protocols.  Under the `key_security > key_management` section, uncomment the `insecure_plaintext_keyring` settings:

```
key_security:
  key_management_system:
    insecure_plaintext_keyring:
      enabled: true
      use_for_new_keys: true
```

This instructs the tooling to store generated private keys in plaintext within the `generated_adscert_keyring.json` file.  Because we recommend checking this file into your SCM, we do **NOT** recommend using this setting for production.

Save the `adscert_config.yaml` file.

To validate your configuration, run the `adscert vet` command.  This will check for syntax errors and data consistency issues between files.

```
$ adscert vet
Checking ads.cert configuration files...
- Loaded adscert_config.yaml
- Loaded generated_adscert_keyring.json
- Loaded generated_adscert_dns_publication.json
- Loaded fully_deployed_adscert_keyring.json
- Loaded fully_deployed_adscert_dns_publication.json

Configured ads.cert call sign domains:
 - Domain: signer.tk  Realm: delivery

ads.cert keyring entries:
 - Domain: signer.tk  Realm: delivery
   0 keys configured
   Due for new key generation

ads.cert fully deployed keyring entries:
 - Domain: signer.tk  Realm: delivery
   0 keys configured

ads.cert DNS record publications:
 - Domain: signer.tk  Realm: delivery
   _delivery._adscert.signer.tk. TXT DNS record not found (matches intent)

Keyring OK
1 domains/realms due for new key generation on next keyring autorotation run.

Fully-deployed Keyring OK
0 modifications due

DNS Publication List OK
0 DNS entries pending publication.

All config state validations PASSED
```

You can now submit/push this change to your SCM repository.

## Step 4: run the ads.cert key rotation tool

Notice on the output from the last step that the `adscert vet` tool flagged the ads.cert keyring as "due for new key generation."  The ads.cert software generates keys using a separate step so that the process generating these secrets can run within a privileged, access controlled environment.  In an ideal ads.cert deployment, no human administrators will have direct access to the raw key materials unless they take some action that would result in an auditable event.  To permit this isolation, the key rotation step runs as a privileged role account which has access to key encryption operations.

For this initial walkthrough, we configured the application to use non-encrypted keys in the prior step, so no privileged key management system access is needed currently.  Run the `adscert keyring autorotate` command which will scan the keyring for required actions and generate the necessary keys.

```
$ adscert keyring autorotate
Checking ads.cert configuration files...
- Loaded adscert_config.yaml
- Loaded generated_adscert_keyring.json
- Loaded fully_deployed_adscert_keyring.json

Configured ads.cert call sign domains:
 - Domain: signer.tk  Realm: delivery

ads.cert keyring entries:
 - Domain: signer.tk  Realm: delivery
   0 keys configured
   Due for new key generation

Keyring OK
1 domains/realms due for new key generation on next keyring autorotation run.

Generating keys:
  Domain: signer.tk  Realm: delivery
    Public key: Bm8J1RW3RxHp_-mx3lE7eAuYObfALvwurVjXtcaYFVA
    Key encryption strategy: insecure-plaintext-kms://
    Status: NEW
    Last updated: 2022-02-11T12:34:56-0700

Wrote 1 key(s) to file generated_adscert_keyring.json

New file summary:
  Domain: signer.tk  Realm: delivery
    Public key: Bm8J1RW3RxHp_-mx3lE7eAuYObfALvwurVjXtcaYFVA
    Key encryption strategy: insecure-plaintext-kms://
    Status: NEW
    Last updated: 2022-02-11T12:34:56-0700

Rollout the modified generated_adscert_keyring.json for changes to take effect.
```

After running this command, note that the `generated_adscert_keyring.json` file has been updated to include the new key material.

```
TODO
```

At this point, you would check in the `generated_adscert_keyring.json` file into your SCM so that continuous deployment builds would be able to pick up the new file state.

# Step 5: simulate completed keyring deployment

Normally we would need to wait until our release process completes and pushes the configuration to production, letting our CI environment tag in the SCM repository what keyring state represents the actually deployed version.  We'll simulate this step for illustration purposes.

Overwrite the `fully_deployed_adscert_keyring.json` file with the current content of `generated_adscert_keyring.json`:

```
$ cp generated_adscert_keyring.json fully_deployed_adscert_keyring.json
```

Submit this change to your source repository.

Now the `fully_deployed_adscert_keyring.json` file indicates to the tooling that DNS records can be published containing the new keys.

# Step 6: schedule DNS records for publication

Running `adscert vet` will now indicate that DNS records are due for publication.

Run the `adscert keyring autorotate` command which will make two changes:

* Modifies `generated_adscert_keyring.json` to indicate that the public key is ready for deployment to DNS
* Modifies `generated_adscert_dns_publication.json` to reflect the new DNS records that need publication

Note that the output provides a few DNS records that the system wants created.  These DNS publication requests are stored in `generated_adscert_dns_publication.json` 

Submit this change to your source repository.

# Step 7: publish DNS records

The prior step output a list of DNS records to publish.  This publishing action can be automated based on the content of the `generated_adscert_dns_publication.json` file, although for this example we can simply add these records using the DNS admin UI.

TODO: Add screenshot

To simulate the continuous deployment environment completing the DNS push action, overwrite the `fully_deployed_adscert_dns_publication.json` file with the content of `generated_adscert_dns_publication.json`:

```
$ cp generated_adscert_dns_publication.json fully_deployed_adscert_dns_publication.json
```

Submit this change to your source repository.

You can validate that your DNS servers match the DNS intent by running `adscert vet --checkdns`.

```
TODO: show output
```

# Step 9: try out signing a request

Now that you have a fully configured signing client, test it with a signed request.  Run the following command to sign an example URL invocation.

```
$ adscert authconnections testsign --local --demo https://example.com/
TODO: add output
```

This will output the signature that would be sent for the URL invocation (but it doesn't actually invoke the URL).  The `--demo` flag instructs the app to use a fake set of keys and DNS records for the receiving end to simulate example.com being a real environment.  The `--local

You can verify the signature message in a similar fashion.  In this case, the `--demo` flag instructs the test app to impersonate example.org's ads.cert Call Sign domain but otherwise use the signer's keys from DNS.

```
$ SIGNATURE_MESSAGE='from=signer.tk&from_key=Bm8J1R&invoking=example.com&nonce=numberusedonce&status=1&timestamp=210430T132456&to=example.org&to_key=000d8H; sigb=YWJjZGVmZ2hp&sigu=QUJDREVGR0hJ'
$ adscert authconnections testverify --local --demo $SIGNATURE_MESSAGE https://example.com/
TODO: output
```

# Step 10: run the signatory server

Let's now exercise the signatory server which more accurately reflects what you will use in a deployment.

Open a separate terminal and run the following command to start the signatory RPC server:

```
$ adscert signatory --rpc_server_port=8001 --monitoring_port=8002 --demo
```

This server will be able to sign and verify requests using your ads.cert keys.  As with the previous step, use the signature testing tool to generate a test signature, but this time using the RPC server to generate the signature:

```
$ adscert authconnections testsign --signatory=localhost:8001 --demo https://example.com/
TODO: output the RPC server request and response.
```

This time, instead of the test app using an in-process ads.cert signing implementation, it sent a signing request to the RPC server to obtain the signature message.

```
$ SIGNATURE_MESSAGE='from=signer.tk&from_key=Bm8J1R&invoking=example.com&nonce=numberusedonce&status=1&timestamp=210430T132456&to=example.org&to_key=000d8H; sigb=YWJjZGVmZ2hp&sigu=QUJDREVGR0hJ'
$ adscert authconnections testverify --signatory=localhost:8001 --demo $SIGNATURE_MESSAGE https://example.com/
TODO: output
```

# Step 11: send RPCs to the signatory server

Now that you have a working signatory server, let's send it RPCs like it would receive in ad serving.

The gRPC development kit comes with a [`grpc_cli` tool](https://github.com/grpc/grpc/blob/master/doc/command_line_tool.md) that you can use to interact with gRPC services.  If you want to take advantage of this tool, you will need to install it according to the instructions listed on the site.  If you do not want to install it, then simply reading through this section should still be sufficient.

TODO: Include a grpc_cli walkthorugh.

# Step 12: Package and deploy a Docker image

Now that you have a working environment, you can try packaging and deploying the ads.cert signatory server in a container environment.

For ease of versioning and deployment, we suggest packaging the ads.cert configuration files into a Docker image.  The default template generates a Dockerfile configuration to assist you with setting this up.

```
TODO: show example Dockerfile
```

Build the Docker image locally:

```
$ docker build . --tag github.com/cmlight/adscert-deploy-example:latest
```

This packages the config files alongside the adscert application binary.  You can see this result by running the deployment Docker image in an ephemeral container and executing an interactive shell:

```
$ docker run -it --entrypoint /bin/sh github.com/cmlight/adscert-deploy-example:latest
/app $ whoami
appuser
/app $ ls -la
...
-rwxr-xr-x    1 root     root      15503950 Feb 12 20:46 adscert
-rw-r--r--    1 root     root           174 Feb 10 22:34 adscert_config.yaml
```

This container can be deployed in a standard containerized environment.

TODO: show deployment in Kubernetes

# Step 13: Secure keys with encryption

Until this point, the `generated_adscert_keyring.json` file ccontains unencrypted keys that aren't suitable for use in a production environment, as anyone who has access to the source code repository would be able to see the plaintext keys.  In this step, we'll add in key encryption using a key management system (KMS) to encrypt the keyring private keys at rest.

This demo will show use of the Google Cloud Key Management Service (Cloud KMS), although these same concepts apply to other products.

First, create a KMS keyring using your preferred platform's configuration utilities.  In Google Cloud:

```
$ KMS_KEY_RING_NAME=my-adscert-key-ring
$ KMS_KEY_RING_LOCATION=global
$ KMS_KEY_NAME
$ gcloud kms keyrings create $KMS_KEY_RING_NAME \
    --location $KMS_KEY_RING_LOCATION
$ gcloud kms keys create $KMS_KEY_NAME \
    --keyring $KMS_KEY_RING_NAME \
    --location $KMS_KEY_RING_LOCATION \
    --purpose "encryption"    
```

Normally you will go through additional steps to limit access to this KMS configuration to specific role accounts, but this is outside the scope of this tutorial.

In the `adscert_config.yaml` file uncomment the following lines to enable the `google_gcp_kms` config:

```
key_security:
  key_management_system:
    google_gcp_kms:
      use_for_new_keys: true
      region_configs:
        - region: global
          resource_name: projects/GCP_PROJECT_ID/locations/KMS_KEY_RING_LOCATION/keyRings/KMS_KEY_RING_NAME/cryptoKeys/KMS_KEY_NAME
```

Update the resource_name to point to the key encryption key you just configured.

If you try running `adscert vet` now, you'll see that the application will reject the current configuration since two key management systems are configured as being used for new keys:

```
$ adscert vet

TODO: add output
```

Comment out the `use_for_new_keys` attribute in the `insecure_plaintext_keyring` configuration.  Now the `adscert vet` command should accept the configuration.

```
$ adscert vet

TODO: add output
```

Let's add another key to our ads.cert keyring.  (Normally we would not need to take these manual steps since the auto rotation logic would perform these steps for you, but we need to rotate the key prematurely.)

```
$ adscert keyring newkey --callsign=signer.tk --realm=delivery

TODO: add output

1 domains/realms due for new key generation on next keyring autorotation run.
```

This action added a placeholder in the `generated_adscert_keyring.json` file scheduing a new key creation.

Now run `adscert keyring autorotate` to simulate generating the actual key material:

```
TODO: add output
```

If you attempt to run the command from earlier, you'll see that it still generates signatures using the old key configuration.

```
$ adscert authconnections testsign --local --demo https://example.com/
```

Set the keyring to prefer the new key for signing operations:

```
$ adscert keyring setprimary --callsign=signer.tk --realm=delivery --keyid=abc123
```

Now the signing action will generate signatures using the new key.

```
$ adscert authconnections testsign --local --demo https://example.com/
```

# Step 14: monitor application performance

The ads.cert applications and APIs produce monitoring data you can incorporate into your broader monitoring infrastructure.

To inspect the monitoring signals exposed by the signatory server, run the signatory server instance and then navigate to the monitoring URL:

```
TODO: add info
```

In a separate terminal, run the "load test" utility which will send multiple simultaneous requests to the signatory server:

```
adscert authconnections loadtest signing --signatory=localhost:8001 --demo
```

This will send requests to the signatory server at a configurable rate.

Refresh the monitoring URL and note the changes as the monitoring counters increase.

Now try invoking the API directly usign the `grpc_cli` command line tool to see the monitoring information returned in the response.

```
TODO: show example
```
