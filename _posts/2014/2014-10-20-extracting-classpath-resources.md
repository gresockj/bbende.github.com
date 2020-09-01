---
layout: post
title: "Extracting Classpath Resources"
description: ""
category: "Development"
tags: [Java]
---
{% include JB/setup %}

I recently wanted to write some code that would recursively process a classpath 
location and extract the contents to the local filesystem. An example being 
a directory of configuration files in a Maven project under src/main/resources 
which gets packaged into a jar, and then extracted later when the jar is executed.

Normally if you need to read a file from the classpath you would use getResourceAsStream()
from the ClassLoader:
<pre><code>this.getClass().getClassLoader().getResourceAsStream(resource);</code></pre>

This only works for a single resource that you know of, but in this scenario
you don't know all of the resources ahead of time. After reading a lot 
of StackOverflow posts I came across [this one](http://stackoverflow.com/questions/3923129/get-a-list-of-resources-from-classpath-directory) 
which had a nice solution that didn't rely on any third-party libraries. I'm not taking
credit for creating this solution, but wanted to cover it since it wasn't obvious to me
at first.

The starting point of the solution is to get a list of all the resources on the classpath 
from the java.class.path system property:

    public static Collection<String> getResources(final Pattern pattern){
        final ArrayList<String> retval = new ArrayList<String>();
        final String classPath = 
            System.getProperty("java.class.path", ".");
        final String[] classPathElements = classPath.split(
            File.pathSeparator);
        for(final String element : classPathElements){
            retval.addAll(getResources(element, pattern));
        }
        return retval;
    }

After getting that list the next thing to do is check each element and determine if the element
is a directory on the file system, or a jar. This is done by creating a File object for each
element and checking the isDirectory() method. If it returns false the element is assumed to be a
jar:

    private static Collection<String> getResources(
            final String element, final Pattern pattern){
        final ArrayList<String> retval = new ArrayList<String>();
        final File file = new File(element);
        if(file.isDirectory()){
            retval.addAll(getResourcesFromDirectory(file, pattern));
        } else{
            retval.addAll(getResourcesFromJarFile(file, pattern));
        }
        return retval;
     }
     
I'm going to skip over the case of a directory, but in the case of a jar, all the entries
are read using the ZipFile class:

    private static Collection<String> getResourcesFromJarFile(
            final File file, final Pattern pattern){
        final ArrayList<String> retval = new ArrayList<String>();
        ZipFile zf = null;
        try{
            zf = new ZipFile(file);
            final Enumeration e = zf.entries();
            while(e.hasMoreElements()){
                final ZipEntry ze = (ZipEntry) e.nextElement();
                final String fileName = ze.getName();
                final boolean accept = pattern.matcher(fileName).matches();
                if(accept){
                    retval.add(fileName);
                }
            }
        } catch(final ZipException e){
            throw new Error(e);
        } catch(final IOException e){
            throw new Error(e);
        } finally {
            try {
                zf.close();
            } catch (final IOException e1) {
                throw new Error(e1);
            }
        }
        return retval;
    }
    
Now once you have this list of resources you can use getResourceAsStream() to read each
resource and write the contents somewhere:

    Collection<String> resources = ResourceList.getResources(pattern);
    for (String resource : resources) {
        InputStream in = null;
        try {
            in = this.getClass().getClassLoader()
                                .getResourceAsStream(resource);
            ...
        } catch (Exception e) {
            ...
        } finally {
            // close stream
        }
    }
    
Something worth noting is that when testing this from an IDE, the classpath of your project
will be based off the target directory where the IDE built/compiled your project, and this
is where the contents of src/main/resources will end up, so the above code would find a 
directory and not a jar file. The same holds true when building a single Maven project. 

You can test finding the resources in a jar file by setting up two projects where project2 depends 
on project1, and project2 tries to get the resource list from a directory in src/main/resources of 
project1. If you build these projects on the command line, project2 will have to access the project1 
jar to find the resources.