# isis-module-publishing #

[![Build Status](https://travis-ci.org/isisaddons/isis-module-publishing.png?branch=master)](https://travis-ci.org/isisaddons/isis-module-publishing)

This module, intended for use with [Apache Isis](http://isis.apache.org), provides an implementation of Isis'
`PublishingService` API that persists published events using Isis' own (JDO) objectstore.  The intention is that these
are polled through some other process (for example using [Apache Camel](http://camel.apache.org)) so that external
systems can be updated.  To support this the persisted events have a simple state (QUEUED or PROCESSED) so that the
updating process can keep track of which events are outstanding.

The module also contains an implementation of the related `EventSerializer` API.  This is responsible for converting
the published events into a string format for persistence; specifically JSON.  The JSON representation used is that
of the [Restful Objects](http://restfulobjects.org) specification and includes the URL of the original publishing
object such that it can be accessed using Isis' Restful Objects viewer if required.

## Screenshots ##

The following screenshots show an example app's usage of the module.

#### Installing the Fixture Data ####

Installing fixture data...

![](https://raw.github.com/isisaddons/isis-module-publishing/master/images/010-install-fixtures.png)

... returns a simplified customer object, with a name, an address, a collection of orders and also a list of 
published events as a contributed collection.  The fixture setup results in one published event already:
 
![](https://raw.github.com/isisaddons/isis-module-publishing/master/images/020-update-address-using-customer-action.png)

#### Updating the Customer's Address (published action) ####

The customer's `updateAddress` action is a _published_ action:
 
    @PublishedAction(value = PublishedCustomer.UpdateAddressEventPayloadFactory.class)
    public PublishedCustomer updateAddress(
            final @Named("Line 1") String line1,
            final @Named("Line 2") @Optional String line2,
            final @Named("Town") String town) {
        ReferencedAddress address = getAddress();
        if(address == null) {
            address = container.newTransientInstance(ReferencedAddress.class);
        }
        address.setLine1(line1);
        address.setLine2(line2);
        address.setTown(town);
        setAddress(address);
        container.persistIfNotAlready(address);
        return this;
    }

![](https://raw.github.com/isisaddons/isis-module-publishing/master/images/030-update-address-prompt.png)

When the action is invoked, a `PublishedEvent` is created by the framework, and persisted by the module:

![](https://raw.github.com/isisaddons/isis-module-publishing/master/images/040-action-invocation-event-created.png)

The `PublishedEvent` holds the following details:

![](https://raw.github.com/isisaddons/isis-module-publishing/master/images/050-action-invocation-details.png)

The `serialized form` field holds a JSON representation of the `PublishedCustomer.UpdateAddressEventPayloadFactory`
class specified in the `@PublishedAction` annotation.

In addition, the `ReferencedAddress` entity is also a published object:

    @PublishedObject // using the default payload factory
    public class ReferencedAddress ... { ... }

This means that as well as raising and persisting the action invocation event, a separate event is raised and persisted
 for the change to the `ReferencedAddress`, the JSON representation of which includes a URL back to the changed address object: 
 
![](https://raw.github.com/isisaddons/isis-module-publishing/master/images/060-address-object-changed-event.png)

Note that both the published events (for the action invocation on customer and the change of address) are associated by
the same transaction Id (a GUID).  This GUID can also be used to associate the event back to any persisted commands (as
per the [Isis Addons Command](http://github.com/isisaddons/isis-module-command) module and to any audit entries (as per
 the [Isis Addons Audit](http://github.com/isisaddons/isis-module-audit) module).

#### Updating the Customer's Name (published changed object) ####

Changes to the customer are also published:

    @PublishedObject(value = PublishedCustomer.ObjectChangedEventPayloadFactory.class)
    public class PublishedCustomer ... { ... }

In this case a custom payload factory is specified:

    public static class ObjectChangedEventPayloadFactory implements PublishedObject.PayloadFactory {
        public static class PublishedCustomerPayload extends EventPayloadForObjectChanged<PublishedCustomer> {

            public PublishedCustomerPayload(PublishedCustomer changed) { super(changed); }

            public String getAddressTown() {
                final ReferencedAddress address = getChanged().getAddress();
                return address != null? address.getTown(): null;
            }
            public String getCustomerName() {
                return getChanged().getName();
            }
            public SortedSet<ReferencedOrder> getOrders() {
                return getChanged().getOrders();
            }
        }
        @Override
        public EventPayload payloadFor(Object changedObject, PublishedObject.ChangeKind changeKind) {
            return new PublishedCustomerPayload((PublishedCustomer) changedObject);
        }
    }

The custom payload exposes additional related information: the customer's name (a simple scalar), the customer's 
address' town (traversing a reference), and the orders of the customers (traversing a collection).  


So, changing the customer's name:

![](https://raw.github.com/isisaddons/isis-module-publishing/master/images/070-change-customer-object.png)

... causes an event to persisted:

![](https://raw.github.com/isisaddons/isis-module-publishing/master/images/080-customer-changed-object-event-created.png)

The additional information of the custom payload is captured in the serialized form of the event; in the screenshot
below, see 'addressTown' property:

![](https://raw.github.com/isisaddons/isis-module-publishing/master/images/090-customer-changed-event-details.png)


## Relationship to Apache Isis Core ##

Isis Core 1.6.0 included the `org.apache.isis.core:isis-module-publishing:1.6.0` and also 
`org.apache.isis.core:isis-module-publishingeventserializer:1.6.0` Maven artifacts.  This module is a direct copy of 
the code of those two modules, with the following changes:

* package names have been altered from `org.apache.isis` to `org.isisaddons.module.publishing`
* the `persistent-unit` (in the JDO manifest) has changed from `isis-module-publishing` to 
  `org-isisaddons-module-publishing-dom`

Otherwise the functionality is identical; warts and all!

At the time of writing the plan is to remove this module from Isis Core (so it won't be in Isis 1.7.0), and instead 
continue to develop it solely as one of the [Isis Addons](http://www.isisaddons.org) modules.


## How to configure ##

You can either use this module "out-of-the-box", or you can fork this repo and extend to your own requirements. 

To use "out-of-the-box":

* update your classpath by adding this dependency in your webapp project's `pom.xml`:

<pre>
    &lt;dependency&gt;
        &lt;groupId&gt;org.isisaddons.module.publishing&lt;/groupId&gt;
        &lt;artifactId&gt;isis-module-publishing-dom&lt;/artifactId&gt;
        &lt;version&gt;1.6.0&lt;/version&gt;
    &lt;/dependency&gt;
</pre>

* assuming you are using the provided `RestfulObjectsSpecEventSerializer` (that is, haven't written your own 
  implementation of the `EventSerializer` API), then also update your classpath to add this dependency in your 
  webapp project's `pom.xml`:

<pre>
    &lt;dependency&gt;
        &lt;groupId&gt;org.apache.isis.core&lt;/groupId&gt;
        &lt;artifactId&gt;isis-core-viewer-restfulobjects-rendering&lt;/artifactId&gt;
        &lt;version&gt;1.6.0&lt;/version&gt;
    &lt;/dependency&gt;
</pre>

  Check for later releases by searching [Maven Central Repo](http://search.maven.org/#search%7Cga%7C1%7Cisis-module-publishing-dom).
  
* update your `WEB-INF/isis.properties`:

<pre>
    isis.services-installer=configuration-and-annotation
    isis.services.ServicesInstallerFromAnnotation.packagePrefix=
                    ...,\
                    org.isisaddons.module.publishing,\
                    ...

    isis.services = ...,\
                    org.isisaddons.module.publishing.dom.eventserializer.RestfulObjectsSpecEventSerializer,\
                    org.isisaddons.module.publishing.dom.PublishingServiceContributions,\
                    ...
</pre>
                    
The `RestfulObjectsSpecEventSerializer` (or some other implementation of `EventSerializer`) must be registered.  The  
`PublishingServiceContributions` service is optional but recommended; see below for more information.

If instead you want to extend this module's functionality, then we recommend that you fork this repo.  The repo is 
structured as follows:

* `pom.xml   ` - parent pom
* `dom       ` - the module implementation, depends on Isis applib
* `fixture   ` - fixtures, holding a sample domain objects and fixture scripts; depends on `dom`
* `integtests` - integration tests for the module; depends on `fixture`
* `webapp    ` - demo webapp (see above screenshots); depends on `dom` and `fixture`


#### Setting the Base URL for the Restful Objects Event Serializer ####

The `RestfulObjectsSpecEventSerializer` serializes event payloads into a JSON string that contains URLs such that an
external (subscribing) system can then access the publishing and related objects (using those URLs) by way of Isis'
Restful Objects viewer.

For this to work correctly, the base url must be specified in `isis.properties`, for example:

    isis.viewer.restfulobjects.RestfulObjectsSpecEventSerializer.baseUrl=http://isisapp.mycompany.com:8080/restful/

The default value if not specified is in fact `http://localhost:8080/restful/` (for development/testing purposes only).

## API & Implementation ##

### PublishingService ###

The `PublishingService` API (in the Isis applib) is defined as:

    public interface PublishingService {
        public void publish(EventMetadata metadata, EventPayload payload);
        void setEventSerializer(EventSerializer eventSerializer);
    }

The `EventMetadata` is a concrete class with the fields:

    public class EventMetadata {
        
        private final UUID transactionId;
        private final int sequence;
        private final String user;
        private final java.sql.Timestamp javaSqlTimestamp;
        private final String title;
        private final EventType eventType;
        private final String targetClass;
        private final String targetAction;
        private final Bookmark target;
        private final String actionIdentifier;
        
        ...
    }

where:

* `transactionId` is a unique identifier (a GUID) of the transaction in which this event was raised.
* `sequence` discriminates multiple events raised in a single transaction
* `user` is the name of the user that invoked the action or otherwise caused the event to be raised
* `timestamp` is the timestamp for the transaction
* `title` is the title of the publishing object
* `eventType` is an enum distinguishing action invocation or (for changed objects) object created/update/deletedACTION_INVOCATION,
* `targetClass` holds the class of the publishing event, eg com.mycompany.myapp.Customer
* `targetAction` is a user-friendly name for the action (applicable only for action invocation events)
* `target` is a Bookmark of the target object, in other words a provides a mechanism to look up the publishing object, 
  eg CUS:L_1234 to identify customer with id 1234. ("CUS" corresponds to the @ObjectType annotation/facet).
* `actionIdentifier` is the formal/canonical action name  (applicable only for action invocation events)

This module provides `org.isisaddons.module.publishing.dom.PublishingService` as an implementation of the API.  The
service is annotated with `@DomainService` so there is no requirement to register in `isis.properties`.

### EventSerializer ###

The `EventSerializer` API (in Isis applib) is defined as:

    public interface EventSerializer {
        public Object serialize(EventMetadata metadata, EventPayload payload);    
    }

This module provides `org.isisaddons.module.publishing.dom.eventserializer.RestfulObjectsSpecEventSerializer` as an
implementation of this API.  Note that this service must be explicitly registered in `isis.properties` _and_ its 
Maven dependency on `o.a.i.core:isis-core-viewer-restfulobjects-rendering` must be added to the classpath (see above
for details).  This has been done to allow alternative implementations of the `EventSerializer` API to be configured
instead if required.


## Supporting Services ##

As well as the `PublishingService` and `EventSerializer` implementations, the module also provides a number of other
domain services:

* `PublishingServiceRepository` provides the ability to search for persisted (`PublishedEvent`) events.  None of its
  actions are visible in the user interface (they are all @Programmatic) and so this service is automatically 
  registered.
  
* `PublishingServiceContributions` provides the `publishedEvents` contributed collection to the `HasTransactionId` 
  interface. This will therefore display all published events that occurred in a given transaction, in other words 
  whenever a command, an audit entry or another published event is displayed.

## Related Modules/Services ##

The Isis applib defines a number of closely related APIs, `PublishingService` being one of them.  Implementations of 
these various services can be found referenced by the [Isis Add-ons](http://isisaddons.org) website.

The `CommandContext` defines the `Command` class which provides request-scoped information about an action invocation. 
Commands can be thought of as being the cause of an action; they are created "before the fact". 

The `CommandService` service is an optional service that acts as a `Command` factory and allows `Command`s to be 
persisted. `CommandService`'s API introduces the concept of a transactionId; this is the same value as is passed to the 
`PublishingService`'s in the `EventMetadata` object.

The `AuditingService3` service enables audit entries to be persisted for any change to any object. The command can be 
thought of as the "cause" of a change, the audit entries as the "effect".

If all these services are configured - such that commands, audit entries and published events are all persisted, then 
the transactionId that is common to all enables seamless navigation between each. (This is implemented through 
contributed actions/properties/collections; `PublishedEvent` implements the `HasTransactionId` interface in Isis' 
applib, and it is this interface that each module has services that contribute to).


## Legal Stuff ##
 
#### License ####

    Copyright 2014 Dan Haywood

    Licensed under the Apache License, Version 2.0 (the
    "License"); you may not use this file except in compliance
    with the License.  You may obtain a copy of the License at

        http://www.apache.org/licenses/LICENSE-2.0

    Unless required by applicable law or agreed to in writing,
    software distributed under the License is distributed on an
    "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
    KIND, either express or implied.  See the License for the
    specific language governing permissions and limitations
    under the License.


#### Dependencies ####

There are no third-party dependencies.

##  Maven deploy notes ##

Only the `dom` module is deployed, and is done so using Sonatype's OSS support (see 
[user guide](http://central.sonatype.org/pages/apache-maven.html)).

#### Release to Sonatype's Snapshot Repo ####

To deploy a snapshot, use:

    pushd dom
    mvn clean deploy
    popd

The artifacts should be available in Sonatype's 
[Snapshot Repo](https://oss.sonatype.org/content/repositories/snapshots).

#### Release to Maven Central (scripted process) ####

The `release.sh` script automates the release process.  It performs the following:

* perform sanity check (`mvn clean install -o`) that everything builds ok
* bump the `pom.xml` to a specified release version, and tag
* perform a double check (`mvn clean install -o`) that everything still builds ok
* release the code using `mvn clean deploy`
* bump the `pom.xml` to a specified release version

For example:

    sh release.sh 1.6.0 \
                  1.6.1-SNAPSHOT \
                  dan@haywood-associates.co.uk \
                  "this is not really my passphrase"
    
where
* `$1` is the release version
* `$2` is the snapshot version
* `$3` is the email of the secret key (`~/.gnupg/secring.gpg`) to use for signing
* `$4` is the corresponding passphrase for that secret key.

If the script completes successfully, then push changes:

    git push
    
If the script fails to complete, then identify the cause, perform a `git reset --hard` to start over and fix the issue
before trying again.

#### Release to Maven Central (manual process) ####

If you don't want to use `release.sh`, then the steps can be performed manually.

To start, call `bumpver.sh` to bump up to the release version, eg:

     `sh bumpver.sh 1.6.0`

which:
* edit the parent `pom.xml`, to change `${isis-module-command.version}` to version
* edit the `dom` module's pom.xml version
* commit the changes
* if a SNAPSHOT, then tag

Next, do a quick sanity check:

    mvn clean install -o
    
All being well, then release from the `dom` module:

    pushd dom
    mvn clean deploy -P release \
        -Dpgp.secretkey=keyring:id=dan@haywood-associates.co.uk \
        -Dpgp.passphrase="literal:this is not really my passphrase"
    popd

where (for example):
* "dan@haywood-associates.co.uk" is the email of the secret key (`~/.gnupg/secring.gpg`) to use for signing
* the pass phrase is as specified as a literal

Other ways of specifying the key and passphrase are available, see the `pgp-maven-plugin`'s 
[documentation](http://kohsuke.org/pgp-maven-plugin/secretkey.html)).

If (in the `dom`'s `pom.xml`) the `nexus-staging-maven-plugin` has the `autoReleaseAfterClose` setting set to `true`,
then the above command will automatically stage, close and the release the repo.  Sync'ing to Maven Central should 
happen automatically.  According to Sonatype's guide, it takes about 10 minutes to sync, but up to 2 hours to update 
[search](http://search.maven.org).

If instead the `autoReleaseAfterClose` setting is set to `false`, then the repo will require manually closing and 
releasing either by logging onto the [Sonatype's OSS staging repo](https://oss.sonatype.org) or alternatively by 
releasing from the command line using `mvn nexus-staging:release`.

Finally, don't forget to update the release to next snapshot, eg:

    sh bumpver.sh 1.6.1-SNAPSHOT

and then push changes.
