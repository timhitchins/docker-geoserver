# docker-geoserver

A simple docker container that runs GeoServer influenced by this docker
recipe: https://github.com/eliotjordan/docker-geoserver/blob/master/Dockerfile

## Getting the image

There are various ways to get the image onto your system:

The preferred way (but using most bandwidth for the initial image) is to
get our docker trusted build like this:

```shell
docker pull kartoza/geoserver
```

### To build yourself with a local checkout using the build script:

Edit the build script to change the following variables:

- The variables below represent the latest stable release you need to build. i.e 2.15.2

   ```text
   BUGFIX=2
   MINOR=15
   MAJOR=2
   ```

```shell
git clone git://github.com/kartoza/docker-geoserver
cd docker-geoserver
./build.sh
```

Ensure that you look at the build script to see what other build arguments you can include whilst building your image.

If you do not intend to jump between versions you need to specify that in the build script.

### Building with war file from a URL

If you need to build the image with a custom GeoServer war file that will be downloaded from a server, you
can pass the war file url as a build argument to docker, example:

```shell
docker build --build-arg WAR_URL=http://download2.nust.na/pub4/sourceforge/g/project/ge/geoserver/GeoServer/2.13.0/geoserver-2.13.0-war.zip --build-arg GS_VERSION=2.13.0
```

**Note: war file version should match the version number provided by `GS_VERSION` argument otherwise we will have a mismatch of plugins and GeoServer installed.**

### Building with Oracle JDK

Download `jdk-8u201-linux-x64.tar.gz` or the latest version from [Oracle Java](https://www.oracle.com) and save the contents into
the resources folder. This used to be done by the setup scripts but no longer works due to the changes
in the licencing terms from Oracle which require users to login to their site.

To replace OpenJDK Java with the Oracle JDK, set build-arg `ORACLE_JDK=true`:

```shell
docker build --build-arg ORACLE_JDK=true --build-arg GS_VERSION=2.13.0 -t kartoza/geoserver .
```

### Building with plugins

Inspect setup.sh to confirm which plugins (community modules or standard plugins) you want to include in
the build process, then add them in their respective sections in the script.

You should ensure that the plugins match the  version for the GeoServer WAR zip file.

### Removing Tomcat extras during build

To remove Tomcat extras including docs, examples, and the manager webapp, set the
`TOMCAT_EXTRAS` build-arg to `false`:

```shell
docker build --build-arg TOMCAT_EXTRAS=false --build-arg GS_VERSION=2.13.0 -t kartoza/geoserver .
```

### Building with specific version of  Tomcat

To build using a specific tagged release for tomcat image set the
`IMAGE_VERSION` build-arg to `8-jre8`: See the [dockerhub tomcat](https://hub.docker.com/_/tomcat/)
to choose which tag you need to build against.

```shell
docker build --build-arg IMAGE_VERSION=8-jre8 --build-arg GS_VERSION=2.13.0 -t kartoza/geoserver:2.13.0 .
```

### Building with file system overlays (advanced)

The contents of `resources/overlays` will be copied to the image file system
during the build. For example, to include a static Tomcat `setenv.sh`,
create the file at `resources/overlays/usr/local/tomcat/bin/setenv.sh`.

You can use this functionality to write a static GeoServer directory to
`/opt/geoserver/data_dir`, include additional jar files, and more.

Overlay files will overwrite existing destination files, so be careful!

#### Build with CORS Support

The contents of `resources/overlays` will be copied to the image file system
during the build. For example, to include a static web xml with CORS support `web.xml`,
create the file at `resources/overlays/usr/local/tomcat/conf/web.xml`.

## Run (manual docker commands)

**Note:** You probably want to use docker-compose for running as it will provide
a repeatable orchestrated deployment system.

You probably want to also have PostGIS running too. To create a running
container do:

```shell
docker run --name "postgis" -d -t kartoza/postgis:9.4-2.1
docker run --name "geoserver"  --link postgis:postgis -p 8080:8080 -d -t kartoza/geoserver
```

You can also use the following environment variables to pass a
user name and password to PostGIS:

* `-e USERNAME=<PGUSER>`
* `-e PASS=<PGPASSWORD>`

You can also use the following environment variables to pass arguments to GeoServer:

* `GEOSERVER_DATA_DIR=<PATH>`
* `ENABLE_JSONP=<true or false>`
* `MAX_FILTER_RULES=<Any integer>`
* `OPTIMIZE_LINE_WIDTH=<false or true>`
* `FOOTPRINTS_DATA_DIR=<PATH>`
* `GEOWEBCACHE_CACHE_DIR=<PATH>`
* `GEOSERVER_ADMIN_PASSWORD=<password>`

In order to prevent clickjacking attacks GeoServer defaults to 
setting the X-Frame-Options HTTP header to SAMEORIGIN. Controls whether the X-Frame-Options 
filter should be set at all. Default is true
* `XFRAME_OPTIONS="true"`
* Tomcat properties:

  * You can change the variables based on [geoserver container considerations](http://docs.geoserver.org/stable/en/user/production/container.html). These arguments operate on the `-Xms` and `-Xmx` options of the Java Virtual Machine
  * `INITIAL_MEMORY=<size>` : Initial Memory that Java can allocate, default `2G`
  * `MAXIMUM_MEMORY=<size>` : Maximum Memory that Java can allocate, default `4G`

### Control flow properties

The control flow module is installed by default and it is used to manage request in geoserver. In order
to customise it based on your resources and use case read the instructions from
[documentation](http://docs.geoserver.org/latest/en/user/extensions/controlflow/index.html). 
These options can be controlled by environment variables

* Control flow properties environment variables

    if a request waits in queue for more than 60 seconds it's not worth executing,
    the client will  likely have given up by then
    * REQUEST_TIMEOUT=60 
    don't allow the execution of more than 100 requests total in parallel
    * PARARELL_REQUEST=100 
    don't allow more than 10 GetMap in parallel
    * GETMAP=10 
    don't allow more than 4 outputs with Excel output as it's memory bound
    * REQUEST_EXCEL=4 
    don't allow a single user to perform more than 6 requests in parallel
    (6 being the Firefox default concurrency level at the time of writing)
    * SINGLE_USER=6 
    don't allow the execution of more than 16 tile requests in parallel
    (assuming a server with 4 cores, GWC empirical tests show that throughput
    peaks up at 4 x number of cores. Adjust as appropriate to your system)
    * GWC_REQUEST=16 
    * WPS_REQUEST=1000/d;30s


**Note:**

### Changing GeoServer password on runtime

The default GeoServer user is 'admin' and the password is 'geoserver'. You can pass the environment variable
GEOSERVER_ADMIN_PASSWORD to  change it on runtime.

```shell
docker run --name "geoserver"  -e GEOSERVER_ADMIN_PASSWORD=myawesomegeoserver -p 8080:8080 -d -t kartoza/geoserver
```

## Run (automated using docker-compose)

We provide a sample ``docker-compose.yml`` file that illustrates
how you can establish a GeoServer + PostGIS + GeoGig orchestrated environment
with nightly backups that are synchronised to your backup server via btsync.

If you are **not** interested in the backups, GeoGig and btsync options, comment
out those services in the ``docker-compose.yml`` file.

If you start the stack using the compose file make sure you login into GeoServer using username:`admin`
and password:`myawesomegeoserver` as specified by the env file `geoserver.env`

Please read the ``docker-compose``
[documentation](https://docs.docker.com/compose/) for details
on usage and syntax of ``docker-compose`` - it is not covered here.

If you **are** interested in btsync backups, install [Resilio sync]
on your desktop NAS or other backup  destination and create two
folders:

* one for database backup dumps
* one for geoserver data dir

Then make a copy of each of the provided EXAMPLE environment files e.g.:

```shell
cp docker-env/btsync-db.env.EXAMPLE docker-env/btsync-db.env
cp docker-env/btsync-media.env.EXAMPLE docker-env/btsync-media.env
```

Then edit the two env files, placing your Read/Write Resilio keys
in the place provided.

To run the example do:

```shell
docker-compose up
```

Which will run everything in the foreground giving you the opportunity
to peruse logs and see that everything spins up nicely.

Once all services are started, test by visiting the GeoServer landing
page in your browser: [http://localhost:8600/geoserver](http://localhost:8600/geoserver).

To run in the background rather, press ``ctrl-c`` to stop the
containers and run again in the background:

```shell
docker-compose up -d
```

**Note:** The ``docker-compose.yml`` **uses host based volumes** so
when you remove the containers, **all data will be kept**. Using host based volumes
 ensures that your data persists between invocations of the compose file. If you need
 to delete the container data you need to run `docker volume prune`. Pruning the volumes will
 remove all the storage volumes that are not in use so users need to be careful of such a move.
 Either set up btsync (and test to verify that your backups are working, we take
**no responsibility** if the examples provided here do not produce
a reliable backup system).

## Run (automated using rancher)

An even nicer way to run the examples provided is to use our Rancher
Catalogue Stack for GeoServer. See [http://rancher.com](http://rancher.com)
for more details on how to set up and configure your Rancher
environment. Once Rancher is set up, use the Admin -> Settings menu to
add our Rancher catalogue using this URL:

https://github.com/kartoza/kartoza-rancher-catalogue

Once your settings are saved open a Rancher environment and set up a
stack from the catalogue's 'Kartoza' section - you will see
GeoServer listed there.

If you want to synchronise your GeoServer settings and database backups
(created by the nightly backup tool in the stack), use [Resilio
sync](https://www.Resilio.com/) to create two Read/Write keys:

* one for database backups
* one for GeoServer media backups

**Note:** Resilio sync is not Free Software. It is free to use for
individuals. Business users need to pay - see their web site for details.

You can try a similar approach with Syncthing or Seafile (for free options)
or Dropbox or Google Drive if you want to use another commercial product. These
products all have one limitation though: they require interaction
to register applications or keys. With Resilio Sync you can completely
automate the process without user intervention.

## Storing data on the host rather than the container.

Docker volumes can be used to persist your data.

If you need to use geoserver data directory that contains sample examples and configurations download
it from [geonode](http://build.geonode.org/geoserver/latest/) site as indicated below:

```shell

# Example - ${GS_VERSION} is the geoserver version i.e 2.13.0
wget http://build.geonode.org/geoserver/latest/data-2.13.x.zip
unzip data-2.13.x.zip -d ~/geoserver_data
cp scripts/controlflow.properties ~/geoserver_data
chmod -R a+rwx ~/geoserver_data
docker run -d -p 8580:8080 --name "geoserver" -v $HOME/geoserver_data:/opt/geoserver/data_dir kartoza/geoserver:${GS_VERSION}
```

Create an empty data directory to use to persist your data.

```shell
mkdir -p ~/geoserver_data && chmod -R a+rwx ~/geoserver_data
docker run -d -v $HOME/geoserver_data:/opt/geoserver/data_dir kartoza/geoserver
```


## Credits

* Tim Sutton (tim@kartoza.com)
* Shane St Clair (shane@axiomdatascience.com)
* Alex Leith (alexgleith@gmail.com)
* Admire Nyakudya (admire@kartoza.com)
* Gavin Fleming (gavin@kartoza.com)

=====================================================================
PERSONALIZED INSTALL INSTRUCTIONS
=====================================================================

## Setup for server

### Install Docker

Install Docker (from https://docs.docker.com/install/linux/docker-ce/ubuntu/#set-up-the-repository)

#### Setup the Repo

1. Update the apt package index:
```
sudo apt-get update
```

2. Install packages to allow apt to use a repository over HTTPS:
```
sudo apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg-agent \
    software-properties-common
```

3. Add Docker’s official GPG key:
```
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
```

Verify that you now have the key with the fingerprint `9DC8 5822 9FC7 DD38 854A E2D8 8D81 803C 0EBF CD88`, 
by searching for the last 8 characters of the fingerprint.
```
sudo apt-key fingerprint 0EBFCD88
```

4. Use the following command to set up the stable repository
```
sudo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"
```

#### Install Docker Engine -Community

1. Update the apt package index.
```
sudo apt-get update
```

2. Install the latest version of Docker Engine - Community and containerd
```
sudo apt-get install docker-ce docker-ce-cli containerd.io
```

4. Verify that Docker Engine - Community is installed correctly by running the `hello-world` image.
```
sudo docker run hello-world
```

#### Install Docker Compose

Manually install Docker Compose (from https://docs.docker.com/compose/install/#install-compose)

1. Run this command to download the current stable release of Docker Compose:
```
sudo curl -L "https://github.com/docker/compose/releases/download/1.25.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
```

2. Apply executable permissions to the binary:
```
sudo chmod +x /usr/local/bin/docker-compose
```

3. Test the intallation
```
docker-compose --version
```

#### Run Docker without Sudo 

1. Create the `docker` group
```
sudo groupadd docker
```

2. Add your user to the `docker` group
```
sudo usermod -aG docker $USER
```

3. On Linux, you can also run the following command to activate the changes to groups:
```
newgrp docker
```

4. Verify that you can run `docker` commands without `sudo`.
```
docker run hello-world
```

#### Start on Boot

Follow these instructions so that Docker and its services start automatically on boot.
(from https://docs.docker.com/install/linux/linux-postinstall/#configure-docker-to-start-on-boot)

1. systemd
```
sudo systemctl enable docker
```
To disable this behavior, use disable instead.
```
sudo systemctl disable docker
```

### Setup Geoserver

Clone this repo to geoserver_config directory
```
mkdir ~/geoserver_config && cd ~/geoserver_config
git clone https://github.com/timhitchins/docker-geoserver.git
```

Create the directories that will be mapped to the pg11 and geoserver data containers.  
Keep in mind that these will need to be updated in the `docker-compose.yml` file as well.

```
mkdir -p ~/geoserver_config/geoserver_data && \
chmod -R a+rwx ~/geoserver_config/geoserver_data
mkdir -p ~/geoserver_config/pg_data && \
chmod -R a+rwx ~/geoserver_config/pg_data
```

Change the local pass/user files.
The password is set in `geoserver.env` as the var `GEOSERVER_ADMIN_PASSWORD`.
CHANGE THIS PASSWORD FROM THE DEFAULT IN THIS FILE!
```
nano ~/geoserver_config/docker-geoserver/docker-env/geoserver.env
```

Assumming that the `docker-compose.yml` has been updated to map to the new dirs, build and run from root:

```
cd ~/geoserver_config/docker-geoserver
docker-compose up -d
```

The go to http://<hostname>:8600/geoserver and sign-in with user and password.

