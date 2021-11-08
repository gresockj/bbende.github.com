---
layout: post
title: "Apache NiFi 1.15.0 - Parameter Context Inheritance"
description: ""
author: Joe Gresock
category: "Development"
tags: [NiFi,Parameter Contexts]
---
{% include JB/setup %}

#### (Guest author: Joe Gresock)

Here we discuss the benefits and intricacies of the new Parameter Context Inheritance in Apache NiFi 1.15.0.

### Introduction

The Parameter Context is a powerful way to make flows more portable in Apache NiFi.  Paired with Process Groups imported from Flow Definition JSON or NiFi Registry, a Parameter Context allows different values to be supplied to the same basic flow in multiple NiFi instances or even within the same overall NiFi flow in a single instance.  

However, as the number of parameters grows, Parameter Contexts can become more difficult to maintain, resulting in the inevitable Monolithic Parameter Context.  Let's see how this happens:
1. You have a Process Group called "Source ABC Kafka to S3" with parameters for your Kafka brokers and Source ABC's topic, and for the S3 bucket.
2. You have another Process Group called "Source DEF Kafka to GCS" with parameters for the same Kafka brokers but Source DEF's topic, and for the GCP Cloud Storage bucket.
3. You have a third Process Group called "Source GHI Kafka to Elasticsearch" with parameters for the same Kafka brokers but Source GHI's topic, and for the Elasticsearch hosts, username, and password.
4. In all of your Process Groups, you use some common Parameters tagging the flowfiles with site-specific information.
5. Now, where do you put all those parameters: in multiple separate Parameter Contexts, or in one Monolithic Parameter Context?  You don't want to keep repeating the Kafka brokers or the common site-specific parameters, and you also don't want to break up these Process Groups because these are the units that you deploy atomically across your organization in different sites.  So, you go with the Monolithic approach.

However, the Monolithic Parameter Context has its own problems:
- You often end up with multiple slightly different parameters, like "Source ABC Kafka Topic" and "Source DEF Kafka Topic".  Didn't you make the Parameter Context to parameterize the topic in the first place?
- Parameterizing a Process Group with more Parameters than it needs is a bad practice, resulting in confusing extra parameters.

What if you could avoid the Parameter duplication and yet still compose groups of Parameters as needed?

### Introducing Parameter Context Inheritance

With Apache NiFi 1.5.0 comes the ability to add Inherited Parameter Contexts to an existing Parameter Context.  Any inherited Parameters are also available in the Context that inherits them, down to as many levels of inheritance as you desire.  This structure allows Parameter Contexts to contain smaller groups of related Parameters, leaving the inheritance to group disparate sets of Parameter Contexts when needed.

Let's see what it looks like!

First, select Parameter Contexts from the top-right hamburger menu in NiFi:

<img src="{{ BASE_PATH }}/assets/images/nifi-parameter-context-inheritance/01-param-context-menu.png" class="img-responsive" width="30%" height="30%">

Then click the '+' button on the top-right of the NiFi Parameter Contexts view to add a new Parameter Context.

<img src="{{ BASE_PATH }}/assets/images/nifi-parameter-context-inheritance/02-param-context-view.png" class="img-responsive">

You'll notice a new tab, "INHERITANCE".  We'll get to that in a moment, but first let's simulate the example above to make it more concrete.

### Designing with Smaller, Composable Contexts

The ability to compose Parameter Contexts through inheritance suggests an overall preference of smaller, logically grouped Parameter Contexts.  Let's see how this works in the example from the Introduction.

Note: the entire flow discussed below, along with Parameter Contexts, can be downloaded here to save time in constructing it: <a href="{{ BASE_PATH }}/assets/attachments/nifi-parameter-context-inheritance/Parameter_Context_Inheritance_Demo.json" target="_blank">Parameter_Context_Inheritance_Demo.json</a>

First, create a context for our common Kafka settings:
1. Name: `Kafka`
2. Parameters:
  - Kafka Brokers: `localhost:9092`
  - Kafka Group ID: `MyGroup`

Next, create a context for each of the Kafka topics:
1. Name: `Source ABC`, with Parameter "Kafka Topic": `source-abc-topic`
2. Name: `Source DEF`, with Parameter "Kafka Topic": `source-def-topic`
3. Name: `Source GHI`, with Parameter "Kafka Topic": `source-ghi-topic`

Perhaps you would prefer to name these like "Source ABC Kafka", but this depends on your overall use case.

Then we'll add a Parameter Context for S3 settings:
1. Name: `S3`
2. Parameters:
  - S3 Bucket: `my-bucket`
  - S3 Region: `us-west-2`

Then we'll add a Parameter Context for GCP Cloud Storage settings:
1. Name: `GCP Cloud Storage`
2. Parameters:
  - GCP Project ID: `my-project`
  - GCS Bucket: `my-bucket`

Then we'll add a Parameter Context for Elasticsearch settings:
1. Name: `Elasticsearch`
2. Parameters:
  - Elasticsearch URL: `http://localhost:9200`
  - Elasticsearch Username: `elastic`
  - Elasticsearch Password: `password`

Finally, we'll add a Parameter Context for the common site-specific parameters:
1. Name: `Site Properties`
2. Parameters:
  - Site Identifier: `1234`
  - Site Data Manager: `MyDM`

When we're all done, our Parameter Contexts should look like this:

<img src="{{ BASE_PATH }}/assets/images/nifi-parameter-context-inheritance/03-param-context-list.png" class="img-responsive">

### Adding the Inheritance

Now that we have our Parameter Contexts set up, we can begin composing them.  Our goal here is to have one Parameter Context per Process Group, composed of the smaller ones.

Notice in our example that we will need both the `Site Properties` and the `Kafka` Parameters in all of our Process Groups.  Then, we'll need one of each `Source` Context in each Process Group, and one of each destination-related Context in each group.  Further, let's add the detail that we know we'll need additional Process Groups with the same Kafka Brokers and Topics but with different destinations in the future.  This suggests the following composition:

- `Kafka and Site` - inherits from `Kafka` and `Site Properties`
- `ABC Kafka and Site` - inherits from `Kafka and Site` and `Source ABC`
- `DEF Kafka and Site` - inherits from `Kafka and Site` and `Source DEF`
- `GHI Kafka and Site` - inherits from `Kafka and Site` and `Source GHI`
- `ABC Kafka to S3` - inherits from `ABC Kafka and Site` and `S3`
- `DEF Kafka to GCS` - inherits from `DEF Kafka and Site` and `GCP Cloud Storage`
- `GHI Kafka to Elasticsearch` - inherits from `ABC Kafka and Site` and `Elasticsearch`

The hierarchy for one of these top-level Parameter Contexts would look like this:
- `ABC Kafka to S3` Context
  - `ABC Kafka and Site` Context
    - `Kafka and Site` Context
      - `Kafka` Context
        - `Kafka Brokers` Parameter with value `localhost:9092`
        - `Kafka Group ID` Parameter with value `MyGroup`
      - `Site Properties` Context
        - `Site Identifier` Parameter with value `1234`
        - `Site Data Manager` Parameter with value `MyDM`
    - `Source ABC` Context
      - `Kafka Topic` Parameter with value `source-abc-topic`

Let's compose one of these examples, and the rest can be seen by downloading the flow above.

First, create a new Parameter Context named `Kafka and Site` and go to the Inheritance tab:

<img src="{{ BASE_PATH }}/assets/images/nifi-parameter-context-inheritance/04-inheritance-example-1.png" class="img-responsive">

There are all of our individual Parameter Contexts!  Let's drag `Kafka` to the right:

<img src="{{ BASE_PATH }}/assets/images/nifi-parameter-context-inheritance/05-inheritance-example-2.png" class="img-responsive">

Then drag `Site Properties` to the right (doesn't matter if it's on the top or bottom for now):

<img src="{{ BASE_PATH }}/assets/images/nifi-parameter-context-inheritance/06-inheritance-example-3.png" class="img-responsive">

Click Apply, and then Edit `Kafka and Site` to see the new Parameters:

<img src="{{ BASE_PATH }}/assets/images/nifi-parameter-context-inheritance/07-inherited-parameters-1.png" class="img-responsive">

Notice that all four Parameters from these two Contexts are available in the view, and that each of them has an arrow icon.  You can click on one of these to "Go To" the Parameter Context in which the Parameter is actually defined.  This allows you to easily navigate to the Parameter in order to edit the actual value.

Skipping ahead, we'll show what `ABC Kafka to S3` looks like after the full setup:

<img src="{{ BASE_PATH }}/assets/images/nifi-parameter-context-inheritance/08-inherited-parameters-2.png" class="img-responsive">

### Parameter Overriding

Now, let's say we want to add another Process Group that uses all the same Parameters as `ABC Kafka to S3`, but with a `Kafka Group ID` of `MyOtherGroup`.  Here we can use Parameter overriding by creating a new Parameter Context that inherits from `ABC Kafka to S3` and then adding a `Kafka Group ID` Parameter directly to that new context:

<img src="{{ BASE_PATH }}/assets/images/nifi-parameter-context-inheritance/09-overridden-parameter.png" class="img-responsive">

Notice that `Kafka Group ID` appears at the top of the list, even though it's not in alphabetical order.  Direct Parameters always appear at the top of the list so you can see them grouped together.  Also notice that you can edit this Parameter directly from this context.  You are editing the one on the new Context, rather than the inherited Parameter.  And finally, notice that the value displayed is the one you just provided: `MyOtherGroup`.

What is the Parameter overriding order, then?

### Parameter Overriding Order

- Direct Parameters always take precedence
- Next, Parameters inside directly inherited Parameter Contexts take precedence, from top to bottom in the `Selected Parameter Context` portion of the Inheritance tab
- This recursively repeats in a *depth-first* manner for as many layers of inherited Parameter Contexts exist

So, for example, consider the following Parameter Context hierarchy:
- `A` Context
  - `foo` Parameter with value `A.foo`
  - `B` Inherited Context
    - `foo` Parameter with value `B.foo`
    - `bar` Parameter with value `B.bar`
  - `C` Inherited Context
    - `foo` Parameter with value `C.foo`
    - `bar` Parameter with value `C.bar`
    - `grandchild` Inherited Context
      - `foo` Parameter with value `grandchild.foo`
      - `bar` Parameter with value `grandchild.bar`
      - `baz` Parameter with value `grandchild.baz`

Then, `A` effectively has the following Parameters:
- `A.foo` (overrides all others since it is at the top level)
- `B.bar` (overrides `C.bar` because it is first in the list of A's inherited contexts)
- `grandchild.baz` (because nothing overrides this)

If we rearranged Contexts `B` and `C` so that `C` was at the top of the list of A's inherited Parameter Contexts, the new effective list would be:
- `A.foo`
- `C.bar`
- `grandchild.baz`

And if we then deleted `C.bar`, the effective list would be:
- `A.foo`
- `grandchild.bar` (because `grandchild.bar` is in the effective list of the parameters of `C`, which appears above `B` in the inherited contexts of `A`)
- `grandchild.baz`

Finally, if we deleted `grandchild.bar`, we'd end up with:
- `A.foo`
- `b.bar`
- `grandchild.baz`

Notice that as we move these inherited Parameter Contexts around, we are actually changing the effective values of the parameters.  Once we Apply these changes, all referenced components are stopped and restarted just as if we were changing these Parameter values like usual.

Now, as far as best practices go, as we increase the number of overridden Parameters, we likely increase the potential for confusion, so it is probably best to use overriding sparingly if possible.

### Wrapping Up

So, taking our example from the Introduction as implemented with Inherited Parameter Contexts, we can now make granular updates to our Parameter Contexts and have the changes propagate to all Contexts that inherit from them.  So, if we need to change the `Site Identifier`, we only have to change this in one location: the `Site Properties` Parameter Context.  If we need to change our `Kafka Broker`, we need only change it in the `Kafka` Context.  We can also add additional Parameter Contexts through inheritance as our flows evolve.  And each Process Group now has only exactly the Parameters it needs, now that we've done away with the Monolithic Parameter Context.

In conclusion, the new Inherited Parameter Context feature of Apache NiFi 1.15.0 brings the next level of flexibility and maintainability to NiFi's already powerful Parameter Context framework.
