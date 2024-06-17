---
title: "SSH dev via VSCode with DevPod"
date: 2024-07-08 00:00:00
categories: [Dev]
tags: [dev]     # TAG names should always be lowercase
published: false
---

I followed the instructions here: [RPi Pico SDK Dev Container](https://github.com/dodancs/pico-sdk-dev-container)

This setup uses:
* DevPod for MacOS
* Visual Studio Code
* the VSCode extension [Dev Containers](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-containers) by Microsoft

It lets you ssh into the docker container on the Kube cluster and trigger the build, but I dont think thats what most people are here for. If youre on linux, you wont have this problem, and if youre on MacOS, you can just ssh into your build server and run cmake to build the examples per the pico getting started guide instructions.



Devcontainer json file below.
```json
// For format details, see https://aka.ms/devcontainer.json. For config options, see the
// README at: https://github.com/devcontainers/templates/tree/main/src/ubuntu
{
	"name": "Raspberry Pi Pico Development",
	// Or use a Dockerfile or Docker Compose file. More info: https://containers.dev/guide/dockerfile
	"image": "mcr.microsoft.com/devcontainers/base:jammy",

	// Features to add to the dev container. More info: https://containers.dev/features.
	"features": {
	},

	// Use 'forwardPorts' to make a list of ports inside the container available locally.
	// "forwardPorts": [],

	// We need some things from APT to build for the Pico.
	// From https://datasheets.raspberrypi.com/pico/getting-started-with-pico.pdf - Chapter 2: The SDK
	"onCreateCommand": "sudo apt update; sudo apt install -y cmake gcc-arm-none-eabi libnewlib-arm-none-eabi build-essential libstdc++-arm-none-eabi-newlib",
	// On creation we'll clone down the pico-sdk and the examples and we'll place them into your workspace
	// for easy search and build.
	"postCreateCommand": {
		"Clone pico-sdk": "sudo mkdir /opt/pico-container; sudo chmod a+w /opt/pico-container; git clone https://github.com/raspberrypi/pico-sdk.git /opt/pico-container/pico-sdk; cd /opt/pico-container/pico-sdk; git submodule update --init",
	},

	// The standard environment variable used by the system to find the Pico SDK.
	"remoteEnv": {
		"PICO_SDK_PATH": "/opt/pico-container/pico-sdk",
	},

	// If you're using VSCode it's nice to have these two expansion packs
	"customizations": {
		"vscode": {
			"extensions": [
				"ms-vscode.cpptools-extension-pack",
				"ms-vscode.cmake-tools",
				"raspberry-pi.raspberry-pi-pico",
			]
		}
	},

	// Uncomment to connect as root instead. More info: https://aka.ms/dev-containers-non-root.
	// "remoteUser": "root"
}

```
