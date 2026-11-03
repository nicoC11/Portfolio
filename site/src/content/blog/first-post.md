---
title: 'Phase 0: setting up an AWS account the safe way'
description: 'MFA, budgets, cost alerts and CLI access before building anything.'
pubDate: 'Nov 02 2026'
heroImage: '../../assets/aws-iam.png'
---

I recently passed the AWS Certified Cloud Practitioner exam, and the best way to make that knowledge stick is to build real things. My plan is to deploy this very website on AWS, then a VPC-based infrastructure, all defined as code and documented here.

Before creating a single resource, I wanted the account itself to be safe and predictable. This post covers that "Phase 0".

## 1. Lock down the root user

The root user can do anything, including closing the account, so it gets the strictest treatment: a long unique password, MFA enabled, and no access keys. After this step I don't use it for day-to-day work.

## 2. Create an IAM admin user (and skip Identity Center)

AWS recommends IAM Identity Center for people, but my account is on the new free plan with credits. According to AWS's documentation, enabling AWS Organizations, Control Tower or IAM Identity Center on that plan makes the credits expire immediately and moves the account to the paid plan. To avoid risking that, I created a regular IAM user, put it in an `admins` group with `AdministratorAccess`, and enabled MFA on it.

I also turned on the account setting that lets IAM users see billing information, so I never need root just to check costs.

## 3. Budgets and cost alerts

Even a static website can surprise you if something is left running by mistake, so I set up three layers:

- **A $1 budget** as an early tripwire. In normal operation my spending should stay below it, so if it fires, something is wrong.
- **A $5 monthly budget** as a safety ceiling, with alerts at 80% of actual spend and 100% of forecasted spend.
- **Cost Anomaly Detection** on the default services monitor, with a low absolute threshold and daily email summaries.

One detail matters: the budgets exclude the *Credit* and *Refund* charge types. Otherwise the credits would absorb the costs and the alerts would never fire, exactly when I need them.

Budgets only alert, they don't stop spending, and the data can lag by several hours. They are a safety net, not a brake.

## 4. CLI access without long-lived keys

Instead of creating access keys, I used `aws login`, available since AWS CLI v2.32. It signs in through the browser as the IAM user and stores temporary credentials in a named profile:

```bash
aws login --profile portfolio
export AWS_PROFILE=portfolio
aws sts get-caller-identity
```

The last command is the sanity check: the returned ARN must contain my IAM user, never `root`.

## 5. Pinned tooling with mise

Infrastructure code is only reproducible if everyone runs the same tool versions. For that I use [mise](https://mise.jdx.dev/), a version manager for development tools. It plays the role that `nvm`, `pyenv` or `asdf` play for a single language, but for any tool: Terraform, Node.js, and many others.

It installs each tool in its own folder inside my home directory, so there is no `sudo` and no conflict with system packages. I can also keep several versions side by side and switch between them per project.

The key feature is the `mise.toml` file. Running this inside the repo:

```bash
mise use terraform@1.17.1
```

writes the pinned version into `mise.toml`, which I commit alongside the code. Anyone who clones the repo and runs `mise install` gets exactly the versions I used. I'm doing the same for Node.js, which the Astro site needs to build.

Why it matters for this project: Terraform's behavior can change between versions, and a plan that works on my machine should work the same way in CI. Pinning the version in the repo removes "it works on my machine" from the picture, and it also documents which tools the project needs.

## 6. The repository

The GitHub repo is public, licensed under MIT, and starts with the Terraform `.gitignore` template, so state files and `.tfvars` can never be committed by accident. Secret scanning is enabled, and I authenticate with an ed25519 SSH key instead of storing tokens.

## What I took away

- The free plan has a clock: it ends after six months or when the credits run out, so I added a calendar reminder to upgrade to the paid plan before then.
- Cost control is mostly about layers: a tripwire, a ceiling and anomaly detection each catch different problems.
- Small habits, like pinning tool versions and never committing credentials, are cheap now and painful to retrofit later.

## Next

With the site built locally, the next step is defining its infrastructure in Terraform: a private S3 bucket, CloudFront, a certificate and DNS. I'll write that up as soon as it's deployed.
