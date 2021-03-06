# inSONMnia

[![Build Status](https://travis-ci.org/sonm-io/insonmnia.svg?branch=master)](https://travis-ci.org/sonm-io/insonmnia)

It's an early **alpha** version of the platform for [SONM.io](https://sonm.io) project.

For now it has lots of unfinished task. The main idea is to show that such platform can be implemented and to chose a techstack for future implementation.

# What is it here?

This repository contains code for Hub, Miner and CLI.

# Where can I get it?

A docker container contained every CLI, Miner, Hub can be found on public DockerHub: [sonm/insonmnia](https://hub.docker.com/r/sonm/insonmnia/)

```bash
docker pull sonm/insonmnia
```

If you want it's easy to build all the components. You need *golang > 1.8*:

```bash
make build
```

Also there is a Dockerfile to build a container:

```bash
docker build .
```

# Roadmap

Look at milestone https://github.com/sonm-io/insonmnia/milestones

# How to run

## Hub

To start a hub it's needed to expose a couple of ports.
*10001* handles gRCP requests from CLI
*10002* is used to handle communication with miners

```bash
docker run --rm -p 10002:10002 -p 10001:10001  sonm/insonmnia sonmhub
```

## Miner

To run Miner from the container you need to pass docker socket inside and specify IP of the HUB

```bash
 docker run -d —env DOCKER_API_VERSION=1.24 —net=host -v /run:/var/run sonm/insonmnia:alpha3 sonmminer —hubaddress=<hubip:10002>
```

## CLI commands

CLI sends commands to a hub. A hub must be pointed via *--hub=<hubip:port>*. Port is usually *10001*.

### ping

Just check that a hub is reachable and alive.

```bash
sonmcli --hub <hubip:10001> ping
OK
```

### list

List shows a list of miners connected to a hub with tasks assigned to them.

**NOTE: later each miner will have a unique signed ID instead of host:port**

```bash
sonmcli --hub <hubip:port> list
Connected Miners
{
	"<minerip:port": {
		"values": [
			"2b845fcc-143a-400b-92c7-aac2867ab62f",
			"412dd411-96df-442a-a397-6a2eba9147f9"
		]
	}
}
```

### start a container

To start a container you have to pick a hub and miner connected to that hub.
You can pick a miner from output of List command. See above.

```bash
./sonmcli —hub <hubip:port> —timeout=3000s  start —image schturmfogel/sonm-q3:alpha  —miner=<minerhost:port>
```
The result would look like:
```
ID <jobid>, Endpoint [27960/tcp-><ip:port> 27960/udp-><ip:port>]
```
 + **jobid** is an unique name for the task. Later it can be used to specify a task for various operations.
 + **Endpoint** describes mapping of exposed ports (google for Docker EXPOSE) to the real ports of a miner

**NOTE**: later STUN will be used for UDP packets and LVS (ipvs) or userspace proxy (like SSH tunnel) for TCP. Miners who have a public IPv4 or can be reached via IPv6 would not need this proxy. The proxy is intended to get through NAT.

### stop a container

To stop the task just provide the *jobid*

```bash
sonmcli --hub <hubip:port> stop <jobid>
```

# How to cook a container

Dockerfile for the image should follow several requirements:
 + *ENTRYPOINT* or *CMD* or both must present
 + Network ports should be specified via *EXPOSE*

# Technical overview

Technologies we use right now:

  + *golang* is the main language. Athough golang has disadvantages, we believe that its model is good for fast start and the code is easy to understand. The simplicity leads to less errors. Also it makes easy to contribute to a project as a review process is very clean.
  + *Docker* is a heart of an isolation in our platform. We rely on security features **(It's not 100% safe!)**, metrics, ecosystem it provides. The cool thing Docker is supported by many platforms. Also Docker works a lot on a unikernel approach for container based applications, which opens a huge field for security and portability improvements.
  + *whisper* as a discovery protocol
  + Until the epoch of IPv6 begins we should bring a way to get through NAT. The solution depends on a concrete transport layer. For example, different approaches should be used for UDP (e.g. STUN) and TCP (naive userspace proxy). Each approach has its own overhead and the best fit solution depends on a task.
  + *gRPC* is an API protocol between components. It's very easy to extend, supports traffic compression, flexible auth model, supported by many language. It's becoming more and more popular as a technology for RPC.

## Hub

Hub provides public gRPC-based API. proto files can be found in proto dir.

## Miner

Miner is expected to discover a Hub using Whisper. Later the miner connects to the hub via TCP. Right now a Miner must have a *public IP address*. Hub sends orders to the miner via gRPC on top of the connection. Hub pings the miner from time to time.

