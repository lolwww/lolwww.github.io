---
layout: post
title: How to use Juju with Microsoft Azure
---

Based on the [Juju documentation](https://juju.is/docs/juju/microsoft-azure) and personal experience.

As Juju documentation describes, there are 3 ways to make Juju work with Azure
(for it to be able create, delete, modify resources on Azure):

1. pre-created [Managed Identity](https://learn.microsoft.com/en-us/entra/identity/managed-identities-azure-resources/overview)  with managed-identity credentials (recommended by Juju team)

2. pre-created Managed Identity + service-principal-secret credentials (arguably most commonly used)

3. service-principal-secret credentials, Managed Identity created on bootstrap. (similar to #2 in the end, but not exactly during the process).

In this guide I will try to attempt to document all 3 ways and see which one works better.

For Options 1 and 2 both we will be required to create Managed Identity first, so let's start with this part.

**Managed Identity creation**


