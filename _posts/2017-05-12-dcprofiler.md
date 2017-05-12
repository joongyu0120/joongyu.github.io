---
title: Datacenter-wide Profiler
layout: post
---
# Datacenter-wide Profiler

## Introduction

  Samsung Electronics' internal development group *CPG Cloud Platform & Infrastructure Part* develops and operates a container-based orchestration platform called *Ceres* for IDCs around the world to create and manage in-house services. Because container services are provided and managed for various service providers, efficient placement of containers in the datacenter is very important in terms of cost and performance. However, it is very challenging to schedule containers in a large-scale infrastructure and service environment while simultaneously pursuing resource efficiency and optimum performance. In order to pursue optimal performance, it is necessary not only to quantify and schedule the resources of the hosts in the DC, but also to identify the characteristics of other workloads to be resident in the host to prevent the load from being burdened on specific resources. Also, it would be better if you could find out and deploy a better host to handle the workload. In order to do this, the scheduler must be able aware of the characteristics of the workloads that each container is trying to perform before deployment. It also needs to know the environment information such as OS type, kernel version, installed h/w information of DC hosts and other workload all of the characteristics of. In order to identify the characteristics of the workload, we performs a continuous performance analysis on all the containers present in the DC, collects the profiled data, and gains a statistical view of the workload characteristics of the container.

  In addition to the efficient use of DC resources, it is sometimes necessary to profile the containers that make up the service from the perspective of providers developing services on the cloud. In this case, generally, the service provider installs a solution for profiling in each instance and analyzes it. In case of operating a container-based service, most instances are divided into micro-services, It is not easy to install and set up a profiling solution in every container. The possibility of varying the OS or application runtime of the containers also makes it difficult to install appropriate profiling solution. It is difficult to set up a profiling environment on a cloud where containers are scattered, but it is more difficult to collect and compare among the results. CPG's *Data center-wide Profiler (DC Profiler)* was created to provide blockbox analysis without installing individual profiling solution from service developer, and to help ease the analysis of service units.

## Background ##

#### PMU (Performance Monitoring Unit) ####

The CPU provides PMU (Performance Monitoring Unit) to measure various h/w related performance information. The number of PMUs is limited to four for each CPU core. The PMU provided by the CPU allows you to measure various performance indicators. Here are some of the PMU event lists supported by Linux Perf.

- Branch-instructions
- Branch-misses
- Bus-cycles
- Cache-misses
- Cache-references
- Cpu-cycles
- Instructions
- Mem-loads
- Mem-stores
- Ref-cycles
- Stalled-cycles-frontend
- ...

#### Superscalar Architecture ####

The latest CPU is called the superscalar architecture, which can handle multiple CPU instructions in a single CPU cycle through relocation of the pipeline.

#### Instructions per cycle (IPC) ####

IPC is the average number of instructions processed per CPU cycle. In terms of performance analysis, IPC is often talked about as opposed to CPI. The higher the IPC, the more efficiently the CPU uses system resources when processing instructions.

#### Cycles Retired ####

Number of cycles that effectively processed the instructions

#### Stalled cycles in front-end ####

The number of cycles that were waiting without pushing the command from the front-end to the back-end of the CPU. Generally, this occurs when a command cache miss occurs or when decoding is complicated.

#### Stalled cycles in back-end ####

The number of cycles waiting to use resources such as memory at the CPU's back-end or to wait for long-running instructions (sqrt, log, etc.)

#### Memory load / store issues ####

Number of times memory load / store occurred

## Datacenter-wide Profiler Architecture

The diagram below shows the overall architecture of the *DC Profiler*.

![*DC Profiler* Architecture](/images/dcprofiler.png)

## Components

#### Agents

- PMU collector

 It is installed on a host basis and collects necessary H / W related performance data using PMU. In the Linux kernel, tools are provided to allow profiling through PMU in a single process, process group, or cgroup basis through perf_event. In linux, a container is generally divided into cgroup units, Collect metrics. The PMU Collector performs profiling by alternating the time sharing method (about 10 seconds) for the containers running on the host and sends the result to Kafka.
  
- In-depth profiler

 The host performs sampling-based stack profiling on a per-container basis. It runs on-demand profiling and sends the results to Hadoop when profiling is finished. It can be either of profilers, Perf or Oprofiler.

- Ftrace controller

 Controls the ftrace and tracepoint of the Linux kernel to collect host system events such as signal delivery, process comm, process fork / exit and so on. 

#### API server

API Server provides RESTful API and controls the operation of *Continuous Profiler* and *on-demand profiler* through Configuration Manager.

#### Configuration manager

In order to control the PMU collector for each host, Configuration Manager is set in DC unit. The configuration of PMU collector is set through API Server. Each PMU collector performs profiling by reflecting the change in its configuration.

#### Kafka

Kafka is distributed streaming platform building real-time streaming data pipelines that reliably get data between systems or applications. We put all collected profiled data into Kafka. 

#### Spark / Spark streaming application
- Continuous profile analyzer
 
 Based on the information collected from PMU Collector and system event collector, profile is processed and stored in HDFS database.

- On-demand profile analyzer
 
 Analyze the results of Raw profiles obtained from the in-depth profiler and graph them in a form that can be analyzed by the human.
    

#### Presto

Distributed SQL engine
  
#### Zeppelin
   
Zeppelin is used as a dash board, and interacts with Presto to provide an interactive SQL Query to the user and display it graphically.

## Architecture Overview

  *DC Profiler* provides two types of profiler.

- Continuous Profiler
- On-demand Profiler

The *continuous profiler* continuously uses the PMU to profile the state of all containers running in the DC, determining the workload characteristics of the container and measuring the load of the host resource. The *on-demand profiler* performs in-depth profiling on specific containers when requested by the user. The profiling performed at this time can be regarded as sampling profiling through Perf or Oprofiler. The results obtained through each profiler are refined and structured by spark streaming and stored in the HDFS of the Hadoop cluster. *DC Profiler* is interworked with the scheduler for managing the container. Therefore, if service unit analysis is required, it is possible to grasp the list of containers constituting the service and to profile only the objects separately.
 Agents are installed on each host for continuous/on-demand profiling operation. PMU collector and ftrace controller for *continuous profiler*, and in-depth profiler for *on-demand profiler*. The results from the PMU collecotr and ftrace controllers are aggregated and structured via Spark Streaming and stored in Hadoop HDFS. The results of the in-depth profiler are stored in a raw format on the HDFS. In this way, profile results stored in Hadoop can be searched in real time in SQL or analyzed through graphical output with In-depth Analyzer.

## Continuous Profiling

### Performance Analysis based on PMU

  Performance analysis through the PMU requires some understanding of the structure of the CPU architecture and terminology. I will skip the detailed explanation of the CPU structure and go beyond the conceptual explanations necessary to understand the *DC Profiler*.

  Recent CPUs have a superscalar architecture, which means they can handle multiple instructions per CPU cycle. Normally, this is expressed as IPC (Instructions per cycle). If IPC is 4, on average, the CPU core processes four instructions on one CPU cycle. However, when you actually run the application, you can not achieve the performance of the ideal IPC provided by the CPU, because there are restrictions on the resources required to process the instruction. The inability of the CPU to perform as well as the ideal IPC means that some of the CPU cycles are wasted, and this is expressed as stalled cycles in front-end / back-end. Conversely, CPU cycles that effectively processed instructions are said to be retired cycles. If so, what exactly causes the cycle to be wasted? We can estimate this through the location where the stalled occurred. If the stall occurred in the front-end, you might suspect that the instruction cache miss occurred frequently, or if you are dealing with complex instructions that take a long time to decode. Conversely, if the stall occurred in the back-end, it may take a long time to wait for resources such as DRAM memory, or it may have taken time to wait for long processing times (sqrt, log, etc.) . The PMU is an h/w resource in the CPU core that can measure this information. Through the PMU, we can measure the number of CPU cycles, IPC, cache hit ratio, and memory bandwidth of the application. The Linux kernel provides a way to measure PMUs via perf_event.

  Because the workload distribution unit is not an application or a VM unit, we need to analyze the performance of the container unit rather than a process or a single application. Fortunately, the linux kernel supports PMU profiling on a per-cgroup basis. Containers running on Linux usually use cgroups, so we can use this to perform PMU-based performance analysis on a per-container basis. However, *DC Profiler* does not analyze only one application, but all the containers in the host are to be analyzed. Because of the reason, we have to share PMU somehow in order to profile running containers simulteneously. *DC Profiler* performs sampling approach for container-based profiling to solve this problem. For example, if there are containers A, B, and C in the host, A, B, and C are not continuously profiled at the same time, but A 10 seconds, B 10 seconds, C 10 seconds. . This may raise doubts as to not reflecting the proper characteristics of each container. However, if these profile results continue to accumulate, the characteristics of the workload of each container can be grasped. For a related study, see Analyzing hotspots in entropy-based applications on Google's [Google-wide Profiler](http://google.com).

  Based on the result of analyzing the container performance, it was possible to grasp h/w level status information on the container and host basis, and characterization could be performed on the container image executed by the container, that is, workload unit. Container performance analysis results are collected in container image units to evaluate average IPC, cache hit ratio, CPU cycles, branch misses, and memory bandwidth. Also, it classifies various host environments (CPU type, kernel version, etc.) existing in DC to judge which host environment can handle workload more efficiently. This information is passed to the scheduling feedback of the container management platform and is used as a policy for selecting the best location for processing the workload in the future.

### System Event Analysis

  *DC Profiler*'s continuous profiling collects system events based on tracepoint and ftrace of Linux kernel in addition to PMU. However, since the load on the system may be increased depending on the events to be monitored, events that do not affect the system as much as possible are selected and collected. For example, signal generation information, the life cycle of a process, and cmdline information of a process are collected. This is also stored in the database on Hadoop HDFS and can be queried on demand.

## On-demand Profiling

  Continuous profiling using the PMU provides a measure of the current status of the host and the container and the characteristics of the workload. Sometimes, however, you need to perform in-depth profiling with performance penalty(stack, system call, etc.) in development environment. Typically, in this case, the service provider must set up a solution for profiling in each container and set up the environment. This process can be very uncomfortable because the environment of the OS and the runtime of the application are different for each container.
  
  *DC Profiler* eliminates these inconveniences and makes it easy for service providers to profile target containers. Because *DC Profiler* is installed on all hosts in DC and can find the placement related information of the containers constituting the service from the container management platform being used in DC, it is possible to judge which containers to profile when analyzing service unit performance. In addition, since system level profiling is performed, it is advantageous to perform the operation regardless of the OS of the container without installing a separate profiler in the container.
  
  The information gathered from the on-demand profiling of *DC Profiler* is sent to Hadoop's storage and processed as offline batch processing. Currently, we are providing container-level analysis function through perf, and we are developing a function to provide Brendan Gregg's [Flame Graph](http://www.brendangregg.com/flamegraphs.html) on the UI.
  
## Interactive Query System

  In addition to performing profiling and gathering metrics, there is something important. It stores the collected data in an analytical form and displays it in a graphical form. In order to perform performance analysis on a group of logical containers, we collect information collected from each host by Spark Streaming, which is capable of distributed computing, and store it on Hadoop HDFS. After that, the data accumulated in Hadoop is refined once and made into a database. The data in the database is graphed by Zeppelin, and the user can interactively query the data by providing an environment in which SQL query can be easily performed.

## Summary

  In this post, we have given a brief introduction of *DC Profiler*, our platform for continuous profiling and on-demand profiling on container orchestration system. It's specialized to profile containers runnning on our host machines. We explained its features of workload characterization, measuring h/w resource contention, profiling not only a process but containers forming a service on cloud, and so on. If you want to build something similar to your system, we hope you can get some ideas from this.
