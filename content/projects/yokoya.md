+++
title = "YoKOYA"
[extra]
timeframe = "2020"
link = "https://yokoya.co.uk"
tech = ["python", "django", "react.js", "nix"]
+++

## About

YoKOYA is an authentic Japanese izakaya located in Camden, London. 

## A personal element

I used to be a regular customer at YoKOYA until I moved away from London at the start of 2020. It became my favourite place to meet friends as well as for the lonely dinner and drink by myself.

When the coronavirus turned into a lockdown in London, they had to change their business model and start to offer a takeaway service.

## Development phases

It was imperative that we deploy quickly and incrementally improve.

0. Project setup (DNS, Hosting)
1. Static site with takeaway menu and contact details
2. Cart system with online payment gateway
3. Improve design and implement food picture gallery

## Design decisions

Some of the goals we had were:

- Allow the project to "live" *in perpetuum*
  - Low-cost deployment
  - Reproducible dev environment for future development
- Customisability
  - Admin UI for changing data


#### Django

Despite the availability of shiny headless CMSs, I decided to use Django for this project for a number of reasons:

- **Open source**: Lots of headless CMSs are priced
- **Admin UI**: Django already comes with a battle-tested and relatively feature-rich Admin site
- **Simplicity**: Less moving parts for easy long-term maintanance
- **Fast, enough**: For the amount of expected traffic

<!-- #### Nix

Perpetually reproducible. -->
