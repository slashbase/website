---
title: "Moving past a series of experiments"
description: "This is the story of how I did a series of product experiments to figure out the good enough setup to build a tool for working with databases."
pubDate: 2023-06-08
author: Paras Waykole
---

This is the story of how I did a series of product experiments to figure out the good enough setup to build a tool for working with databases.

Quick introduction: if you are new around here, Slashbase is an open-source modern database IDE for dev/data workflows.

It was Apr 10, 2021, Saturday, the day I wrote Slashbase’s first official commit as a side project. Though I had been thinking and experimenting with the idea for long time before that. I wanted a database tool that is accessible in-browser, so I started building it for myself with the language and libraries I knew and soon a series of experiments started. Read below on how I thought about each experiment and how the product evolved over these series of experiments.

## Experiment 1: A fully self-hosted database management tool.

I started building this as an open-source tool that could be self-hosted. I launched it on GitHub and people started trying it out. It connected well to databases in cloud and was decently usable.

Soon, I started seeing problems, such as a self-hosted browser-based database client not being able to connect to local databases for developers. This was the primary workflow the tool was providing, and managing a database dev-tool on self-hosted servers for teams, considering security & scaling was an additional issue.

## Experiment 2: In-browser IDE on localhost.

I removed the user login flow from the open-source repository, added a note further saying that the repository is not for self-hosting, and only for local dev environments.

The result became a docker image and binary file that can be run locally and opens the IDE at localhost in browser and I moved the collaborative feature to a managed and hosted SaaS app for cloud databases.

People started trying it out again and they loved the quick getting started with docker. However, not everyone had docker installed and just running binary server from terminal was inconvenient.

## Experiment 3: In-terminal CLI tool.

What came next is CLI tool that lets you connect to databases and let’s you query in terminal and connects to in-browser IDE served through CDN at app.slashbase.com.

It solved the connecting to local databases issue to in-browser IDE. But the issue here was that users coming back and saying they don’t understand how to get started with the tool and they started asking for tutorials on how to use this thing, which made it seem like a hard to use tool.

## Experiment 4: Desktop App.

In next steps, I ported the web app written in Golang and React to desktop app with lightweight electron alternative wails. I was considering electron first for this and then Tauri which seemed very promising, but ultimately wails got my attention.

Wails felt like great a option to move forward as most of the project was already written in Golang. Porting to wails desktop app was easy and quick by just removing the http server and using the wails events interface to connect react app to go functions.

## Settling down on a conclusion

This finally seems like a great setup to move forward from here. Open-source Slashbase desktop IDE for local development environment and Slashbase in-browser IDE for cloud environment. And Slashbase CLI will be included as console in desktop and in-browser IDE in coming releases. UPDATE: Slashbase CLI is included as console in open-source desktop app.

This now solves the problem of working in local development environment and getting the same experience with the in-browser IDE for cloud environments.

Checkout the latest Slashbase Desktop release here: https://github.com/slashbase/slashbaseide

UPDATE: Slashbase IDE project has been archived and is no longer under development.