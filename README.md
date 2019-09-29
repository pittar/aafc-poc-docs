# Agriculure and Agri-Food Canada: OpenShift Proof of Concept

## Introduction

Agriculture and Agri-Food Canada (AAFC) and Red Hat are collaborating on an on-premises prood of concept.  The goal isn't necessarily to prove the viability or value of OpenShift (that was done in the winter with a different proof of concept), but to prove the viability of building DevOps pipelines and tooling on OpenShift in order to work towards moderrnizing appliaction development.

* Increare quality with Continuous Integration (CI), including:
    * CI Pipeline tooling and examples.
    * Maven proxy and artifact repository.
    * Source Control with a tool that allows for functionality such as Pull Requests.
    * Dependency vulnerability scanning (scanning of included external libraries).
    * Code Quality (code test coverage, best practices, FindBugs, etc...)
    * Automated Functional Testing (automated browser-based testing, if time permits)

All tools implemented in this proof of concept are open source, although some tools have paid versions that include added functionality.  This proof of concept utilizes only open source CI/CD and DevOps tooling.

The pipeline described here is based on this example [DevSecOps pipeline](https://github.com/pittar/ocp-devops-setup/blob/master/full-install.md).

## Tools

### Source Control: Gogs

[Gogs](https://gogs.io/) is an open source Git repository management system.  Gogs was chosen for this proof of concept because there are OpenShift Templates readily available to run it.  Any other moderrn Git repositorry manager such as GitLab or BitBucket could be swapped in without any issue.

The template for deploying a persistent Gogs instance [can be found here](https://github.com/OpenShiftDemos/gogs-openshift-docker/blob/master/openshift/gogs-persistent-template.yaml).

### CI Pipeline: Jenkins

For this proof of conept the standard Jenkins master is utilized that is provided out of the box with OpenShift.  This is augmented by including a list of plugins to enable Dependency Track functionality.  The instructions for how to [start Jenkins with plugins can be found here](https://github.com/pittar/ocp-devops-setup/blob/master/full-install.md#start-jenkins-persistent).

On top of this, we built a custom Jenkins Agent that can run Gradle builds with the version of Gradle that AAFC is currently using.  The Dockerfile to build this Jenkins Agent [can be found here](https://github.com/pittar/containers-quickstarts/blob/master/jenkins-slaves/jenkins-slave-gradle/Dockerfile).

### Maven Proxy: Sonatype Nexus

Nexus is a an extremely popular Maven artifact repository and proxy.  Nexus is used in this proof of concept to proxy and cache Java dependencies from external repositories such as Maven Central.  It is also used to proxy internal dependencies that are required to build AAFC applications.

The template and instructions to deploy Nexus to OpenShift [can be found here](https://github.com/pittar/ocp-devops-setup/blob/master/full-install.md#start-sonatype-nexus-2).

### Dependency Vulnerability Scanning: Dependency-Track

[Dependency-Track](https://dependencytrack.org/) is a stand-alone application that downloads and stores up-to-date [OWASP](https://www.owasp.org) vulnerability data that is used to cross-reference the list of dependencies your application currently uses.  Dependency-Track then creates a report outlining any dependencies that may be affectec by known vunlerabilies as well as their serverity.

Installation instrutions and templates [can be found here](https://github.com/pittar/ocp-devops-setup/blob/master/full-install.md#start-dependency-track).

### Code Quality: SonarQube

[SonarQube](https://www.sonarqube.org/) is another very popular open source tool that is included in most modern CI/CD pipelines.  SonarQube is used to guage code quality and security.  Combined with a plugins such as [JaCoCo](https://github.com/jacoco/jacoco), SonarQube can report on common coding errors, code duplication, potential security issues (e.g. SQL injetion potential) and code test coverage.

The instrutions for how to install SonarQube and the templates [can be found here](https://github.com/pittar/ocp-devops-setup/blob/master/full-install.md#start-sonarqube).


## Pipeline Process

For this proof of concept, one AAFC Java application was chosen to be built and deployed using OpenShift and the DevOps tooling deployed on top of it.

In this scenario two types of OpenShift builds involved:  **Source-to-Image** and **Jenkins Pipeline**.

### Source-to-Image (S2I) Build

The [Source-to-Image](https://docs.openshift.com/container-platform/3.11/using_images/s2i_images/index.html) build is based on a standard (and supported) OpenJDK 1.8 image based on Red Hat Eneterprise Linux (RHEL) 7.  This image is part of the base catalog of images available to teams who have OpenShift subscriptions.  Using an official image based on RHEL and Red Hat OpenJDK is important because it gives you the ability to create support tickets not only for OpenShift issues, but also any issues with the base container image (RHEL) or Java itself (OpenJDK 1.8). S2I images are available for multiple languages, including PHP, Python, Javascript, and more.

Red Hat base images are also frequently updated and patched, ensuring that you always have secure and high-quality base images to work with.

S2I can be used to build an application directly from source (e.g. a Git repository), however, since we are using a Jenkins pipeline to control multiple stages of a build, we will simply use the "binary build" capability to create a new image based on the OpenJDK base image and the artifact (exectuable jar) produced as part of the pipeline build.

### Jenkins Pipeline Build

Jenkins Pipeline builds are defined by a [Jenkinsfile file](https://docs.openshift.com/container-platform/3.11/dev_guide/openshift_pipeline.html) that is usually kept in the root of the source repository.  Jenkinsfiles are written in Groovy and outline all of the steps Jenkins needs to take during the build and deploy process.

This is extremely powerful, since a single file that is kept in your source repository can define exactly how your application should be built.  This also greatly simplifies Jenkins itself, which needs very little manual setup due to the fact that builds are defined as part of the source.

The pipeline for the proof of concept application does a standard `gradle build`, then invokes the `S2I' build to build the resulting container image.

### Build Flow

The build flow for this application runs like so:
* Developer pushes source code to git (either in AAFC internal git repository, or Gogs hosted on OpenShift).
* Developer triggers a build (or a webhook automatically triggers a build upon each push to master).
* Jenkins clones the source repository and inspects the `Jenkinsfile` in the root of the repository to determine pipeline steps.
* Jenkins creates `DEV` and `UAT` projects (enviornments) if they do not yet exist.
* Jenkins builds and tests the app using Gradle, pulling required dependencies from the hosted Nexus repository.
* Jenkins invokes the *S2I* build and passes the freshly-built jar file to the build process to create a new container image for the app.
* Jenkins invokes SonarQube to run a quality report on the code.
* Jenkins creates a *Bill of Materials (BOM)* file containing the full list of dependencies and their versions that the application uses.
* Jenkins uses the *Dependency-Track* plugin to invoke Dependency-Track to build a vulnerability report on the dependencies in the BOM.
* Jenkins *tags* the app for **DEV**.
* In the **DEV** environment, if the application has never been deployed, then create all required resources by instantiating the app template.
* Otherwise, simply call a rolling update on `<app>:dev`.
* Repeat the last 3 steps for **UAT**.

## Future Improvements and Next Steps

For this proof of concept, AAFC only has a single OpenShift cluster.  Normally, in a fully oprationalized world, there are at least two clusters:  Non-Prod and Prod.

Once a production cluster is in place, the pipeline will have to be extended to include promoting the container image from non-prod to production.  This is normally as simple as pushing the image to the `prod` image registry after the successful completion of the last 'non-prod' step in the pipeline.

Selenium Grid can also be added to the pipeline in order to add Firefox and Chrome nodes that are available to be used for Selenium automated functional tests.  This kind of testing is good for making sure the UI actually works as expected, and to help find regressions earlier.

## Other Suggestions

Althoguh the pipeline in this proof of concept concentrates on building and deploying apps to OpenShift, there is no reason why you can't use the same tooling to build applications that will be deployed to non-OpenShift environments.

It is just as easy to build a Jenkinsfile that will run a gradle build and test, report on CVEs with Dependency-Track and run quality reports with SonarQube withouth producing a container image.  Simply skip the step that builds an image and instead, upload the resulting artifact (jar/war/ear) to Nexus, which can be used as a staging repository for your other operations teams.