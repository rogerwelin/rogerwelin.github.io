---
layout: post
comments: true
title:  "Migrate and version control your database with flyway"
date:   2017-08-15 11:43:43
categories: flyway
---

## Introduction
We've come quite a long way to ensure that every commit to a application is built, deployed and tested in an efficient manner, ensuring that the code is deemed production-worthy or not. So how about the database? Put quite frankly it really doesn't matter if your code is top-notch, if your underlying data storage crumbles due to an inefficient process that can introduce errors in the schema definition. So what is a common way to introduce changes to a database? Developer writes a SQL-script containing DDL/DML changes, does a hand-over to an DBA (that has little knowledge ...) to run the script in the production database. Unfortunately this is not a ideal process for a number of reasons:  
* If the databases in the different environments are identical there's a high risk that something that worked in staging will not work in production
* If the application deployment is dependent on schema changes and if that process fails we're stuck until the problem is fixed
* If the datbase changes and application code aren't deployed in tandem it leaves a window for failure if the application code isn't compatible with the schema

So what about the alter

## Supported Databases


## Quick Tutorial


