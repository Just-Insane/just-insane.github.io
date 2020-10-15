---
title: 'Why Scripting is no longer enough'
date: '07-07-2020 00:00'
published: false
categories:
  - 'Blog'
  - 'Automation'
tag:
  - 'Scripting'
  - 'Dev'
---

Scripting, as most poeple know it, is no longer enough. With the rise of cloud technologies and larger environments,
it is no longer enough to have scripts manually run by systems administrators when something goes wrong.

In this blog post we will explore the absurd statement that scripting is no longer enough, and what it means for companies as we move into the cloud era of computing.

## Scripting; Then vs Now

In the past many systems administrators would create scripts for repetitive tasks which they would run manually when needed to resolve problems or complete work.

This works fine when there is just a handful of systems that need to be interacted with. However with the larger adoption of cloud computing, there are ever increasing numbers of
systems that need to be touched in order to perform a task.

For an example of this, let's look at how a company's helpdesk may reset a user's password. In the past, they may have gone into Active Directory Users and Computers, searched for the user,
then right-clicked on their account and reset the password, having to type the new password in twice and clicking OK. Once they hit a larger scale, they will likely find that this is not
fast enough to keep up with their demand for password resets, and may create a script that allows them to type in a user's name then prompt them if and what they want to reset the password to.
This works fine until you start adding in other services, like Azure AD, Federated Services, or other tasks which the helpdesk may require to be effective and swift in their actions. Eventually
you get a bloated utility which is hard to maintain and takes longer to use then manually resetting the password. Sure, there may be a ton of extra features now, however, how often are they actually used?
Wouldn't it be better if users could reset their own passwords? Doing so would allow for your people to be more effective, would require less maintenance of your script, and would improve the overall quality of life of users and administrators.

The same could be said for any script which is manually (or even scheduled to run automatically). It is unlikely that these scripts are being as effective as building the functionality directly into your systems.
If for example, you have a script which takes a CSV file and automatically creates user accounts in your system once a day, depending on your turnover, you could either increase the number of times
per day the script runs, or you could integreate that functionality into the system were the user information is stored. Removing the middle steps of generating a CSV, sending it off, having a script process it,
then take actions to create accounts is time consuming and error prone. What if there is more information needed to create accounts? Then you have to add data to the CSV, update your script (with error handing),
test it, and then migrate both sides of the process at the same time. Alternativly, you could build the functionality of creating user accounts directly into the system that stores your user information.
It may be more work in the short term, but overall results in less time creating and maintaining scripts that do the same with more steps.

As more tasks are moved to the cloud, it becomes even easier to move these tasks out of scripts and into the supporting programs though the use of well-known and secure APIs.

## Reactive vs Proactive Automation

In the previous examples we looked at reactive automations which were done when something happens. Now, let's take a look at how we could change those automations from being reactive to being proactive.

Most modern logging and monitoring solutions have the ability to run automations (scripts) when they detect an issue **is** occuring. If you aren't using a monitoring solution that can run automations, or isn't monitoring your whole environment, you may want to look at upgrading that first, as observability is critical to making informed decisions and keeping a well-running ship.

Notice how in the previous paragraph I mention that automations are run when an issue is occuring? That is very important. Your systems should be able to take a set action (or actions) when they detect an anomily. Of couse, this should still be logged for human review (automatic creation of service tickets usually works best), and if issues continue, an alert should be fired. The main benefit of this is that issues can be resolved before human interaction is required (for example, a VM out of drive space leading to automatic addition of space and drive partitioning). This frees up your team to work on more important tasks, and investigate instances where drives are filling up consistently. Note that it is important for these automated fixes to be [idempotent](https://stackoverflow.com/a/1077421).
