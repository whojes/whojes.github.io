---
layout: default
title: 커리어
parent: 일상
nav_order: 4
has_children: true
print_title: true
share_enable: false
permalink: life/career
---

<p style="font-family: 'Helvetica'">
## *Career*
**[Toss](https://toss.im/team) Server Developer**  
- skills
    - spring boot (2.x.x, webmvc, webflux)
    - mongodb, redis, kafka, mysql(5.7, shardingsphere)
    - k8s, nexus, spring-cloud, api-gateway
    - elk, grafana, impala(hue), pinpoint
    
### SRE Team: 2023.07 ~ 
- works
    - redis key scanner: find heavy(use huge amount of memory or have no ttl) keys automatically
    - nodejs ssr server performance tuning
        - linux perf, node prof, ...
    - find unnecessarily resource-consuming services and fix them 
        - bcrypt, duplicate db/redis/api call
        - wrong usage of libraries
        - unnecessary double (de)serialization
        - find un-bulkheaded between microservices and fix them
        - find blocking codes(such as cpu-intensive encrypting) in eventloop and removing them
        - resource optimization for service peak
    - use cdn for services

### Server Platform Team: 2020.03 ~ 2023.06
- projects
    - unified messaging system (push/inbox/sms/alimtalk/email)
        - massive push sending system for toss and toss-affiliates
    - ab testing tools
    - web socket servers (used for chats, websocket push)
    - achievements server (badge system & services for churn users)
    - common libraries
    - other servers (chat logic server, inbox merging server, user journey server, ...)
    - toss securities platform build
        - unified messaging system, ab testing tools (self-hosted)
        - nexus, gradle, logging, grafana, logstash for self-hosted services
    - toss bank platform build
        - unified messaging system (infra-sharing)
    - toss cb server platform building
        - spring cloud config server
        - spring api gateway server
        - common libraries
            - http req/res logging
            - logging format
            - event tracing
            - common metric collect, prometheus endpoint
    - other affiliates support for messaging/ab-tests
    - sherpa: real-time tracking app-event and help users churned out of services

<br/>

### 2017.02 ~ 2020.02  
[Tmax Cloud](https://www.tmax.co.kr/) engineer

<br/>

## *Educations*

### 2015.02 ~ 2017.02  
Master of Science in Materials Science and Engineering, Seoul National Univ.  
[MFSM](http://mfsm.snu.ac.kr) soft-materials researchers  

### 2011.03 ~ 2015.02
Bachelor of Science in Materials Science and Engineering, Seoul National Univ.
<br/>

</p>