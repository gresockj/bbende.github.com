---
layout: post
title: "Apache NiFi Class Loading"
description: ""
category: "Development"
tags: [NiFi, Java]
---
{% include JB/setup %}

Most Java developers have probably experienced conflicting versions of a library
on the classpath and the resulting runtime issues such as NoSuchMethodError. This usually happens
when an application uses two unrelated client libraries, that each transitively depend on different versions
of another library.

Apache NiFi has traditionally provided isolated class loading per NiFi Archive (NAR). This means that
a given NAR only knows about the libraries included in that NAR, and will not be affected if another
NAR has different versions of the same libraries.

Some components (processors, controller services, reporting tasks) may need to load additional libraries
provided by the user at runtime. For
example, since NiFi can't bundle all of the possible JDBC drivers, the DBCPConnectionPool allows a user
to specify the location of a JAR containing a driver. There could be two DBCPConnectionPools, one
pointing at a MySQL driver, and the other pointing at a Postgres driver.

Up until recently, if a component required loading additional libraries as described above, it was the responsibility of the developer to handle the appropriate class loading. The upcoming NiFi 1.1 release
will introduce a new "per-instance" class loading capability as part of the NiFi framework, and will
simplify these scenarios for the developer.  

The remainder of this post will take a look at how NAR class loading has traditionally worked, and then
give an overview of the new capabilities introduced in the upcoming NiFi 1.1 release.

### NAR Class Loading

As an example, lets imagine we have two NARs, NAR1 and NAR2.

NAR1 bundles a processors JAR containing Processor1, and Processor1 uses Guava version 18, so the guava-18.0.jar is also bundled in NAR1.

NAR2 bundles a processors JAR containing Processor2,
and Processor2 uses Guava 17, so the guava-17.0.jar is also bundled in NAR2.

This would look like the following:

<img src="{{ BASE_PATH }}/assets/images/nifi-class-loading/01-nar-class-loading.png" class="img-responsive">

During start up, NiFi creates a NarClassLoader for each NAR in the lib directory. Whenever a component is
created, which could be from adding a component through the UI, or when loading an existing flow, the
framework looks up which NAR contains the class of the component, and uses that NAR's class loader to
instantiate an instance of the class.

In our example, the NARClassLoader for NAR1 is responsible for loading the Processor1 class, as well as all
dependent classes from the Guava 18 JAR. If multiple instances of Processor1 are created, the Processor1
class and all Guava 18 classes are only loaded once and reused by the NARClassLoader.

When an instance
of Processor2 is created, the NARClassLoader for NAR2 will load the class for Processor2, as well as all dependent classes from the Guava 17 JAR, and will have no idea about the classes loaded by NAR1's class loader.

This is how class loading has traditionally worked in all previous NiFi releases.

### Per-Instance Class Loading

The upcoming 1.1 release introduces a class loader for each instance of a component. By default, the
InstanceClassLoader has the NARClassLoader as it's parent. This allows additional resources to be added
to the InstanceClassLoader, but fallback to the NARClassLoader for everything else.

<img src="{{ BASE_PATH }}/assets/images/nifi-class-loading/02-instance-class-loading-with-nar-parent.png" class="img-responsive">

For most components the InstanceClassLoader will act as a simple pass-through to the NARClassLoader. However,
a component may declare a PropertyDescriptor representing additional resources to add to the classpath.

In our example above, lets imagine Processor1 declared the following property:

      PropertyDescriptor ADDITIONAL_JAR_LOCATION =
        new PropertyDescriptor.Builder()
            .name("Additional JAR Location")
            .description("The full path to a directory or JAR
                that should be added to the classpath.")
            .addValidator(StandardValidators.FILE_EXISTS_VALIDATOR)
            .expressionLanguageSupported(true)
            .dynamicallyModifiesClasspath(true)
            .build();

The **dynamicallyModifiesClasspath** builder method indicates to the framework that this property should
be processed as a classpath resource.

If the first instance of Processor1 had this property set to "/opt/libraries/library1.jar", then library1.jar would be added only to the InstanceClassLoader for the
first instance of Processor1.

If the second instance of Processor1 had this property set to "/opt/libraries/library2.jar", then library2.jar would be added only to the InstanceClassLoader for the
second instance of Processor1.

In this scenario, the Processor1 class and all the Guava 18 classes would still be loaded only once by the NARClassLoader.

### Fully Isolated Per-Instance Class Loading

Sometimes it is necessary to completely isolate all the classes for each instance of a component,
including the component class and its library classes normally loaded by the NARClassLoader. This can be
done by placing the **@RequiresInstanceClassLoading** annotation on a component.

When the framework sees this annotation, rather than creating an InstanceClassLoader with the NARClassLoader
as a parent, it instead adds all of the resource locations from the NARClassLoader to the
InstanceClassLoader so that they will be reloaded for that instance.

<img src="{{ BASE_PATH }}/assets/images/nifi-class-loading/03-instance-class-loading-copy-of-nar.png" class="img-responsive">

In our example above, if Processor1 and Processor2 had the **@RequiresInstanceClassLoading** annotation,
the processor classes and Guava classes would be loaded multiple times by each InstanceClassLoader, and
the NARClassLoader serves only as a source of information to access the resource locations for the given NAR.

A possible reason for doing this would be to guarantee that no state is shared between classes in a client library. For example, if a class in library1.jar has a static variable, we may not want the first instance
of Processor1 to see changes to that static variable made by the second instance of Processor1. By having
completely separate InstanceClassLoaders, all of the classes in library1.jar will be loaded separately in
each InstanceClassLoader, and won't know anything about each other.

It is important to mention that when using the **@RequiresInstanceClassLoading** annotation, it could
result in a higher memory footprint, meaning if you created 100 instances of Processor1, the classes
required by Processor1 would be loaded in memory 100 times, rather than once.

### Summary

The per-instance class loading capabilities allow component developers to more easily
add resources to the classpath at runtime, and give the component developer more control of how
to isolate classes.  

If a component simply needs to add a few JARs to the classpath of a component at runtime, then
this can be done by specifying **dynamicallyModifiesClasspath(true)** on the property.

If the component needs to guarantee complete isolation of classes across instances, then the component
can also declare the **@RequiresInstanceClassLoading** annotation.
