[![experimental](http://badges.github.io/stability-badges/dist/experimental.svg)](http://github.com/badges/stability-badges)
[![Gradle Plugin Portal](https://img.shields.io/badge/gradle%20plugin-v0.1.1-blue.svg)](https://plugins.gradle.org/plugin/com.google.cloud.tools.jib)
[![Gitter version](https://img.shields.io/gitter/room/gitterHQ/gitter.svg)](https://gitter.im/google/jib)

# Jib - Containerize your Gradle Java project

Jib is [Gradle](https://gradle.org/) plugin for building Docker and OCI images for your Java applications.

For information about the project, see the [Jib project README](..).
For the Maven plugin, see the [jib-maven-plugin project](../jib-maven-plugin).

## Upcoming Features

These features are not currently supported but will be added in later releases.

* Support for WAR format
* Export to a Docker context
* Run and debug the built container

## Quickstart

### Setup

*Make sure you are using Gradle version 4.6 or later.*

In your Gradle Java project, add the plugin to your `build.gradle`:

```groovy
plugins {
  id 'com.google.cloud.tools.jib' version '0.1.1'
}
```

See the [Gradle Plugin Portal](https://plugins.gradle.org/plugin/com.google.cloud.tools.jib) for more details.

## Configuration

Configure the plugin by setting the image to push to:

#### Using [Google Container Registry (GCR)](https://cloud.google.com/container-registry/)...

*Make sure you have the [`docker-credential-gcr` command line tool](https://cloud.google.com/container-registry/docs/advanced-authentication#docker_credential_helper). Jib automatically uses `docker-credential-gcr` for obtaining credentials. See [Authentication Methods](#authentication-methods) for other ways of authenticating.*

For example, to build the image `gcr.io/my-gcp-project/my-app`, the configuration would be:

```groovy
jib.to.image = 'gcr.io/my-gcp-project/my-app'
```

#### Using [Amazon Elastic Container Registry (ECR)](https://aws.amazon.com/ecr/)...

*Make sure you have the [`docker-credential-ecr-login` command line tool](https://github.com/awslabs/amazon-ecr-credential-helper). Jib automatically uses `docker-credential-ecr-login` for obtaining credentials. See [Authentication Methods](#authentication-methods) for other ways of authenticating.*

For example, to build the image `aws_account_id.dkr.ecr.region.amazonaws.com/my-app`, the configuration would be:

```groovy
jib.to.image = 'aws_account_id.dkr.ecr.region.amazonaws.com/my-app'
```

#### Using [Docker Hub Registry](https://hub.docker.com/)...

*Make sure you have a [docker-credential-helper](https://github.com/docker/docker-credential-helpers#available-programs) set up. For example, on macOS, the credential helper would be `docker-credential-osxkeychain`. See [Authentication Methods](#authentication-methods) for other ways of authenticating.*

For example, to build the image `my-docker-id/my-app`, the configuration would be:

```groovy
jib.to.image = 'my-docker-id/my-app'
```

#### *TODO: Add more examples for common registries.*

### Build Your Image

Build your container image with:

```shell
gradle build jib
```

Subsequent builds are much faster than the initial build. 

If you want to clear Jib's build cache and force it to re-pull the base image and re-build the application layers, run:

```shell
gradle clean build jib
```

*Having trouble? Let us know by [submitting an issue](/../../issues/new), contacting us on [Gitter](https://gitter.im/google/jib), or posting to the [Jib users forum](https://groups.google.com/forum/#!forum/jib-users).*

### Run `jib` with each build

You can also have `jib` run with each build by attaching it to the `build` task:

```groovy
tasks.build.finalizedBy tasks.jib
```

Then, ```gradle build``` will build and containerize your application.

### Export to a Docker context

*Not yet supported*

## Extended Usage

The plugin provides the `jib` extension for configuration with the following options for customizing the image build:

Field | Type | Default | Description
--- | --- | --- | ---
`from` | [`from`](#from-closure) | See [`from`](#from-closure) | Configures the base image to build your application on top of.
`to` | [`to`](#to-closure) | *Required* | Configures the target image to build your application to.
`jvmFlags` | `List<String>` | *None* | Additional flags to pass into the JVM when running your application.
`mainClass` | `String` | Uses the main class defined in the `jar` task | The main class to launch your application from.
`reproducible` | `boolean` | `true` | Building with the same application contents always generates the same image.<br>Note that this does NOT preserve file timestamps and ownership.
`format` | `String` | `Docker` | Use `OCI` to build an [OCI container image](https://www.opencontainers.org/).
`useProjectOnlyCache` | `boolean` | `false` | If set to true, Jib does not share a cache between different Maven projects.

<a name="from-closure"></a>`from` is a closure with the following properties:

Property | Type | Default | Description
--- | --- | --- | ---
`image` | `String` | `gcr.io/distroless/java` | The image reference for the base image.
`credHelper` | `String` | *None* | Suffix for the credential helper that can authenticate pulling the base image (following `docker-credential-`).
`auth` | [`auth`](#auth-closure) | *None* | Specify credentials directly (alternative to `credHelper`).

<a name="to-closure"></a>`to` is a closure with the following properties:

Property | Type | Default | Description
--- | --- | --- | ---
`image` | `String` | *Required* | The image reference for the target image.
`credHelper` | `String` | *None* | Suffix for the credential helper that can authenticate pulling the base image (following `docker-credential-`).
`auth` | [`auth`](#auth-closure) | *None* | Specify credentials directly (alternative to `credHelper`).

<a name="auth-closure"></a>`auth` is a closure with the following properties (see [Using Specific Credentials](#using-specific-credentials)):

Property | Type
--- | ---
`username` | `String`
`password` | `String`

### Example

In this configuration, the image is:
* Built from a base of `openjdk:alpine` (pulled from Docker Hub)
* Pushed to `localhost:5000/my-image:built-with-jib`
* Runs by calling `java -Xms512m -Xdebug -Xmy:flag=jib-rules -cp app/libs/*:app/resources:app/classes mypackage.MyApp`
* Reproducible
* Built as OCI format

```groovy
jib {
  from {
    image = 'openjdk:alpine'
  }
  to {
    image = 'localhost:5000/my-image/built-with-jib'
    credHelper = 'osxkeychain'
  }
  jvmFlags = ['-Xms512m', '-Xdebug', '-Xmy:flag=jib-rules']
  mainClass = 'mypackage.MyApp'
  reproducible = true
  format = 'OCI'
}
```

### Authentication Methods

Pushing/pulling from private registries require authorization credentials. These can be [retrieved using Docker credential helpers](#using-docker-credential-helpers)<!-- or in the `jib` extension-->. If you do not define credentials explicitly, Jib will try to [use credentials defined in your Docker config](/../../issues/101) or infer common credential helpers.

#### Using Docker Credential Helpers

Docker credential helpers are CLI tools that handle authentication with various registries.

Some common credential helpers include:

* Google Container Registry: [`docker-credential-gcr`](https://cloud.google.com/container-registry/docs/advanced-authentication#docker_credential_helper)
* AWS Elastic Container Registry: [`docker-credential-ecr-login`](https://github.com/awslabs/amazon-ecr-credential-helper)
* Docker Hub Registry: [`docker-credential-*`](https://github.com/docker/docker-credential-helpers)
<!--* Azure Container Registry: [`docker-credential-acr-*`](https://github.com/Azure/acr-docker-credential-helper)
-->

Configure credential helpers to use by specifying them as a `credHelper` for their respective image in the `jib` extension.

*Example configuration:* 
```xml
jib {
  from {
    image = 'aws_account_id.dkr.ecr.region.amazonaws.com/my-base-image'
    credHelper = 'ecr-login'
  }
  to {
    image = 'gcr.io/my-gcp-project/my-app'
    credHelper = 'gcr'
  }
}
```

#### Using Specific Credentials

You can specify credentials directly in the extension for the `from` and/or `to` images.

```xml
jib {
  from {
    image = 'aws_account_id.dkr.ecr.region.amazonaws.com/my-base-image'
    auth {
      username = USERNAME // Defined in 'gradle.properties'.
      password = PASSWORD
    }
  }
  to {
    image = 'gcr.io/my-gcp-project/my-app'
    auth {
      username = 'oauth2accesstoken'
      password = 'gcloud auth print-access-token'.execute().text.trim()
    }
  }
}
```

These credentials can be stored in `gradle.properties`, retrieved from a command (like `gcloud auth print-access-token`), or read in from a file. 

For example, you can use a key file for authentication (for GCR, see [Using a JSON key file](https://cloud.google.com/container-registry/docs/advanced-authentication#using_a_json_key_file)):

```xml
jib {
  to {
    image = 'gcr.io/my-gcp-project/my-app'
    auth {
      username = '_json_key'
      password = file('keyfile.json').text
    }
  }
}
```

## How Jib Works

See the [Jib project README](/../../#how-jib-works).

## Known Limitations

These limitations will be fixed in later releases.

* Does not build directly to a Docker daemon.
* Pushing to Azure Container Registry is not currently supported.

## Frequently Asked Questions (FAQ)

See the [Jib project README](/../../#frequently-asked-questions-faq).

## Community

See the [Jib project README](/../../#community).
