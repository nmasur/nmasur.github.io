---
title: "Prebuilding and Running AWS Instances with NixOS, GitHub Actions, and Terraform"
date: 2024-11-14T15:58:23-07:00
tags: [ ]
draft: false
---

The typical strategy for automatically launching Linux VMs in AWS looks
something like this:

1. Use [Packer](https://www.packer.io/) to create an EC2 instance, connect to
   it via SSH, run commands to configure the machine, snapshot the EC2 as an
AMI, and then terminate the instance.
2. Use [Terraform](https://www.terraform.io/) or another deployment script to
   launch a new EC2 instance using the AMI as a base.
3. Configure [user data
   (cloud-init)](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/user-data.html)
to run initialization scripts on first boot.

This process is brittle, requires multiple steps, including launching a new EC2
instance multiple times. Instead, I like to prebuild a NixOS AMI and let
Terraform import the image into AWS and launch the machine in a single action.

# Prebuilding the Image

In my flake, I use
[nixos-generators](https://github.com/nix-community/nixos-generators) to create
an [x86-64 Amazon
image](https://github.com/nmasur/dotfiles/blob/1022a3998f06819d6b7987d312d62bb7c8bbea15/flake.nix#L347)
from my [pre-configured NixOS
configuration](https://github.com/nmasur/dotfiles/blob/1022a3998f06819d6b7987d312d62bb7c8bbea15/hosts/arrow/modules.nix).
I use `nix build` to create a `.vhd` disk image file which can be uploaded to
S3 afterwards or picked up by Terraform and pushed to S3.

# Importing the AMI

In order to generate a valid AMI, the `.vhd` file must be imported directly
from S3. With Terraform, this can be done using the
[aws_ebs_snapshot_import](https://github.com/nmasur/dotfiles/blob/1022a3998f06819d6b7987d312d62bb7c8bbea15/hosts/arrow/aws/image.tf#L66-L80)
resource.

Once the AMI is imported, you can reference it in a basic
[aws_instance](https://github.com/nmasur/dotfiles/blob/1022a3998f06819d6b7987d312d62bb7c8bbea15/hosts/arrow/aws/ec2.tf#L1-L14)
resource just like any other EC2 launch.

# Putting It All Together

Using a [GitHub Actions
workflow](https://github.com/nmasur/dotfiles/blob/1022a3998f06819d6b7987d312d62bb7c8bbea15/.github/workflows/arrow-aws.yml)
you can perform the following steps to automatically launch everything in one
swoop:

1. [Enable
   KVM](https://github.com/nmasur/dotfiles/blob/1022a3998f06819d6b7987d312d62bb7c8bbea15/.github/workflows/arrow-aws.yml#L51-L57)
in GitHub Actions in order to generate the image locally.
2. [Build the
   image](https://github.com/nmasur/dotfiles/blob/1022a3998f06819d6b7987d312d62bb7c8bbea15/.github/workflows/arrow-aws.yml#L71-L74)
with `nix build`, using a cache if available.
3. [Upload the image to
   S3](https://github.com/nmasur/dotfiles/blob/1022a3998f06819d6b7987d312d62bb7c8bbea15/.github/workflows/arrow-aws.yml#L76-L81),
which can also be done in Terraform instead. One reason to include this step
outside of Terraform is, if you want to sometimes skip building the image to
save time, Terraform doesn't have to expect the image to always be there on
disk.
4. [Run Terraform
   apply](https://github.com/nmasur/dotfiles/blob/1022a3998f06819d6b7987d312d62bb7c8bbea15/.github/workflows/arrow-aws.yml#L102-L112)
to import the image and launch the EC2 instance.
5. [Get the host IP and wait for SSH to be
   ready](https://github.com/nmasur/dotfiles/blob/1022a3998f06819d6b7987d312d62bb7c8bbea15/.github/workflows/arrow-aws.yml#L126-L140)
in order to push content that does not belong in the Nix store (i.e. secrets).
6. [Push
   secrets](https://github.com/nmasur/dotfiles/blob/1022a3998f06819d6b7987d312d62bb7c8bbea15/.github/workflows/arrow-aws.yml#L142-L154)
to the machine with SSH using the pre-generated [SSH key placed in the
config](https://github.com/nmasur/dotfiles/blob/1022a3998f06819d6b7987d312d62bb7c8bbea15/hosts/arrow/modules.nix#L19).
7. The systemd services will automatically start up, including registering
       their [domain names with
Cloudflare](https://github.com/nmasur/dotfiles/blob/1022a3998f06819d6b7987d312d62bb7c8bbea15/modules/nixos/services/cloudflare.nix#L141-L148).

The instance is now running on AWS and ready to send or receive traffic!

