# The Sling PetClinic Application

This application is intended to provide a fast and easy access to Apache Sling using Groovy as a scripting language.

> This tutorial aims to provide an example of how the pieces are connected. Making it easier to start developing
> with Sling.

## Before getting started - getting Sling

I highly recommend checking the Sling sources out from the repository (GIT or SVN) and compiling it by hand. The launchpad
provided by the apache project side is out date.

## Getting started

The parent project will be created from a maven-archetype-quickstart.


    #!/bin/sh
    mvn archetype:generate \
        -DarchetypeGroupId=org.apache.maven.archetypes \
        -DarchetypeArtifactId=maven-archetype-quickstart \
        -DgroupId=io.github.floriansalihovic.sling \
        -DartifactId=pet-clinic \
        -Dpackage=io.github.floriansalihovic.petclinic \
        -Dversion=1.0.0-SNAPSHOT

It is the parent project for modules, which make up the application. After the project was created, a few first changes
should be made to the pom.


    <properties>
      <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
      <sling.user>admin</sling.user>
      <sling.password>admin</sling.password>
      <sling.port>8080</sling.port>
    </properties>

Adding the properties ```sling.user```, ``sling.password``` and ```sling.port``` provides easy access to the basic
configuration of Sling. These configuration properties are needed in modules to access Sling for deployments.

The main module needed for this project is a ```ui``` project. It contains all aspects of the project necessary for
displaying data.


    mvn archetype:generate \
        -DarchetypeGroupId=org.apache.sling \
        -DarchetypeArtifactId=sling-initial-content-archetype \
        -DarchetypeVersion=1.0.0 \
        -DgroupId=io.github.floriansalihovic.petclinic \
        -DartifactId=pet-clinic-ui \
        -Dversion=1.0.0-SNAPSHOT

This command will create the module along with some initial data (definitions, content and a script). The data is
stored under ```pet-clinic-ui/src/main/SLING-INF```.

- ```content/```contains the initial content provided by the archetype.
- ```nodetypes/```provides and example node type, which is basically a data type definition for the Java Content Repository.
- ```scripts/``` provides a basic example for a Sling resource renderer.

When the module is added, some first changes should be made.

- Using the properties defined in the parent ```pom```.
- Updating the plugin versions.
- Renaming ```SLING-INF/scripts``` to ```SLING-INF/apps``` and also adding the directories ```my/node```.
  This is a very sensitive change, because also the the ```pom```'s content needs to be changed accordingly.


    <!-- project.build.plugins -->
    <plugin>
      <groupId>org.apache.felix</groupId>
      <artifactId>maven-bundle-plugin</artifactId>
      <extensions>true</extensions>
      <version>2.4.0</version>
      <configuration>
        <instructions>
          <Sling-Nodetypes>SLING-INF/nodetypes/nodetypes.cnd</Sling-Nodetypes>
          <Sling-Initial-Content>
            SLING-INF/apps/my/node;overwrite:=true;uninstall:=true;path:=/apps/my/node,
            SLING-INF/content;overwrite:=true;uninstall:=true;path:=/content
          </Sling-Initial-Content>
        </instructions>
      </configuration>
    </plugin>


The change made was very subtle. From ```SLING-INF/scripts;``` to ```SLING-INF/apps/my/node;```. This was done mainly
for initial consistency of the project. The way scripts (components) are resolved in the repository follows a strict
scheme, which needs to be understood in order to work with sling efficiently. A detailed guide on components follows
later in the tutorial.

## Initial content

Before working efficiently on the application, providing some data to work with is needed. In Sling, adding content can
be done in a variety of ways. One would be a simple POST via ```curl```from the command line:


    #!/bin/sh
    curl -FfirstName="George" \
         -FlastName="Franklin" \
         -Faddress="Madison, 110 W. Liberty St." \
         -Ftelephone="6085551023" \
         http://localhost:8080/sling/content/owners/georgefranklin

This would create the node ```/sling/content/owners/georgefranklin``` with the given properties. A more reusable
approach is providing a module with demo content, which can be refined as the application matures. Using the same Maven
command used when creating the ```pet-clinic-ui``` with a different ```archetypeId``` generates the appropriate module.


    mvn archetype:generate \
        -DarchetypeGroupId=org.apache.sling \
        -DarchetypeArtifactId=sling-initial-content-archetype \
        -DarchetypeVersion=1.0.0 \
        -DgroupId=io.github.floriansalihovic.petclinic \
        -DartifactId=pet-clinic-demo-content \
        -Dversion=1.0.0-SNAPSHOT

The module's layout equals the one of the ```pet-clinic-ui``` so we make the same modifications. But since this module
is used as a container for content only, the folders ```scripts/``` and ```nodetypes/``` can be deleted right away. In
```pet-clinic-ui```, the folder ```content/``` can be deleted - the content will be stored in ```pet-clinic-demo-content```.

Running ```mvn install -PautoInstallBundle``` will install the project in the local repository and deploy it to the
running Sling instance. When calling ```http://localhost:8080/my-first-node.html``` a simple representation of
```my-first-node.xml``` is displayed. Similar to my-first-node.xml the application's initial content is set up. By
providing content under ```SLING-INF/sling/content/owners``` like


    <node>
      <primaryNodeType>nt:unstructured</primaryNodeType>

      <property>
        <name>firstName</name>
        <type>String</type>
        <value>George</value>
      </property>

      <property>
        <name>lastName</name>
        <type>String</type>
        <value>Franklin</value>
      </property>

      <property>
        <name>address</name>
        <type>String</type>
        <value>110 W. Liberty St.</value>
      </property>

      <property>
        <name>city</name>
        <type>String</type>
        <value>Madison</value>
      </property>

      <property>
        <name>telephone</name>
        <type>String</type>
        <value>6085551023</value>
      </property>

    </node>

test data is added. The archetype's initial content and script are kept in order to have reference data throughout the tutorial -
until dedicated render components are provided. To deploy the data properly, the ```SLING-INF/sling/content/owners```
directory has to be added to the plugins ```Sling-Initial-Content`` configuration.


    <plugin>
      <groupId>org.apache.felix</groupId>
      <artifactId>maven-bundle-plugin</artifactId>
      <extensions>true</extensions>
      <version>2.4.0</version>
      <configuration>
        <instructions>
          <Sling-Nodetypes>SLING-INF/nodetypes/nodetypes.cnd</Sling-Nodetypes>
          <Sling-Initial-Content>
            SLING-INF/apps;overwrite:=true;uninstall:=true;path:=/apps,
            SLING-INF/content;overwrite:=true;uninstall:=true;path:=/content,
            SLING-INF/sling/content/owners;overwrite:=true;uninstall:=true;path:=/sling/content/owners
          </Sling-Initial-Content>
        </instructions>
      </configuration>
    </plugin>

## Everything is a resource

The mantra of Sling application design is "Everything is a resource". The programming model allows addressing resources
rendered in various ways by providing a resource type. The resource type is used for script resolution, which is basically
the process of determining the correct way to render a resource.

In the PetClinic application, different types of resource are easily identified:

- Pages
- Owners (of pets)
- Pets
- Vetenarians
- Visits

JackRabbit - the underlying content repository (basically the persistence layer) - is an unstructured data store.
There is no schema and those resources can basically be stored at any location. The obvious differentiation made divides
resources in data and resource which render data - associated via resource type. Not going into the resolution in full
depth, the latter resource are stored in ```/libs``` and ```/apps```. The locations can be looked up in the [Apache Felix
OSGi](http://localhost:8080/system/console/jcrresolver) (Sling must be running) console. Those resources are refered to as scripts,
renderers or (view) components. In ```/libs``` standard Sling components are stored, whereas in ```/apps``` application
specific scripts are stored, which can also override components provided in ```/libs``` (by resource type as illustrated
in the following short example).

Having a resource with a resource type of ```owners/owner```, Sling will look into ```/apps/owners/owner``` for any kind
of component used for displaying and if none is found, ```/libs/owners/owner``` will be checked. That is the short story.

The data and the selected scripts for rendering have to be set in a broader context though. When working with data, or
resources in the context of Sling, there are always various resources involved in creating and modifying data.

The outline can be imagined as the following the user wants to create a visit for his pet ar the veterinarian. In order
to do so, he has to register himself and his pet at for a visit at a certain time for a visit at a specific pet clinic
run by a doctor.

So naturally pets and their owners, the vets and so on appear to be resource. But so are the HTML pages requested. Sling
does not differentiate between those subjects in terms of resources - only how the data may be accessed, modified and
displayed.

This tutorial aims to provide an example of how the pieces are connected.

## About resources and resource types in the PetClinic

The application is basically a Sling flavored [Spring PetClinic](http://docs.spring.io/docs/petclinic.html) version based
on the [Kotlin](https://github.com/cheptsov/kotlin-nosql-mongodb-petclinic) port. That means the objectives are set and
a port to Sling can be done without looking too much left and right.

The application provides basically a web interfaces to add and modify everything a pet clinic would need to work with.
As already stated, pages are resources just as entities in the domain data and Sling provides a lot of power for creating
them.