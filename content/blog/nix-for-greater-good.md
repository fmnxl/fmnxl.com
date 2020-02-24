+++
title = "Use nix for greater good"
date = 2019-11-27
+++

## !! My background

For the last 4 years I've been primarily working as a Frontend developer. Frontend is not the first place
you think that would use nix, because current tools are not so bad. Starting a react project is as easy as
running `create-react-app`, installing dependencies as `npm install`.

## Motivation

We wanted to add end-to-end testing to our react native project, and to make it run on our
continuous integration (CI) process. Having worked on another react native project with E2E tests that run
for an hour, I knew we somehow had to slash that down to an acceptable level. A couple of avenues that could
make the tests run faster are:

- Run tests in parallel
- Only rebuild parts of the app that needs to be rebuilt
- Avoid re-installing dependencies (Android SDK)

The challenge to these is that CI typically run on empty environments. Docker doesn't provide the level of
cache granularity required to avoid building unnecesary parts of the app.

## Nix


