This guide will show you how to set up a Scala app using the Play framework and PostgreSQL.

This guide assumes you have:

- An Aptible account
- An SSH key associated with your Aptible user account
- The [Aptible command line tool installed](/topics/cli/how-to-install-cli)

## 1. Provision Your App

Tell the Aptible API that you want to provision an application. Until you push code and trigger a build, Aptible uses this as a placeholder.

Use the `apps:create` command: `aptible apps:create $APP_HANDLE`

For example:

    aptible apps:create play-example

## 2. Add a Git Remote

Add a Git remote named "aptible":

    git remote add aptible git@beta.aptible.com:$APP_HANDLE.git

For example:

    git remote add aptible git@beta.aptible.com:play-example.git

## 3. Add a Dockerfile and a Procfile

A Dockerfile is a text file that contains the commands you would otherwise execute manually to build a Docker image. Aptible uses the resulting image to run your containers.

A Procfile explicitly declares what processes we should run for your app.

A few guidelines:

1. The files should be named "Procfile" and "Dockerfile": One word, initial capital letter, no extension.
2. Place both files in the root of your repository.
3. Be sure to commit them to version control.

Here is a sample Dockerfile for a conventional Play app:

    # Dockerfile
    FROM quay.io/aptible/java:oracle-java8

    # Install prerequisite tools
    RUN apt-get update && apt-get -y install wget unzip

    # Install Scala
    RUN cd /usr/share && \
        wget http://www.scala-lang.org/files/archive/scala-2.10.4.tgz && \
        tar xzf scala-2.10.4.tgz && \
        ln -s /usr/share/scala-2.10.4 /usr/share/scala && \
        rm -f /tmp/scala-2.10.4.tgz
    ENV PATH $PATH:/usr/share/scale/bin

    # Install Activator (omit if not needed)
    RUN cd /opt && \
        wget http://downloads.typesafe.com/typesafe-activator/1.2.10/typesafe-activator-1.2.10.zip && \
        unzip typesafe-activator-1.2.10.zip && \
        rm -f /opt/typesafe-activator-1.2.10.zip && \
        mv /opt/activator-1.2.10 /opt/activator
    ENV PATH $PATH:/opt/activator

    ADD . /app
    WORKDIR /app

    RUN activator -XX:+UseCodeCacheFlushing clean compile dist

    # Change the following command to match your actual app name
    RUN cd playexample/target/universal && unzip playexample-1.0-SNAPSHOT.zip

    EXPOSE 80

Here is a sample Procfile for a Play app. You will need to update this command to match your actual app name, instead of `playexample`:

    web: playexample/target/universal/*-SNAPSHOT/bin/playexample \
         -Dhttp.port=$PORT \
         -DapplyEvolutions.default=true \
         -Ddb.default.driver=org.postgresql.Driver \
         -Ddb.default.url=$DATABASE_URL

Alternatively, you can omit the runtime database configuration options and instead use your `conf/application.conf` file.

## 4. Provision and Connect a Database

By default, `aptible db:create $DB_HANDLE` will provision a 10GB PostgreSQL database.

`aptible db:create` will return a connection string on success. The host value is mapped to a private subnet within your stack and cannot be used to connect from the outside Internet. Your containerized app can connect, however.

Add the database connection string to your app as an environment variable:

    aptible config:set DATABASE_URL=$CONNECTION_STRING

To connect locally, see [the `aptible db:tunnel` command](/topics/cli/how-to-connect-to-database-from-outside/).


## 5. Deploy Your App

Push to the master branch of the Aptible Git remote:

    git push aptible master

If your app deploys successfully, a message will appear near the end of the remote output with a default VHOST:


    VHOST play-example.on-aptible.com provisioned.

In this example, once the ELB provisions you could visit play-example.on-aptible.com to test out your app.
