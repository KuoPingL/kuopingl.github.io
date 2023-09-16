---
layout: post
title:  "DevOps : An Introduction to CI/CD"
date:   2022-08-08 16:15:07 +0800
categories: [devops, cicd]
---

## Background
Whenever we are developing an application, we always need to go through tedious routines involving **develop**, **test**, **build** and **deploy**.

Often time, some of the steps need to be performed multiple times before the final release happens, which costs time and work.

Wouldn't it be nice to have someone help us with these work lowering the cost of development ?

That's where **CI/CD** comes in to play.

**CI/CD** is a term that has been appearly all over the web for a couple of years.

They are practices that help developers to automatically **build**, **test** and **release** projects, reducing a great amount of work during development lifecycle.

But how do we implement it ?

In this article, we will dive into the world of **CI/CD** and see how it helps us during development.

## What is CI/CD ?

The term **CI/CD** is a combination of two seperate terms **CI** and **CD**. Each plays a different role during developemnt lifecycle.

><br>
>
>**CI**, an acronym for **Continuous Integration**
><br/>

It is used to help **developers** in making sure the **updated version** of the application is **regularly built, tested and merged to back to the main branch**.

><br>
>
>**CD** on the other hand, has two terms, **Continous Deployment** and **Continous Delivery**, that can be used interchangably, but represent the range of lifecycle that automation is used.
><br/>

**Continous Delivery** is an extension to **CI**, where it automatically **deploys all changes to a testing and / or production environment after the build stage**.

while ...

**Continous Deployment** takes a step further, **every change that pass all stages will be released to customers**. 

><br>

## Tools
There are tons of tools out there that can help us perform CI/CD. But none of them

## GitLab

<center>
<img src = "/images/posts/jekyll/cicd/gitlab_cicd.png" style = "width:90%"/>
<br>
GitLab CI/CD Flow
</center>

<br>




## References
1. Red Hat: [What is CI/CD?](https://www.redhat.com/en/topics/devops/what-is-ci-cd)
2. Atlassian : [Continuous integration vs. delivery vs. deployment](https://www.atlassian.com/continuous-delivery/principles/continuous-integration-vs-delivery-vs-deployment)
3. [DevOps漫谈之一：DevOps、CI、CD都是什么鬼？](https://blog.jjonline.cn/linux/238.html)
4. GitLab : [CI/CD Concepts](https://docs.gitlab.com/ee/ci/introduction/index.html)



<br><br><br><br><br><br><br><br><br><br><br>
