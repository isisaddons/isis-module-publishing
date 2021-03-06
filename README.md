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


## Domain Model ##

A full description of Isis' Publishing Service API can be found on 
[this page](http://isis.apache.org/reference/services/publishing-service.html) of the [Isis website](http://isis.apache.org).
However, the diagram below illustrates the main concepts:

![](https://raw.github.com/isisaddons/isis-module-publishing/master/images/class-diagram.png)

The `PublishedObject` and `PublishedAction` annotations specify the event payload factory to use to create a 
representation of the event; if no factory is specified then a default factory is used.

The Isis runtime passes the event to the `PublishingService` so that it can be published.  The `PublishingService` 
in turn delegates to the `EventSerializer` to create a representation of the provided event, along with various metadata
about the event.

The diagram was generated by [yuml.me](http://yuml.me); see appendix at end of page for the DSL.


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
 
    @Action(
            publishing = Publishing.ENABLED,
            publishingPayloadFactory = PublishedCustomer.UpdateAddressEventPayloadFactory.class
    )
    public PublishedCustomer updateAddress(
            @ParameterLayout(named="Line 1") final String line1,
            @ParameterLayout(named="Line 2") @Parameter(optionality = Optionality.OPTIONAL) final String line2,
            @ParameterLayout(named="Town") final String town) {
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

    @DomainObject(
            publishing = Publishing.ENABLED // using the default payload factory
    )
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

    @DomainObject(
            publishing = Publishing.ENABLED,
            publishingPayloadFactory = PublishedCustomer.ObjectChangedEventPayloadFactory.class
    )
    public class PublishedCustomer ... { ... }

In this case a custom payload factory is specified:

    public static class ObjectChangedEventPayloadFactory implements PublishingPayloadFactoryForObject {
        @Override
        public EventPayload payloadFor(final Object changedObject, final PublishingChangeKind publishingChangeKind) {
            return new PublishedCustomerPayload((PublishedCustomer) changedObject);
        }
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


## How to run the Demo App ##

The prerequisite software is:

* Java JDK 8 (>= 1.9.0) or Java JDK 7 (<= 1.8.0)
** note that the compile source and target remains at JDK 7
* [maven 3](http://maven.apache.org) (3.2.x is recommended).

To build the demo app:

    git clone https://github.com/isisaddons/isis-module-publishing.git
    mvn clean install

To run the demo app:

    mvn antrun:run -P self-host
    
Then log on using user: `sven`, password: `pass`


## Relationship to Apache Isis Core ##

Isis Core 1.6.0 included the `org.apache.isis.module:isis-module-publishing-jdo:1.6.0` and also `org.apache.isis.module:isis-module-publishingeventserializer:1.6.0` Maven artifacts.  This module is a direct copy of the code of those two modules, with the following changes:

* package names have been altered from `org.apache.isis` to `org.isisaddons.module.publishing`
* the `persistent-unit` (in the JDO manifest) has changed from `isis-module-publishing` to 
  `org-isisaddons-module-publishing-dom`

Otherwise the functionality is identical; warts and all!

Isis 1.7.0 no longer ships with `org.apache.isis.module:isis-module-publishing-jdo` or `org.apache.isis.module:isis-module-publishingeventserializer-ro`; instead use this addon module.


## How to configure/use ##

You can either use this module "out-of-the-box", or you can fork this repo and extend to your own requirements. 

#### "Out-of-the-box" ####

To use "out-of-the-box":

* update your classpath by adding this dependency in your webapp project's `pom.xml`:

<pre>
    &lt;dependency&gt;
        &lt;groupId&gt;org.isisaddons.module.publishing&lt;/groupId&gt;
        &lt;artifactId&gt;isis-module-publishing-dom&lt;/artifactId&gt;
        &lt;version&gt;1.13.0&lt;/version&gt;
    &lt;/dependency&gt;
</pre>

* assuming you are using the provided `RestfulObjectsSpecEventSerializer` (that is, haven't written your own 
  implementation of the `EventSerializer` API), then also update your classpath to add this dependency in your 
  webapp project's `pom.xml`:

<pre>
    &lt;dependency&gt;
        &lt;groupId&gt;org.apache.isis.core&lt;/groupId&gt;
        &lt;artifactId&gt;isis-core-viewer-restfulobjects-rendering&lt;/artifactId&gt;
        &lt;version&gt;1.13.0&lt;/version&gt;
    &lt;/dependency&gt;
</pre>

  Check for later releases by searching [Maven Central Repo](http://search.maven.org/#search%7Cga%7C1%7Cisis-module-publishing-dom).
  
* if using `AppManifest`, then update its `getModules()` method:

    @Override
    public List<Class<?>> getModules() {
        return Arrays.asList(
                ...
                org.isisaddons.module.publishing.PublishingModule.class,
                ...
        );
    }

* otherwise, update your `WEB-INF/isis.properties`:

<pre>
    isis.services-installer=configuration-and-annotation
    isis.services.ServicesInstallerFromAnnotation.packagePrefix=
            ...,\
            org.isisaddons.module.publishing,\
            ...
</pre>


#### "Out-of-the-box" (-SNAPSHOT) ####

If you want to use the current `-SNAPSHOT`, then the steps are the same as above, except:

* when updating the classpath, specify the appropriate -SNAPSHOT version:

    <version>1.14.0-SNAPSHOT</version>

* add the repository definition to pick up the most recent snapshot (we use the Cloudbees continuous integration service).  We suggest defining the repository in a `<profile>`:

<pre>
    &lt;profile&gt;
        &lt;id&gt;cloudbees-snapshots&lt;/id&gt;
        &lt;activation&gt;
            &lt;activeByDefault&gt;true&lt;/activeByDefault&gt;
        &lt;/activation&gt;
        &lt;repositories&gt;
            &lt;repository&gt;
                &lt;id&gt;snapshots-repo&lt;/id&gt;
                &lt;url&gt;http://repository-estatio.forge.cloudbees.com/snapshot/&lt;/url&gt;
                &lt;releases&gt;
                    &lt;enabled&gt;false&lt;/enabled&gt;
                &lt;/releases&gt;
                &lt;snapshots&gt;
                    &lt;enabled&gt;true&lt;/enabled&gt;
                &lt;/snapshots&gt;
            &lt;/repository&gt;
        &lt;/repositories&gt;
    &lt;/profile&gt;
</pre>


#### Forking the repo ####

If instead you want to extend this module's functionality, then we recommend that you fork this repo.  The repo is 
structured as follows:

* `pom.xml   ` - parent pom
* `dom       ` - the module implementation, depends on Isis applib
* `fixture   ` - fixtures, holding a sample domain objects and fixture scripts; depends on `dom`
* `integtests` - integration tests for the module; depends on `fixture`
* `webapp    ` - demo webapp (see above screenshots); depends on `dom` and `fixture`


#### PublishedEvent serialized form ####

The `PublishedEvent` entity can either persist the serialized form of the event as a zipped byte array or as a CLOB.  Which is used is determined by a configuration setting in `isis.properties`: 

    # whether to persist the event data as a "clob" or as a "zipped" byte[]
    isis.persistor.datanucleus.PublishingService.serializedForm=clob

If not specified, then "zipped" is the default.


#### Setting the Base URL for the Restful Objects Event Serializer ####

The `RestfulObjectsSpecEventSerializer` serializes event payloads into a JSON string that contains URLs such that an
external (subscribing) system can then access the publishing and related objects (using those URLs) by way of Isis'
Restful Objects viewer.

For this to work correctly, the base url must be specified in `isis.properties`, for example:

    isis.viewer.restfulobjects.RestfulObjectsSpecEventSerializer.baseUrl=\
                                             http://isisapp.mycompany.com:8080/restful/

The default value if not specified is in fact `http://localhost:8080/restful/` (for development/testing purposes only).


## API & Implementation ##

### PublishingService ###

The `PublishingService` API (in Isis' applib) is defined as:

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
* `eventType` is an enum distinguishing action invocation or (for changed objects) object created/update/deleted,
* `targetClass` holds the class of the publishing event, eg com.mycompany.myapp.Customer
* `targetAction` is a user-friendly name for the action (applicable only for action invocation events)
* `target` is a Bookmark of the target object, in other words a provides a mechanism to look up the publishing object, 
  eg CUS:L_1234 to identify customer with id 1234. ("CUS" corresponds to the @ObjectType annotation/facet).
* `actionIdentifier` is the formal/canonical action name  (applicable only for action invocation events)

This module provides `org.isisaddons.module.publishing.dom.PublishingService` as an implementation of the API.  The
service is annotated with `@DomainService` so there is no requirement to register in `isis.properties`.

### EventSerializer ###

The `EventSerializer` API (in Isis' applib) is defined as:

    public interface EventSerializer {
        public Object serialize(EventMetadata metadata, EventPayload payload);    
    }

This module provides `org.isisaddons.module.publishing.dom.eventserializer.RestfulObjectsSpecEventSerializer` as an
implementation of this API.  Note that this service must be explicitly registered in `isis.properties` _and_ its 
Maven dependency on `o.a.i.core:isis-core-viewer-restfulobjects-rendering` must be added to the classpath (see above
for details).  This has been done to allow alternative implementations of the `EventSerializer` API to be configured
instead if required.


## Supporting Services ##

As well as the `PublishingService` and `EventSerializer` implementations, the module also provides a couple of other
domain services:

* `PublishingServiceRepository` provides the ability to search for persisted (`PublishedEvent`) events.  None of its
  actions are visible in the user interface (they are all `@Programmatic`).
  
* The`PublishingServiceMenu` provides actions to search for `PublishedEvent`s, underneath an 'Activity' menu on the
secondary menu bar.

* `PublishingServiceContributions` provides the `publishedEvents` contributed collection to the `HasTransactionId`
  interface.  This will therefore display all published events that occurred in a given transaction.

* `RestfulObjectsSpecEventSerializer` is an implementation of the `EventSerializer` API that serializes the event into
   a JSON representation based on that of the [http://restfulobjects.org](Restful Objects specification).

In 1.7.0, it is necessary to explicitly register `PublishingServiceContributions` in `isis.properties`, the
rationale being that this service contributes functionality that appears in the user interface.

In 1.8.0 the above policy is reversed: the  `PublishingServiceMenu` and `PublishingServiceContributions`
services are both automatically registered, and both provide functionality that will appear in the user interface.
If this is not required, then either use security permissions or write a vetoing subscriber on the event bus to hide
this functionality, eg:

    @DomainService(nature = NatureOfService.DOMAIN)
    public class HideIsisAddonsPublishingFunctionality {

        @Programmatic @PostConstruct
        public void postConstruct() { eventBusService.register(this); }

        @Programmatic @PreDestroy
        public void preDestroy() { eventBusService.unregister(this); }

        @Programmatic @Subscribe
        public void on(final PublishingModule.ActionDomainEvent<?> event) { event.hide(); }

        @Inject
        private EventBusService eventBusService;
    }

The default `RestfulObjectsSpecEventSerializer` is also automatically registered.  If you want to use some other
implementation of `EventSerializer`, then register in `isis.properties`.

## Related Modules/Services ##

The Isis applib defines a number of closely related APIs.

The `CommandContext` defines the `Command` class which provides request-scoped information about an action invocation. 
Commands can be thought of as being the cause of an action; they are created "before the fact".   The `CommandService` 
service is an optional service that acts as a `Command` factory and allows `Command`s to be persisted. 
`CommandService`'s API introduces the concept of a transactionId; this is the same value as is passed to the 
`PublishingService` in the `EventMetadata` object.

The `AuditingService3` service enables audit entries to be persisted for any change to any object. The command can be 
thought of as the "cause" of a change, the audit entries as the "effect".

If these services and also the `PublishingService` are all configured then the transactionId that is common to all (in
the persisted command, audit entry and published event objects) enables seamless navigation between each.  This is 
implemented through contributed actions/properties/collections; each of the persisted entities implements the 
`HasTransactionId` interface in Isis' applib, and it is this interface that each module has services that contribute to.

Implementations of these various services can be found on the [Isis Add-ons](http://isisaddons.org) website.

Finally, Dan Haywood's [camel-isis-pubsubjdo](https://github.com/danhaywood/camel-isis-pubsubjdo) project up on github shows how to poll and process the persisted `PublishedEvent` table using [Apache Camel](http://camel.apache.org).

## Change Log ##

* `1.13.0` - Released against Isis 1.13.0
* `1.12.0` - released against Isis 1.12.0
* `1.11.0` - released against Isis 1.11.0
* `1.10.0` - Released against Isis 1.10.0
* `1.8.1` - Released against Isis 1.8.0; closes <a href="https://github.com/isisaddons/isis-module-publishing/issues/1">#1</a>.
* `1.8.0` - Released against Isis 1.8.0.  Services are automatically registered; their UI can be suppressed using subscriptions.
* `1.7.0` - Released against Isis 1.7.0.
* `1.6.0` - re-released as part of isisaddons, with classes under package `org.isisaddons.module.publishing`


## Legal Stuff ##
 
#### License ####

    Copyright 2013~2016 Dan Haywood

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


#### Release an Interim Build ####

If you have commit access to this project (or a fork of your own) then you can create interim releases using the `interim-release.sh` script.

The idea is that this will - in a new branch - update the `dom/pom.xml` with a timestamped version (eg `1.13.0.20161017-0738`).
It then pushes the branch (and a tag) to the specified remote.

A CI server such as Jenkins can monitor the branches matching the wildcard `origin/interim/*` and create a build.
These artifacts can then be published to a snapshot repository.

For example:

    sh interim-release.sh 1.14.0 origin

where

* `1.14.0` is the base release
* `origin` is the name of the remote to which you have permissions to write to.


#### Release to Maven Central ####

The `release.sh` script automates the release process.  It performs the following:

* performs a sanity check (`mvn clean install -o`) that everything builds ok
* bumps the `pom.xml` to a specified release version, and tag
* performs a double check (`mvn clean install -o`) that everything still builds ok
* releases the code using `mvn clean deploy`
* bumps the `pom.xml` to a specified release version

For example:

    sh release.sh 1.13.0 \
                  1.14.0-SNAPSHOT \
                  dan@haywood-associates.co.uk \
                  "this is not really my passphrase"
    
where
* `$1` is the release version
* `$2` is the snapshot version
* `$3` is the email of the secret key (`~/.gnupg/secring.gpg`) to use for signing
* `$4` is the corresponding passphrase for that secret key.

Other ways of specifying the key and passphrase are available, see the `pgp-maven-plugin`'s 
[documentation](http://kohsuke.org/pgp-maven-plugin/secretkey.html)).

If the script completes successfully, then push changes:

    git push origin master
    git push origin 1.13.0

If the script fails to complete, then identify the cause, perform a `git reset --hard` to start over and fix the issue
before trying again.  Note that in the `dom`'s `pom.xml` the `nexus-staging-maven-plugin` has the 
`autoReleaseAfterClose` setting set to `true` (to automatically stage, close and the release the repo).  You may want
to set this to `false` if debugging an issue.
 
According to Sonatype's guide, it takes about 10 minutes to sync, but up to 2 hours to update [search](http://search.maven.org).

## Appendix: yuml.me DSL

<pre>
[PublishingService{bg:green}]++-serializesUsing>[EventSerializer{bg:green}]
[PublishingService]++-defaultOpf>[PublishedObject.PayloadFactory]
[PublishingService]++-defaultApf>[PublishedAction.PayloadFactory]
[EventPayload{bg:pink}]^-[EventPayloadForChangedObject{bg:pink}]
[EventPayload]^-[EventPayloadForActionInvocation{bg:pink}]
[PublishedObject{bg:yellow}]annotatedWith-.->[PublishedObject.PayloadFactory]
[PublishedAction{bg:yellow}]annotatedWith-.->[PublishedAction.PayloadFactory]
[PublishedObject.PayloadFactory{bg:blue}]creates-.->[EventPayloadForChangedObject]
[PublishedAction.PayloadFactory{bg:blue}]creates-.->[EventPayloadForActionInvocation]
[EventMetadata]
[EventSerializer]-serializes>[EventMetadata]
[EventSerializer]-serializes>[EventPayload]
</pre>
