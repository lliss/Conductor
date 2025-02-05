# Conductor #

This module is in the very earliest stages of development and at the
time of writing is not functional.

The goal is to create a module akin to [maestro](http://drupal.org/project/maestro) but
built on top of [ctools](http://drupal.org/project/ctools) with an architecture inspired
by [views](http://drupal.org/project/views) with proper exportables support and a much
higher degree of flexibility and to allow realtime message handling, rather than requiring
later cron jobs.  The hope is to support [Rules](http://drupal.org/project/rules)
actions and conditions in the near future, enabling an enormous amount of functionality
without reinventing the wheel.

## Performance ##

Conductor aims to be highly performant and scalable with capabilities built in, from the
beginning, to run without hitting the database any more than is absolutely required.  Workflows are cached
allowing them to be loaded by name easily from in memory key-value stores.  Persistence of state
is optional for workflows that do not need to be resumed.  Workflow state storage has a pluggable
storage backend allowing it, too, to be persisted to a NoSQL or in memory data store.


## Terminology ##

1. Workflow
2. Activity
3. Workflow State

### 1. Workflow ###

A workflow is a process with a beginning and an end.  These processes have different steps, generically refereed to as
activities, that a particular workflow moves through.

#### Storage ####

Conductor workflows are implemented as [ctools exportables](http://drupalcode.org/project/ctools.git/blob_plain/7.x-1.x:/help/export.html).
This means they may be provided in code or in the database and if there is a rendition
in both code and the database the version in the database is used but may be reverted
to the version specified in code via the user interface.

Conductor workflows want to cache themselves.  Workflows in the database will be cached
so that they can be retrieved from memory on sites using [memcached](http://drupal.org/project/memcache)
without making additional database calls for processing.

### 2. Activity ###

An activity is a single step in a workflow process.
Activities are implemented as [ctools plugins](http://drupalcode.org/project/ctools.git/blob_plain/7.x-1.x:/help/plugins-api.html).

####Storage ####

Activities are members of the workflow object and are stored in a serialized or exported
format inside the workflow itself.  There is no external storage of activities beyond a
workflow that a given activity is a member of.


### 3. Workflow State ###

Workflow state represents a single instance of a workflow.

####Storage ####

Workflow state is stored using another ctools plugin and so storage can be swapped out.
The activities on a workflow are stored with the workflow state in their configuration
at the time that workflow is stored.  This is important because, without this,
changing the workflow while a particular instance is in the process of completion could
result in stranded workflow items with inconsistent state.

## About workflow processing ##

Conductor workflows are currently started off by some module other than conductor which
hands in whatever context is necessary for processing.  This becomes the bases for the
workflow state which is passed from activity to activity.  The activity can then ask
questions of the WorkflowActivityState descended object and use the context present to perform
its own activities.

When an activity is completed (always starting with the `start` activity), its name is
added to the `completedActivities` array in the WorkflowState object and each of its
specified outputs is activated making them eligible for processing.

Workflow activities are designed to handle branching conditions.  In this way, the final context
is used to trigger one or more new activities. For example, upon completion an activity could trigger two more activities, one to create a log entry and another to continue the movement of 
data through the workflow.  The final goal is to use conditional logic (if, and, or) to support branching.  In this way the results of one activity create different paths through the workflow.     

## Scalability ##

Conductor is designed to be a fully integrated Drupal module and rely on nothing more than Drupal and dependent contributed Drupal modules to function.  However it is being designed with high load scenarios in mind and as such queuing of data and the use alternate storage engines for short term data handling.

## Architecture ##

Conductor relies on object-oriented data structures.  Workflows, activities, and activity states are all stored as objects and each has its own necessary properties and methods.
Activity and activity state objects are contained within the workflow object.  This allows activities and activity states to poll the workflow for other activities and activity states.  This design also allows for the entire workflow to be stored while waiting for needed conditions for an activity to be met.  Essentially a started workflow has a list of necessary tasks as designated at its instantiation.  The analogy is to consider sending a spouse to the market with a grocery list.  After leaving the house it is not possible to alter the list until the spouse returns home.  A that time a new list or an adjusted list can be issued and the spouse can be sent to the market again (analogous to restarting a workflow with new or altered activities).  This design ensures data integrity.

## Updating Conductor ##

One question that needs to be answered for Conductor to be a viable option as the center
point for a site's workflow management, is how we are going to manage updating to a new
version of workflow with processes in the midst of processing.  At the time of writing the
API is highly unstable and we can't make any promises about whether changes will break
existing workflows.
