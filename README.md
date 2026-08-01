# Unofficial preview builds

A Terraform [provider network mirror][mirror] serving preview builds of
[`th7/terraform-provider-github`][fork], a fork of
[`integrations/terraform-provider-github`][upstream].

**This is not an official release.** These binaries are built by hand from a
fork, are not GPG signed, and are not published to the Terraform Registry. Use
the official provider unless you specifically need a change that has not landed
upstream yet.

## Available

| Version | Platforms | Contains |
| --- | --- | --- |
| `6.13.1-th7.1` | `linux_arm64` | v6.13.0 plus a fix preserving repository fields the GitHub API omits for read-only tokens |

## Use

```hcl
# ~/.terraformrc
provider_installation {
  network_mirror {
    url     = "https://th7.github.io/terraform-provider-github/mirror/"
    include = ["registry.terraform.io/th7/*"]
  }
  direct { exclude = ["registry.terraform.io/th7/*"] }
}
```

```hcl
terraform {
  required_providers {
    github = {
      source  = "th7/github"
      version = "6.13.1-th7.1"
    }
  }
}
```

The version carries a prerelease suffix, so it is never selected automatically
by a version constraint — it has to be asked for exactly.

Licensed MIT, Copyright (c) 2020 GitHub, as the upstream project is.

[mirror]: https://developer.hashicorp.com/terraform/internals/provider-network-mirror-protocol
[fork]: https://github.com/th7/terraform-provider-github
[upstream]: https://github.com/integrations/terraform-provider-github
