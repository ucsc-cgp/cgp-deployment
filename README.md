# dcc-ops

## About

This repository contains our Docker-compose and setup bootstrap scripts used to create a deployment of the [UCSC Genomic Institute's](http://ucsc-cgl.org) Computational Genomics Platform for AWS.  The system is designed to receive genomic data, run analysis at scale on the cloud, and return analyzed results to authorized users.  It uses, supports, and drives development of several key GA4GH APIs and open source projects. In many ways it is the generalization of the [PCAWG](https://dcc.icgc.org/pcawg) cloud infrastructure developed for that project and a potential reference implementation for the [NIH Commons](https://datascience.nih.gov/commons) concept.

## Components

The system has components fulfilling a range of functions, all of which are open source and can be used independently or together.

![Computational Genomics Platform architecture](docs/dcc-arch.png)

These components are setup with the install process available in this repository:

* [Spinnaker](spinnaker/README.md): our data submission and validation system
* [Redwood](redwood/README.md): our cloud data storage and indexer based on the ICGC Cloud Storage system
* [Boardwalk](boardwalk/README.md): our file browsing portal on top of Redwood
* [Consonance](consonance/README.md): our multi-cloud workflow orchestration system
* [Action Service](action/README.md): a Python-based toolkit for automating workflow execution

These are related projects that are either already setup and available for use on the web or are used by components above.

* [Dockstore](http://dockstore.org): our workflow and tool sharing platform
* [Toil](https://github.com/BD2KGenomics/toil): our workflow engine, these workflows are shared via Dockstore

## Installing the Platform

These directions below assume you are using AWS.  We will include additional cloud instructions as `dcc-ops` matures.

### Collecting Information

Make sure you have:

* your AWS key/secret key
* you know what region you're running in e.g. `us-west-2`

### Starting an AWS VM

Use the AWS console or command line tool to create a host. For example:

* Ubuntu Server 16.04
* r4.xlarge
* 250GB disk

We will refer to this as the host VM throughout the documentation below and it is the machine running all the Docker containers for each of the components below.

You should make a note of your security group name and ID and ensure you can connect via ssh.

### AWS Tasks

Make sure you do the following:

* assign an Elastic IP (a static IP address) to your instance
* open inbound ports on your security group
    * 80 <- world
    * 8080 <- world
    * 22 <- world
    * 443 <- world
    * all TCP <- the elastic IP of the VM (Make sure you add /32 to the Elastic IP)
    * all TCP <- the security group itself

### Setup for Redwood

Here is a summary of what you need to do. See the Redwood [README](redwood/README.md) for details.

#### Re-route Service Endpoints
Redwood exposes storage, metadata, auth services. Each of these should be made subdomains of your "base domain".
* Make sure you have a domain name associated with your Elastic IP ('example.com')
* Have the subdomains 'auth', 'metadata', and 'storage' point to the same Elastic IP ('auth.example.com', 'metadata.example.com', and 'storage.example.com' and 'example.com' all resolve to the same Elastic IP)

#### Make your S3 Bucket
* On the AWS console, go to S3 and create a bucket.
* Assign it a name. Keep note of the name given to it.
* Get the S3 endpoint. It dependent on your region. See [here](http://docs.aws.amazon.com/general/latest/gr/rande.html#s3_region) for the list.

#### Create an AWS IAM Encryption Key
* Go [here](http://docs.aws.amazon.com/kms/latest/developerguide/create-keys.html) and follow the instruction for making an AWS IAM Encryption key. Make sure you create it in same region where you created your VM!
* Take note of the AWS IAM Encryption Key ID. You can find it in the AWS console > Services > IAM > Encryption Keys > [your key's details page] > ARN. It is the last part of the ARN (e.g. *arn:aws:kms:us-east-1:862902209576:key/* **0aaad33b-7ead-44be-a56e-3d00c8777042**

Now we're ready to install Redwood.

### Setup for Consonance

See the Consonance [README](consonance/README.md) for details.  Consonance assumes you have an SSH key created and uploaded to a location on your host VM.  Other than that, there are no additional pre-setup tasks.

#### Consonance CLI on the Host VM

You probably want to install the Consonance command line on the host VM so you can submit work from outside the Docker containers running the various Consonance services.  Likewise, you can install the CLI on other hosts and submit work to the queue.

Download the `consonance` command line from the Consonance releases page:

https://github.com/Consonance/consonance/releases

For example:

```
wget https://github.com/Consonance/consonance/releases/download/2.0.0-alpha.15/consonance
sudo mv consonance /usr/local/bin/
sudo chmod a+x /usr/local/bin/consonance
# running the command will install the tool and prompt you to enter your token, please get the token after running install_bootstrap
consonance
```

Follow the interactive directions for setting up this CLI.  You will need the elastic IP you setup previously (or, better yet, the "base domain" from above).

### Setup for Boardwalk

Here is a summary of what you need to do. See the Boardwalk [README](boardwalk/README.md) for details.

#### Create a Google Oauth2 app

You need to create a Google Oauth2 app to enable Login and token download from the dashboard. If you don't want to enable this on the dashboard during installation, simply enter a random string when asked for the *Google Client ID* and the *Google Client Secret*. You can consult [here](http://bitwiser.in/2015/09/09/add-google-login-in-flask.html#creating-a-google-project) under "Creating A Google Project" if you want to read more details. Here is a summary of what you need to do:

* Go to [Google's Developer Console](https://console.developers.google.com/).
* On the upper left side of the screen, click on the drop down button.
* Create a project by clicking on the plus sign on the pop-up window.
* On the next pop up window, add a name for your project.
* Once you create it, click on the "Credentials" section on the left hand side.
* Click on the "OAuth Consent Screen". Fill out a product name and choose an email for the Google Application. Fill the rest of the entries as you see fit for your purposes, or leave them blank, as they are optional. Save it.
* Go to the "Credentials" tab. Click on the "Create Credentials" drop down menu and choose "OAuth Client ID".
* Choose "Web Application" from the menu. Assign it a name.
* Under "Authorized JavaScript origins", enter `http://<YOUR_SITE>`. Press Enter. Add a second entry, same as the first one, but use *https* instead of *http*
* Under "Authorized redirect URIs", enter `http://<YOUR_SITE>/gCallback`. Press Enter. Add a second entry, same as the first one, but use *https* instead of *http*
* Click "Create". A pop up window will appear with your Google Client ID and Google Client Secret. Save these. If you lose them, you can always go back to the Google Console, and click on your project; the information will be there.

Please note: at this point, the dashboard only accepts login from emails with a 'ucsc.edu' domain. In the future, it will support different email domains.

### Running the Installer

Once the above setup is done, clone this repository onto your server and run the bootstrap script

    # note, you may need to checkout the particular branch or release tag you are interested in...
    git clone https://github.com/BD2KGenomics/dcc-ops.git && cd dcc-ops && sudo bash install_bootstrap

#### Installer Question Notes

The `install_bootstrap` script will ask you to configure each service interactively.

* Consonance
* Redwood
  * Install in prod mode
  * If the base URL is _example.com_, then _storage.example.com_, _metadata.example.com_, _auth.example.com_, and _example.com_ should resolve via DNS to your server elastic IP.
  * Enter your AWS Key and Secret Key when requested. Redwood will use these to sign requests for upload and download to your S3 bucket
  * On question 'What is your AWS S3 bucket?', put the name of the s3 bucket you created for Redwood.
  * On question 'What is your AWS S3 endpoint?', put the S3 endpoint pertaining to your region. See [here](http://docs.aws.amazon.com/general/latest/gr/rande.html#s3_region).
  * On question 'What is your AWS IAM KMS key ID?', put your encryption key ID (See 'Create an AWS IAM Encryption Key" above). If you don't want server-side encryption, you can leave this blank.
* Boardwalk
  * Install in prod mode
  * On question `What is your Google Client ID?`, put your Google Client ID. See [here](http://bitwiser.in/2015/09/09/add-google-login-in-flask.html#creating-a-google-project)
  * On question `What is your Google Client Secret?`, put your Google Client Secret. See [here](http://bitwiser.in/2015/09/09/add-google-login-in-flask.html#creating-a-google-project)
  * On question `What is your DCC Dashboard Host?`, put the domain name resolving to your Virtual Machine (e.g. `example.com` your base URL mentioned above in the Redwood install)
  * On question `What is the user and group that should own the files from the metadata-indexer?`, type the `USER:GROUP` pair you want the files downloaded by the indexer to be owned by. The question will show the current `USER:GROUP` pair for the current home directory. Highly recommended to type the same value in there (e.g. `1000:1000`)
  * On question `How should the database for billing should be called?`, type the name to be assigned to the billing database.
  * On question `What should the username be for the billing database?`, type the username for the billing database.
  * On question `What should the username password be for the billing database?`, type some password for the billing database.
  * On question `What is the AWS profile?`, type some random string (DEV, PROD)
  * On question `What is the AWS Access key ID?`, type some random string (DEV, PROD)
  * On question `What is the AWS secret access key?`, type some random string (DEV, PROD)
  * On question `What is the Consonance Address?`, type some random string (DEV, PROD)
  * On question `What is the Consonance Token`, type some random string (DEV, PROD)
  * On question `What is the Luigi Server?`, type some random string (DEV, PROD)
  * On question `What is the Postgres Database name for the action service?`, type the name to be assigned to the action service database.
  * On question `What is the Postgres Database user for the action service?`, type the username to be assigned to the the action service database.
  * On question `What is the Postgres Database password for the action service?`, type the password to be assigned to the action service database.

* Common
  * Installing in `dev`mode will use letsencrypt's staging service, which won't exhaust your certificate's limit, but will install fake ssl certificates. `prod` mode will install official SSL certificates.  

Once the installer completes, the system should be up and running. Congratulations! See `docker ps` to get an idea of what's running.

## Post-Installation

### Confirm Proper Function

To test that everything installed successfully, you can run `cd test && ./integration.sh`. This will do an upload and download with core-client and check the results.

### Upload and Download

End users should be directed to use the [quay.io/ucsc_cgl/core-client](https://quay.io/repository/ucsc_cgl/core-client)
docker image as documented in its [README](https://github.com/BD2KGenomics/dcc-spinnaker-client/blob/develop/README.md).
The `test/integration.sh` file also demonstrates normal core-client usage.

Here is a sample command you can run from the `test` folder to do an upload:

**NOTE:** Make sure you create an access token for yourself first. You can do so by running within `dcc-ops` the command `redwood/admin/bin/redwood token create -u myemail@ucsc.edu -s 'aws.upload aws.download'`. This will create a global token that you can use for testing for upload and download on any project. End users should only be given project-specific scopes like _aws.PROJECT.upload_.

```
sudo docker run --rm -it -e ACCESS_TOKEN=<your_token> -e REDWOOD_ENDPOINT=<your_url.com> \
            -v $(pwd)/manifest.tsv:/dcc/manifest.tsv -v $(pwd)/samples:/samples \
            -v $(pwd)/outputs:/outputs quay.io/ucsc_cgl/core-client:1.1.0-alpha spinnaker-upload \
            --force-upload /dcc/manifest.tsv
```
Here is a sample command you can run to download the using a manifest file. On the dashboard, go to the "BROWSER" tab, and click on "Download Manifest" at the bottom of the list. Save this file, and run the following command. This will download the files specified from the manifest:

```
sudo docker run --rm -it -e ACCESS_TOKEN=<your_token> -e REDWOOD_ENDPOINT=<your_url.com> \
            -v $(pwd)/<your_manifest_file_name.tsv>:/dcc/dcc-spinnaker-client/data/manifest.tsv \
            -v $(pwd)/samples:/samples -v $(pwd)/outputs:/outputs \
            -v $(pwd):/dcc/data quay.io/ucsc_cgl/core-client:1.1.0-alpha \
            redwood-download /dcc/dcc-spinnaker-client/data/manifest.tsv /dcc/data/
```


### Troubleshooting

If something goes wrong, you can [open an issue](https://github.com/BD2KGenomics/dcc-ops/issues/new) or [contact a human](https://github.com/BD2KGenomics/dcc-ops/graphs/contributors).

### Tips

* This [blog post](https://www.digitalocean.com/community/tutorials/how-to-remove-docker-images-containers-and-volumes) is helpful if you want to clean up previous images/containers/volumes.

### To Do

* should use a reference rather than checkin the consonance directory, that ends up creating duplication which is not desirable
* the bootstrapper should install Java, Dockstore CLI, and the Consonance CLI
