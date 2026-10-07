---
title: "OCI Cloud Security Assessment Tool"
date: 2026-08-10
description: "OCI security assessment tool that inventories cloud resources and detects common misconfigurations such as public storage, weak IAM, open network rules, and missing encryption."
tags:
  - Oracle
  - IAM
image: /images/xsskernel/Orac.jpg
---

## Introduction

Most cloud breaches don't start with a clever exploit. They start with something simple: a public bucket, a user without MFA, a network rule left open. These mistakes are easy to make and even easier to miss.

What makes them fixable is that everything needed to find them is already behind an API. With read-only credentials, you can inventory a whole tenancy without touching a single resource, and check each one against a simple rule.

This post is about that. I'm using Oracle Cloud (OCI), and the result is a Python tool that scans a tenancy and flags common misconfigurations like public storage, weak IAM, open network rules, and missing encryption.

