---
title: "#5 How did I do with my goals? Jan-Mar 2024"
date: 2024-03-16
# lastmod: 2022-07-15T15:03:31+02:00
draft: false
summary: Periodic updates. 
tags: [monthly, goals]
# weight: 2
---

I have never set publicly my goals for the future months. In my very first blog post I just said I would do some kind of report every couple of months, looking back at what went well and what I need to improve. So, I cannot give marks or stuff like that to precise goals.
For this reason, I'm going to give some opinions with marks to some topics about what happened in my work life in the last two months (and I think I will do the same in the next reports). Little spoiler: it didn't go too good.

### Work-life balance: D-
January and Febrary were terrible from work-life balance perspective. The amount of work we (me and my collegues) faced was too much. By the end of Febryuary though, we re-started to breath.

### Personal studies: B
For the reason I mentioned above, it was nearly impossible to dedicate free time to personal studies. Nevertheless, I tried to push myself and go ahead.
- I published on my LinkedIn profile just today a first document representing my personal notes about **Apache Spark**. When I started study Spark at October 2023, I noticed that most of the guide or tutorial or even books were not so clear in the explaination (understanding in depth the concepts of Driver, Cluster Manager, Spark Application is not trivial in my opinion), meaning that all these resouces are pretty similar and take many things for granted. So, I decided to put together the best information I found along the time to put in order all these concepts in my mind. Even with this method, and even now, I have some doubts about some concepts. If anyone, other than me, is reading this post and wants to see these notes, here's the link: https://mypersonal-notes.notion.site/A-Gentle-Introduction-to-Spark-fc00af2a86f142f39e228a63052a7101. I really appreciate any kind of feedbacks.

- I posted on my GitHub account 2 new repositories:
    - one is just the attempt to re-create a project I found on YouTube by "CodeWithYu" ([link here](https://www.youtube.com/watch?v=GqAcTrqKcrY&ab_channel=CodeWithYu)) that explains how to go through an end-to-end Data Engineering project putting together technologies like **Apache Kafka**, Apache Airflow, Apache Zookeper, Apache Spark, Cassandra database, all containerized with Docker. There is not much of my own in this project, but it was a first approach with a complex environment.
    - one is inspired from a YouTube video ([link here](https://www.youtube.com/watch?v=qi7uR3ItaOY&ab_channel=CodewithIrtiza)) by "Code with Irtiza" that explains how to use Apache Kafka for a simple project about Food Ordering. I extended the project with:
        - Spark Streaming section to elaborate data coming from a Kafka Topic and store them inside a Cassandra DB;
        - a frontend section where users can simulate orders;
        - a frontend section where shop workers can notify users that the order is ready;
        - both these two frontend sections communicate with the backend through Kafka.

    If you want to know more about those projects, check my GitHub account.

    I am honestly pretty satisfied with this project. The goal from now on is to study something about the theoretical aspects of **Apache Kafka** because I just used it in my code, but I don't have a clear big picture about what is it and when and where it's used. These two projects though were two great starting points.