# Red Hat JBoss BPM Suite & Entando Client Onboarding FSI Demo

A _Client Onboarding_ FSI Demo, developed in Red Hat JBoss BPM Suite and Entando, running on OpenShift.

*TODO*: Insert link to introduction video on Youtube.

## Use case

*TODO*: Describe the use-case.


## Running the demo

This demo runs both on local installations of OpenShift Origin and OpenShift Container Platform,
as well as OpenShift Enterprise, OpenShift Dedicated and OpenShift Online instances.

A provisioning script has been provided which fully automates the installation of this demo. The only prerequisit is a local `oc` client. The
demo is installed in the OpenShift environment to which the `oc` client is connected.

If you don't have a local installation of OpenShift on your machine, but do have a VirtualBox virtualization environment installed, you can also
build a demo VirtualMachine using the provided https://www.vagrantup.com[Vagrant] and https://www.ansible.com[Ansible scripts].


## Running on OpenShift

This section describes how to run this demo on your OpenShift environment. If you don't have an OpenShift environment (either locally or online)
available, please scroll down to the *Building the Virtual Machine* section.

### Prerequisites

* Install https://www.openshift.org/download.html[the oc client tools].

* Login into the OpenShift in which the demo needs to be provisioned. Depending on the installation you can login with:
  - `oc login <url>`: The `oc` client will ask for a username and password.
  - `oc login <url> <toke>`: The provided token is used for authentication.

NOTE: To copy-paste the login command and token, go to your OpenShift web console and look for _Help_ > _Command line tools_.

### Install demo

Our demo uses the JBoss BPM Suite Intelligent Process Server as the process execution runtime. To be able to use this runtime, we need to configure our OpenShift environment to support the JBoss ImageStreams and Templates required by this runtime:

* First, login to OpenShift as the system admin: `oc login -u system:admin`

* Run the following command to install the Process Server ImageStreams:
```
oc create -f https://raw.githubusercontent.com/jboss-openshift/application-templates/master/processserver/processserver64-image-stream.json -n openshift
```
* Run the following commands to install the Process Server templates:
```
oc create -f https://raw.githubusercontent.com/jboss-openshift/application-templates/master/processserver/processserver64-postgresql-s2i.json -n openshift
oc create -f https://raw.githubusercontent.com/jboss-openshift/application-templates/master/processserver/processserver64-postgresql-persistent-s2i.json -n openshift
```

Now that we've configured ImageStreams and Templates we can provision our demo. Use the provided provision scripts to setup, configure, build and deploy the demo on OpenShift:

* First, login to OpenShift with your user account (e.g. developer/developer): `oc login`
* Next, run the provided `provision.sh` script to provision the demo:

```
./openshift/provision.sh setup client-onboarding
```

### Delete the OpenShift application and project

```
./openshift/provision.sh delete client-onboarding
```

### Run the demo

To run the demo just open your web browser and point to [https://localhost:8443/console/](https://localhost:8443/). Login with `developer/developer` credentials and click on `Client Onboarding` project.

Now on the `Overview` page you'll see three applicantions:

__fsi-backoffice__

In the _fsi-backoffice_ application you can access with the following users:

* `account/adminadmin` this user starts the process
* `legal/adminadmin` this user reviews legal documents
* `knowledge/adminadmin` this user reviews non legal documents
* `Manager/adminadmin` this user reviews all the processes


[http://fsi-backoffice-client-onboarding-developer.127.0.0.1.xip.io/fsi-backoffice/](http://fsi-backoffice-client-onboarding-developer.127.0.0.1.xip.io/fsi-backoffice/)


__fsi-customer__

*TODO* Add a description and the credentials to use

[http://fsi-customer-client-onboarding-developer.127.0.0.1.xip.io/fsi-customer/](http://fsi-customer-client-onboarding-developer.127.0.0.1.xip.io/fsi-customer/)

__processserver64-postgresql-s2i__

*TODO* Add a description

## Building the Virtual Machine

If you don't have an OpenShift environment available, you can also build a Virtual Machine with the full demo-environment (including OpenShift)
provisioned

### Prerequisites

* An installation of https://www.virtualbox.org[VirtualBox]
* A https://www.vagrantup.com[Vagrant] installation.
* The Vagrant VBGuest plugin needs to be installed. If you don't have this plugin installed, you can install it with the command: `vagrant plugin install vagrant-vbguest`
* An https://www.ansible.com[Ansible] installation.

### Building the VM

We've provided a [Vagrantfile](vagrant/Vagrantfile) which fully automates the installation of the VirtualMachine. Run the following command in
the directory in which you checked out this Git repository:
```
./vagrant/vagrant up
```

Vagrant uses the provided Ansible playbook [entando-client-onboarding-playbook.yml](vagrant/entando-client-onboarding-playbook.yml) to provision
the VM and install:
* Gnome
* Docker
* OpenShift
* Client Onboarding Demo


## BPM Suite Demo Key Capabilities
This demo, apart from demonstrating an end-to-end Client Onboarding _process driven application_, also demonstrates a number of
key capabilites of the Red Hat BPM Suite platform. We will discuss these capabilities in more detail in this paragraph

### Signaling and data passing
After the process is started, and an initial e-mail is sent in the _SendApplicationLink_ subprocess, the process needs to wait for the
user to have clicked the link in the mail and has signed up on the Entando page (which essesntially creates an account) before it can continue.
To accomplish this, the main process contains a subprocess that waits at a "Catching Intermediate Signal Event", until the Entando Platform
signals the process engine to continue (which happens when the user creates his account after he/she has clicked the link in the e-mail).

With this signal, the Entando platform also passes the username of the user has created in the system. This allows the process instance to store
that username as a process variable and later use it to (dynamically) assign tasks to this user account (see below).

### Timer-based Escalation and Notification
For the business process to be successful, we need to make sure that the customers that are signed up by the account manager indeed click the link in the e-mail
they recieve and register themselve in the system. As explained above, the Entando sends a signal to the process when this event has occurred. However, we want to make sure to follow up with the customer if he/she does not click the link in the e-mail and does not register with the system and process.

To accomplish this, the sub-process in which we wait for the incoming signal has a Timer Boundary Event attached to it. In this demo the timer is configured to fire every 60 seconds. Also note that we've configured to NOT cancel the subprocess when it fires (CancelActivity=false), which means that although the timer fires, the sub-process instance stays active. When the timer fires, the process goes to the _Signal Account Manager_ activity, which signals the account manager that the user has not yet activated his/her account in the system.

### Data-Driven Task Assignment
As stated above, the Client Onboarding process contains Human Tasks (e.g. upload document) that are assigned to the new customer that is being onboarded.
This means that we need to assign tasks to a user account that is created after the process has been started. This is accomplished by Entando sending the
username of the new account when singnaling the process to continue. This username is stored as a process variable (_accountName_) which is later used to
assign tasks to that username using the following syntax ```#{accountName}```. This is for example done in the _Enrichment Upload Document_ task in the _EnrichmentProcess_.

### Store Your Process Variables Everywhere
By default jBPM/BPM Suite process instances variables are stored as a BLOB in the database. This can be sufficient for a lot of use-case, but there are also
use-cases in which the data needs to be stored in a relation table outside of the BLOB. This for example allows to share data between the process instances and
other parts of the system (e.g. the user-interface) and to easier search on data created in and used by the process.

More information on this topic can be found here:

In this demo, the Client data (_com.redhat.bpms.demo.fsi.onboarding.model.Client_) is stored in its own table in the database. This is accomplished by annotating
the domain model with JPA (Java Persistence API) annotations (see: https://github.com/entando/fsi-onboarding-bpm/blob/master/commercial-client-onboarding/src/main/java/com/redhat/bpms/demo/fsi/onboarding/model/Client.java)).

Second, we've configured the jBPM JPAPlaceholderResolverStrategy as a custom marshalling strategy on our KJAR (as can be seen [here](https://github.com/entando/fsi-onboarding-bpm/blob/master/commercial-client-onboarding/src/main/resources/META-INF/kie-deployment-descriptor.xml#L9)), which ensures that all JPA-annotated process variables are marshalled using JPA into the table configured on the model, rather than stored in a BLOB. In the case of our _Client_ process variable, this means that the process variables is stored in the _client_ table in the database.

### Rule-Driven Sub-Process and Task Creation
One of the powers of Red Hat JBoss BPM Suite is the integration between processes and rules. This allows for the creation of intelligent, rules driven, decisions within processes. An example of this is provided in the _EnrichmentProcess_ subprocess of this demo.

The subprocess contains a _Business Rule_ node/activity called ```Determine Required Documents``` which drives a [decision-table](https://github.com/entando/fsi-onboarding-bpm/blob/master/commercial-client-onboarding/src/main/resources/com/redhat/bpms/demo/fsi/onboarding/enrichment-required-documents.gdst). This decision-table determines, based on the type of business, which documents are required, and adds the name of the document to the ```RequiredDocuments``` collection. For each entry in this collection, the multi-instance sub-process creates a new subprocess instance in which a HumanTask is created for the end-user to upload the given document. This subprocess also contains the logic to auto-validate the document, and in case auto-validation is unsuccessful, create an ```Enrichment Manual Approval``` task for the bank's legal worker to validate the document.

This construct demonstrates how business rules (in this case a decision-table) can drive the creation of sub-processes and human tasks. More background information about multi-instance subprocesses can be found [here](http://mswiderski.blogspot.nl/2015/01/multiinstance-characteristic-example.html).

### Document Management
The client onboarding process requires the user to upload various types of documents (e.g. identification, credit, etc.), and the bank's automated systems, and legal and knowledge workers to validate these documents. Ideally we don't want our Documents to be stored as process variables in a BLOB, but would like to connect our process engine to a dedicated Content Management System (CMS) or Document Management System (CMS). BPM Suite/jBPM provides this integration through its ```DocumentMarshallingStrategy```, a marshalling strategy that marshals all process variables of type ```org.jbpm.document.Document``` and ```org.jbpm.document.Documents``` (which represents a collection of documents) via its ```DocumentStorageService```. This marshalling strategy is configured on the KJAR's deployment descriptor as shown [here](https://github.com/entando/fsi-onboarding-bpm/blob/master/commercial-client-onboarding/src/main/resources/META-INF/kie-deployment-descriptor.xml#L14).

Out of the box, the marshalling strategy persists the documents as files on disk, but, as with most services in BPM Suite/jBPM, other strategies can be plugged in to connect to any CMS/DMS.

More information about document management in jBPM can be found [here](http://www.schabell.org/2014/07/lightning-strike-brings-redhat-jboss-bpmsuite-ecm-cmis-demo.html) and [here](http://mswiderski.blogspot.nl/2016/08/kie-server-jbpm-extension-brings.html).
