---
published: true
date: '2017-08-04 15:16 -0400'
title: Programming IOS-XR with gRPC and Go
position: top
tags:
  - vagrant
  - iosxr
  - gRPC
  - Go
  - VirtualBox
  - Cisco
  - YANG
author: Nicolas Leiva
excerpt: >-
  The goal of this tutorial is to demonstrate how to program an IOS XR device
  using the gRPC framework. and the Go programming language. The objective is to
  have a single interface/connection to retrieve info from the device, apply
  configs to it, generate telemetry streams, program the RIB/FIB and so on.
---
{% include toc icon="table" title="Programming IOS-XR with gRPC and Go" %}

## Introduction
The goal of this tutorial is to demonstrate how to program an IOS XR device using the gRPC framework. For this purpose, we will use a [gRPC library for Cisco IOS XR](https://github.com/nleiva/xrgrpc) written in [Go](https://golang.org/).
The objective is to have a single interface/connection to retrieve info from the device, apply configs to it, generate telemetry streams, program the RIB/FIB and so on.

We picked [Go](https://golang.org/), due its simplicity, [readability](https://talks.golang.org/2015/simplicity-is-complicated.slide#15), portability and [concurrency](https://github.com/nleiva/xrgrpc#cli-config-multiple-routers-simultaneously-merge) primitives.

This tutorial assumes that you have gone through the XR Toolbox Series before. If you haven't checked out the earlier parts to the XR toolbox Series, then you can do so here:  

>
[XR Toolbox Series]({{ site.url }}/application-hosting/tags/#xr-toolbox)

## Prerequisites

We will use this [Vagrantfile](https://github.com/nleiva/xrgrpc/blob/master/example/definetarget4/Vagrantfile) to setup and run the topology as shown below:  

![topology](https://xrdocs.github.io/xrdocs-images/assets/images/grpc.png)

So you basically need to make sure you download the IOS-XRv image as described here: [IOS-XR Vagrant Quick Start]({{ site.url }}/application-hosting/tutorials/iosxr-vagrant-quickstart) and install [Vagrant](https://www.vagrantup.com/downloads.html) and [VirtualBox](https://www.virtualbox.org/wiki/Downloads). 

Then run `vagrant up` in the folder where you have the [Vagrantfile](https://github.com/nleiva/xrgrpc/blob/master/example/definetarget4/Vagrantfile).

Request access to the Vagrant box by filling up the form [here]({{ site.url }}/getting-started/iosxr-vagrant-beta)
{: .notice--warning}

## Configuring the router for secure gRPC connections

First login to the router (password: vagrant).

```shell
ssh -p 2223 vagrant@localhost
```

Apply the following gRPC and interface config.

```
grpc
 port 57344
 tls
 !
 address-family ipv4
!
```

```
interface GigabitEthernet0/0/0/0
 ipv4 address 192.0.2.1 255.255.255.0
 no shut
 !
```

Then copy the content of the certificate file generated in `/misc/config/grpc/`. We will need this info later on.
 
```shell
bash cat /misc/config/grpc/ems.pem
```
 
## Installing Go on the Ubuntu VM

First login to the Ubuntu VM.

```shell
vagrant ssh vm-1
```

Let's start by installing [Go](https://golang.org/) as described in the [Go Wiki](https://github.com/golang/go/wiki/Ubuntu).

```shell
sudo add-apt-repository ppa:longsleep/golang-backports
sudo apt-get update
sudo apt-get install golang-1.8-go -y
```

Now that we have installed [Go](https://golang.org/), let's create a [workspace](https://golang.org/doc/code.html#Workspaces) directory.

```shell
mkdir $HOME/go
mkdir $HOME/go/src
mkdir $HOME/go/bin
mkdir $HOME/go/pkg
```

And setup some [environment variables](https://golang.org/doc/code.html#GOPATH). You might want to write these to your [profile](http://www.theunixschool.com/2011/07/what-is-profile-file.html) file. 

```shell
export GOROOT=/usr/lib/go-1.8/
export GOPATH=$HOME/go
export PATH=$PATH:$GOPATH/bin:$GOROOT/bin
```

You can verify the installation as follows:

```shell
ubuntu@vm-1:~$ go version
go version go1.8.3 linux/amd64
ubuntu@vm-1:~$ cd $GOPATH
ubuntu@vm-1:~/go$ ls
bin  pkg  src
ubuntu@vm-1:~/go$
```

That's it! [Go](https://golang.org/) is installed in the Ubuntu VM.
{: .notice--success}  

## Getting the gRPC library for Cisco IOS XR

This is super easy with the [Go tools](https://golang.org/cmd/go/#hdr-Download_and_install_packages_and_dependencies).

```shell
go get github.com/nleiva/xrgrpc
```

That's it!. All the code and [dependencies](https://github.com/golang/dep) are now in the VM.
{: .notice--success} 

## Compiling the first example

In this example we will use the [GetConfig](https://github.com/nleiva/xrgrpc/blob/master/proto/ems/ems_grpc.proto#L10) RPC to request the config of the IOS-XRv device for the [YANG](https://github.com/YangModels/yang/tree/master/vendor/cisco/xr) paths specified in [yangpaths.json](https://github.com/nleiva/xrgrpc/blob/master/example/input/yangpaths.json).

```json
{	
    "Cisco-IOS-XR-ifmgr-cfg:interface-configurations": [null],
    "Cisco-IOS-XR-telemetry-model-driven-cfg:telemetry-model-driven": [null],
    "Cisco-IOS-XR-ipv4-bgp-cfg:bgp": [null],
    "Cisco-IOS-XR-clns-isis-cfg:isis": [null]
}
```

For this purpose we go to the `definetarget4` example folder and copy the content of certificate file we obtained previously.

```shell
cd ~/go/src/github.com/nleiva/xrgrpc/example/definetarget4
vim ems.pem 
```

Let's just compile the code ([main.go](https://github.com/nleiva/xrgrpc/tree/master/example/definetarget4/main.go)) for now.

```shell
go build
```

And now you can run the binary file created.

```shell
ubuntu@vm-1:~/go/src/github.com/nleiva/xrgrpc/example/definetarget4$ ./definetarget4
Config from 192.0.2.1:57344
 {
 "data": {
  "Cisco-IOS-XR-ifmgr-cfg:interface-configurations": {
   "interface-configuration": [
    {
     "active": "act",
     "interface-name": "MgmtEth0/RP0/CPU0/0",
     "Cisco-IOS-XR-ipv4-io-cfg:ipv4-network": {
      "addresses": {
       "dhcp": [
        null
       ]
      }
     }
    },
    {
     "active": "act",
     "interface-name": "GigabitEthernet0/0/0/0",
     "Cisco-IOS-XR-ipv4-io-cfg:ipv4-network": {
      "addresses": {
       "primary": {
        "address": "192.0.2.1",
        "netmask": "255.255.255.0"
       }
      }
     }
    }
   ]
  }
 }
}

2017/08/04 18:51:00 This process took 901.242827ms
```

## A few pointers on the code

We will document a complete walk-through in a following tutorial. Well, if you are impatient like me, you can take a look at other examples documented in the [repo](https://github.com/nleiva/xrgrpc) in the meantime.

In this example we basically did four things.

  1) Parse the [YANG path input](https://github.com/nleiva/xrgrpc/blob/master/example/definetarget4/main.go#L28). If none, the default is `../input/yangpaths.json`.

  ```go
  ypath := flag.String("ypath", "../input/yangpaths.json", "YANG path arguments")
  flag.Parse()
  ```

  2) Identify the [target](https://github.com/nleiva/xrgrpc/blob/master/example/definetarget4/main.go#L37). IP address, user credentials, cert file location and a timeout.

  ```go
  router, err := xr.BuildRouter(
      xr.WithUsername("vagrant"),
      xr.WithPassword("vagrant"),
      xr.WithHost("192.0.2.1:57344"),
      xr.WithCreds("ems.pem"),
      xr.WithTimeout(5),
  )
  ```

  3) [Connect](https://github.com/nleiva/xrgrpc/blob/master/example/definetarget4/main.go#L49) to the device. This has to be done just once for all the following RPC calls. In this example we are just making one, but this connection can be re-used to configure the device, generate a telemetry stream or program the RIB/FIB.

  ```go
  conn, ctx, err := xr.Connect(*router)
  if err != nil {
      log.Fatalf("Could not setup a client connection to %s, %v", router.Host, err)
  }
  defer conn.Close()
  ```

  4) Make the [GetConfig](https://github.com/nleiva/xrgrpc/blob/master/example/definetarget4/main.go#L60) call and print out the response.

  ```go
  output, err = xr.GetConfig(ctx, conn, string(js), id)
  if err != nil {
      log.Fatalf("Could not get the config from %s, %v", router.Host, err)
  }
  fmt.Printf("\nConfig from %s\n %s\n", router.Host, output)
  ```

This concludes this tutorial/example. Stay tuned for more!.

Some useful links below:

- **Part 2**: [Validate the intent of network config changes]({{ site.url }}/programmability/tutorials/2017-08-14-validate-the-intent-of-network-config-changes/)
- [gRPC Getting Started](https://github.com/CiscoDevNet/grpc-getting-started)
- [gRPC and GPB for Networking Engineers](https://github.com/nleiva/gmessaging)
