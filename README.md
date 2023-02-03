# Kubernetes clusters managed by Advanced Cluster Management and OLM

## Background

Advanced Cluster Management (ACM) has strong integration with OpenShift. However, it also has support for other Kubernetes distributions. It can offer a central point for managing cluster landscape crossing cloud, distributions and on/off-premise boundaries: OpenShift, AWS Elastic Kubernetes Services (EKS), Azure Kubernetes Service (AKS), Google Kubernetes Engine (GKS) and others.

In OpenShift the Operator Lifecycle Manager (OLM) allows the installation and management of cluster extensions and layered products through operators. A large ecosystem of operators available in [OpertorHub](https://operatorhub.io/) are distributed through OLM.

## Goals

This project is a proof of concept. It aims to demonstrate how OLM can easily get installed and maintained on non-OpenShift distributions managed by ACM. It also aims to identify any gap and pitfalls that may impact the user experience.

## Installation and configuration

The procedures to configure ACM for OLM installation and maintenance is available in [SETUP.md](./SETUP.md).

## Creation and installation of a custom catalog

A subset of the operators available on OpenShift is relevant on other Kubernetes distributions. This part of the proof of concept demonstrates the creation and deployment of a custom catalog. This custom catalog contains a single operator, the compliance operator.

The procedure is available in [CATALOG.md](./CATALOG.md).

## Findings

Findings are documented in [FINDINGS.md](./FINDINGS.md)
