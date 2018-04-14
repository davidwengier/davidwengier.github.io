---
layout: post
title: ! Reviewable Stored Procs and Views with DbUp
published: false
---

Where I currently work we use [DbUp](https://dbup.github.io/) to manage database changes and migrations and for the most part it works fine as long as you have a known schema that you're coming from. At my previous work we had a different model where our migration software would programatically work out what actions to perform based on the schema of the database you were pointing to, and the known definition of the database. This meant you could be in any state, including not having a database at all, and after the migration you would be guaranteed to be at the latest.

One of the benefits of that system was that for Views and Stored Procedures we simply stored the defition as a `.sql` file in our git repository and the migration software took care of the rest. This meant that any changes to views or stored procs were easily reviewable as part of a normal code review or pull request process.

Fortunatly enabling this workflow with DbUp is very straightforward.

## Project format

Our DbUp project looks fairly standard:

[img]

DbUp takes care of running the scripts and making sure none are run more than once via its in build journaling system, a record of which is also stored in the database.

The first step in enabling reviewable stored procs and views is to create a new folder for scripts that will be unjournaled, so they are always run whenever DbUp is run. The basic idea is that you seed your repository with SQL scripts that contain a simple DROP and CREATE script for each stored procedure and view. DbUp runs these scripts every time, essentially making sure the database definition is always correct, and allowing developers to make changes to the existing scripts in the source repository, rather than having to create new ones all the time, as with normal journaled scripts.

[img of DROP and CREATE script]

## The DbUp script runner

The existing DbUp script runner looks fairly basic, like this:

[code]

We need to add a new upgrader to this script and instead of storing the journal in a table we will use the `NullJournal` that is build into DbUp

[code with null journal]

The last piece of the puzzle is to put a filter onto each upgrader so each one only loads the scripts we want. The final code looks like this:

[code with both, and run commands]

## Get reviewing

Every change to the stored proc of view definition script will be just that - a change - so whatever source repository diff process you use will show only what has been done. Additionally you always have the current up-to-date definitions of your scripts in your source repository so you're one step closer to not having to worry about having a known good starting point for your database, at least from the schema point of view.

So far we're rolling this out on a change-by-change basis, but there is no reason all of the relevant parts of the database couldn't be scripted to seed this effort giving you a known baseline.