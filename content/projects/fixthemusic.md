+++
title = "Fixthemusic"
[extra]
timeframe = "2016-present"
link = "https://fixthemusic.com"
tech = ["python", "django", "react.js", "next.js", "postgres", "nginx", "nix"]
+++

## About

Fixthemusic allows people to search and book musicians and bands for their events.

## Journey of growth

I have been the sole developer for Fixthemusic since 2016, mainly on a part-time basis but at times full time between my full-time employments in other companies. Throughout that time, the company has scaled significantly, seeing more than 5x increase in enquiries per day. This presents technical challenges to keep our service consistent and free of bugs.

Here are some major milestones that I've accomplished over the years:

#### Migration into a Single Page App(2016)

The legacy codebase is a Python (Django app) with jQuery on the frontend. While that worked well, it was quite difficult to scale up in terms of the number of unique pages and elements on the site.

I decided to move the frontend code to a React.js project that allows us to work with UI components, which makes it easier to ensure design consistency, enable interactivity, and to make the project more easily maintainable. 

With Django REST Framework, the backend solely provides an API that is used to power the frontend.

#### Optimisation of Django (2017)

As we have more and more data, the backend was becoming increasingly slow and memory consumption was becoming increasingly large. This was mainly due to the ORM nature of Django models, arrising to N+1 problems. I optimised both the user-facing and admin-facing site by preloading data on the SQL layer and caching slow but rarely changing endpoints.

#### CI & Test driven development (2018)

I introduced CircleCI into the project to run Django tests as well as React UI tests (with Enzyme). With increasing daily active users, we had to be more cautious when deploying new changes to the site, and without QA personel, automated testing made sure we have a little downtime as possible.

#### Migration to Heroku (2018)

As we wanted to introduce a significant number of new features to the site, I wanted to be able to focus on UI and API development and less on DevOps. I decided that Heroku was suitable for us. To ensure consistency between dev, staging, and production environments, I deployed services as Docker containers.

#### Server side rendering (2018-2019)

The migration to a single page app (SPA) and a fully client-rendered frontend had a positive impact on the site's user experience, and it allowed me to implement new functionalities quickly. However one trade-off is longer page load time on slower devices and slower connections.

I decided to move the codebase to Next.js, which was becoming a stable SSR framework for React apps. The improvement on page load time was very noticeable on slower devices and connection, which a significant proportion of our users have. It helped us improve our PageSpeed score as well.

#### Migration to NixOS (2020)

The migration to Heroku 2 years earlier allowed us to offload DevOps work almost completely. However, as traffic grows, we had to keep increasing the size and number of Dynos, which was becoming costly (at around $500/month).

I decided to move our infrastructure back to a single server, which we can easily vertically scale in terms of RAM and CPU to accommodate for a realistic amount of growth in the next few years while making significant cost savings.

The migration to NixOS also helped me to:

- Restructure the whole project as a monorepo
- Use systemd to run different services
  - And scheduled jobs
- Provide a simple, reproducible development environment that should stand the test of time.

## Migratory Challenges

One of the biggest challenges to improving the site and keeping up with best practices is to introduce widespread changes and refactoring to the codebase without causing disruptions to the service.

I used a number of solutions to bridge old and new services in such a way that the two could work in tandem with each other. This includes routing urls to different servers (legacy and new), using feature flags, and also extensive unit and integration tests.

## Current Architecture

- REST API with Django REST Framework
- Postgres 11
- Next.js (React.js) Server rendered frontend server
- Payment gateway integration with Stripe
- Systemd scheduled services for automated emails and reminders
- Nginx
