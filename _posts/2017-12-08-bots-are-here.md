---
layout: post
title: Bots are here
tags: [Continuous Delivery, ChatOps]
gh-repo: ivans-innovation-lab/my-company-automations
gh-badge: [star, fork, follow]
---


“Say hello to my little friend” [Atomist](https://www.atomist.com/)! It is a SaaS. It is a ChatOps. **One of it’s main interfaces is ‘Atomist Slack [ro]bot’**. My team member.

## Context

With the rise of distributed systems, project creation is more and more important to individuals and organizations, as is maintaining consistency between a potentially large number of services or components.

New challenges are rising:

 - “How can we manage aspects such as API contracts?” 
 - “How can we apply changes across several repositories at once?”
 - “How can we aggregate and connect all the data coming from PaaS, IaaS, CI tools, chat tools …?”

Those challenges highlight the need for greater collaboration between, and within, teams and this is a challenge that Atomist is addressing.

In one of my previous [posts](http://idugalic.pro/Accelerating-the-digitization/) I wrote that there is important relation between architecture, process (continuous delivery) and the culture (devops, chatops). 
For the purposes of this post we will use an [open source lab](http://lab.idugalic.pro) that I regularly maintain, and focus on [monolithic](https://docs.lab.idugalic.pro/chapter1/monolithic/overview.html) (modular) architectural pattern. You will see how Atomist is helping me steering the process.

![](https://github.com/idugalic/idugalic.github.io/raw/master/img/atomist_1.png)

## Atomist is code

[Atomist](https://www.atomist.com/) consumes your code (Java, C#, JavaScript, Scala, Python, Clojure, even Dockerfiles and Maven POMs), understanding your files, classes, variables, exceptions and more. This understanding is used to modify code directly and to connect code changes to runtime changes. You can find Atomist documentation here: [http://docs.atomist.com/](http://docs.atomist.com/)

You interact with the Atomist development automation API via a client WebSocket connection. An automation client project is any project that connects to the Atomist development automation API. The reference implementation of the automation client is the [`automation-client-ts`](https://github.com/atomist/automation-client-ts) library, which is written in TypeScript. There are a few ways to create a new automation client project. My favorite way is via Slack by running `@atomist generate automation`. The result of this command is a [new automation project](https://github.com/ivans-innovation-lab/my-company-automations) on my github account: https://github.com/ivans-innovation-lab/my-company-automations

### Command handlers

For the purposes of this lab we have created a generator for the command side component:

 - [`commandSideGenerator.ts`](https://github.com/ivans-innovation-lab/my-company-automations/blob/master/src/commands/generator/commandSideGenerator.ts)
 
This generator will create a new project/module based on the [seed project](https://github.com/ivans-innovation-lab/my-company-blog-domain).


I suppose you have already installed:

- [Node.js](https://nodejs.org/)
- `npm install -g @atomist/automation-client`

and you already know how to create a custom automation-client project. Please do so, it’s a lot of fun! For now, you can clone this:

```
$ git clone git@github.com:ivans-innovation-lab/my-company-automations.git
$ cd my-company-automations
$ npm install
$ npm run build
```
If this is the first time you will be running an Atomist API automation-client project
locally, you should first configure your system using the
`atomist-config` script:

```
$ `npm bin`/atomist config [SLACK_TEAM_ID]
```

To start the automation-client project, run the following command:

```
$ npm run autostart
```

## Atomist is SaaS

Atomis is more then the code, it understands the relationship between your code, your tools, your environments, and your running services and brings this information to where you live: chat. 

[Invite Atomist bot](https://atm.st/2wiDlUe) into your Slack team.

You have a new team member now, and you can ‘command’ him to do stuff for you ;).

### Create a new project:

```
@atomist generate command side API 
````
![](https://github.com/idugalic/idugalic.github.io/raw/master/img/atomist_01.png)

In reply you have to enter 'aggregate' name and the 'target' Github repository:

![](https://github.com/idugalic/idugalic.github.io/raw/master/img/atomist_02.png)

You have successfully created a new project based on your seed!

### Create a new issue:

```
@atomist create issue 
````
Push some code on the new branch:

![](https://github.com/idugalic/idugalic.github.io/raw/master/img/atomist_03.png)
Please note commit messages and CI build status in the channel.

### Raise Pull Request:
![](https://github.com/idugalic/idugalic.github.io/raw/master/img/atomist_04.png)

### Merge it to master:
![](https://github.com/idugalic/idugalic.github.io/raw/master/img/atomist_05.png)

### Create a release:
![](https://github.com/idugalic/idugalic.github.io/raw/master/img/atomist_06.png)

### Close the issue:
![](https://github.com/idugalic/idugalic.github.io/raw/master/img/atomist_07.png)


## Kudos

This is just a small part that I was able to show you. I hope that you will find out how Atomist can help your project or product.

Atomist is a true team member. It (he/she :)) helped us integrate tools like CircleCI, Github and Slack. Atomist correlate events this tools propagate into meaningful data that we can understand, and react upon.

I personally believe that tools like this will change the way we work and keep us in the development process loop. The process that we have designed in the beginning.


