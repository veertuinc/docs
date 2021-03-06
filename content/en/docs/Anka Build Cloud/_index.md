---
title: "Anka Build Cloud"
linkTitle: "Anka Build Cloud"
weight: 4
description: >
  Using the Anka Build Cloud to orchestrate Anka macOS VMs and VM Template/Tag storage
---

## What is the Anka Build Cloud?

Docker and DockerHub revolutionized the way developers could build and test their software. However, Docker does not at the time of writing this support macOS. This is why we've created the Anka Build Cloud. The Anka Build Cloud is a suite of software which allows you to manage and store [Anka VM Templates and Tags]({{< relref "docs/Getting Started/creating-your-first-vm.md#anka-build-license--cloud-understanding-vm-templates-tags-and-disk-usage" >}}) in a central repository, orchestrate on-demand (or persistent) macOS VMs for your CI/CD (or developers), and visualize usage or logs. It consists of:

### Anka Virtualization Nodes

These Nodes have our [Virtualization software]({{< relref "docs/Anka Virtualization/_index.md" >}}) installed to run the VMs that you launch from the Controller (or through your CI/CD software).

### Anka Controller

This is a web UI and REST API which helps manage VM Instances and [VM Templates and Tags]({{< relref "docs/Getting Started/creating-your-first-vm.md#anka-build-license--cloud-understanding-vm-templates-tags-and-disk-usage" >}}).

### Anka Registry

This is a repository for your Anka [VM Templates and Tags]({{< relref "docs/Getting Started/creating-your-first-vm.md#anka-build-license--cloud-understanding-vm-templates-tags-and-disk-usage" >}}).

---

![High level architechture](/images/anka-build/high-level.png)

---

> Several of our [CI/CD Plugins and Integrations]({{< relref "docs/CI Plugins and Integrations/_index.md" >}}) require the Controller REST API.