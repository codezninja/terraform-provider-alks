ALKS Provider for Terraform
=========

[![Build Status](https://travis-ci.org/Cox-Automotive/terraform-provider-alks.svg?branch=master)](https://travis-ci.org/Cox-Automotive/terraform-provider-alks)

This module is used for creating IAM Roles via the ALKS API.

## Pre-Requisites

* An ALKS Admin or IAMAdmin STS [assume-role](http://docs.aws.amazon.com/STS/latest/APIReference/API_AssumeRole.html) session is needed. PowerUser access is not sufficient to create IAM roles.
    * This tool is best suited for users with an `Admin` role
    * With an `IAMAdmin|LabAdmin` role, you can create roles and attach policies, but you can't create other infrastructure.
* Works with [Terraform](https://www.terraform.io/) version `0.10.0` or newer.

## Terraform version < 0.13 Installation

* Download and install [Terraform](https://www.terraform.io/intro/getting-started/install.html)

* Download ALKS Provider binary for your platform from [Releases](https://github.com/Cox-Automotive/terraform-provider-alks/releases)

For example on macOS:

```
curl https://github.com/Cox-Automotive/terraform-provider-alks/releases/download/1.5.0/release/terraform-provider-alks_1.5.0_darwin_amd64.zip -O -J -L | unzip
```

* Configure Terraform to use this plugin by placing the binary in `.terraform.d/plugins/` on MacOS/Linux or `terraform.d\plugins\` in your user's "Application Data" directory on Windows.

* Note: If you've used a previous version of the ALKS provider and created a `.terraformrc` file in your home directory you'll want to remove it prior to updating.

## Terraform version >= 0.13 Terraform Installation

* Download and install [Terraform](https://www.terraform.io/intro/getting-started/install.html)

* Download ALKS Provider binary for your platform from [Releases](https://github.com/Cox-Automotive/terraform-provider-alks/releases)
  
For example on macOS:

```
curl https://github.com/Cox-Automotive/terraform-provider-alks/releases/download/1.5.0/release/terraform-provider-alks_1.5.0_darwin_amd64.zip -O -J -L | unzip
```

* Go into the Terraform plugins path; `.terraform.d/plugins/` on MacOS/Linux or `terraform.d\plugins\` in your user's "Application Data" directory on Windows.

* Create the following directories: `coxautoinc.com/engineering-enablement/alks/1.5.0/<OS>_<ARCH>` and put the binary into the `<OS>_<ARCH>/` directory.
  * Note: This `<OS>_<ARCH>` will vary depending on your system. For example, 64-bit MacOS would be: `darwin_amd64` while 64-bit Windows 10 would be: `windows_amd64` 

* Finally, configure Terraform.
    * In your `versions.tf` or `main.tf` file you'll want to add the new ALKS provider as such:
    ```
    terraform {
        required_version = ">= 0.13"
        required_providers {
        alks = {
            source = "coxautoinc.com/engineering-enablement/alks"
            }
        }
    }
    ```

* Note: If you've previously installed our provider and it is stored in your remote state, you may need to run the [`replace-provider` command](https://www.terraform.io/docs/commands/state/replace-provider.html).

## Usage

### Authentication

The ALKS provider offers a flexible means of providing authentication credentials for creating roles. The following methods are supported, in this order, and explained below:

#### Static Credentials

Static credentials can be provided via an `access_key`, `secret_key` and `token` in-line in the ALKS provider block.  This method is generally not recommended, since the credentials could accidentally be committed or shared.

```tf
provider "alks" {
    url        = "https://alks.foo.com/rest"
    version    = ">= 1.4.5, < 2.0.0"
    access_key = "accesskey"
    secret_key = "secretkey"
    token      = "sessiontoken"
}
```

#### Environment variables

You can provide your credentials via the `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY` and `AWS_SESSION_TOKEN` environment variables. If you need to pass multiple AWS credentials (when using a combination of Roles, like `PowerUser` and `IAMAdmin`) you can use the `ALKS_` prefix in place of `AWS_` (ex: `ALKS_ACCESS_KEY_ID`) as these are prioritized over the `AWS_` prefixed environment varaibles for the provider.

```tf
provider "alks" {
    url     = "https://alks.foo.com/rest"
    version = ">= 1.4.5, < 2.0.0"
}
```

```console
$ alks sessions open -i
$ export AWS_ACCESS_KEY_ID="accesskey"
$ export AWS_SECRET_ACCESS_KEY="secretkey"
$ export AWS_SESSION_TOKEN="sessiontoken"
$ terraform plan
```

#### Shared Credentials file

You can use an AWS credentials file to specify your credentials. The default location is `$HOME/.aws/credentials` on Linux and OSX, or `"%USERPROFILE%\.aws\credentials"` for Windows users. If we fail to detect credentials inline, or in the environment, Terraform will check this location last. You can optionally specify a different location in the configuration via the `shared_credentials_file` attribute, or via the environment with the `AWS_SHARED_CREDENTIALS_FILE` variable. This method also supports a profile configuration and matching `AWS_PROFILE` environment variable.

```tf
provider "alks" {
    url                     = "https://alks.foo.com/rest"
    version                 = ">= 1.4.5, < 2.0.0"
    shared_credentials_file = "/Users/brianantonelli/.aws/credentials"
    profile                 = "foo"
}
```

#### Machine Identities

You can use a role created with ALKS with the `enable_alks_access` flag set to `true` to authenticate requests against ALKS.

In order to do this, ALKS must be called from within AWS using STS credentials from an instance profile associated with the role with `enable_alks_access` set.  This also works from Lambda functions in the same way.

The STS credentials are used and provided in the same way that the AWS CLI uses the credentials, so there is nothing special you have to do to use Machine Identities.

Your ALKS provider block can look just like this:

```tf
provider "alks" {
    url     = "https://alks.foo.com/rest"
    version = ">= 1.4.5, < 2.0.0"
}
```

Since Machine Identities work with Instance Profile Metadata directly, it can be helpful to assume another role or cross account trust.  For example:

```tf
provider "alks" {
   url     = "https://alks.foo.com/rest"
   version = ">= 1.4.5, < 2.0.0"
   assume_role {
      role_arn = "arn:aws:iam::112233445566:role/acct-managed/JenkinsPRODAccountTrust"
   }
}
```

#### Multiple Provider Configuration

You can configure multiple ALKS providers to each have their own account context. 

The initial provider must have credentials set in a default way (static, shared credentials file, environment variables, etc) before the second provider can determine whether your account/role combination are allowed. 

The second (or so) provider can then be used to generate resources for multiple accounts in one plan / apply.

Note: This only works for accounts you have access to!

```tf
# PROVIDER 1
provider "alks" {
  url = "https://alks.coxautoinc.com/rest"
}

# PROVIDER 2
provider "alks" {
  url     = "https://alks.coxautoinc.com/rest"
  account = "<account No>"
  role    = "<role>"
  alias   = "second"
}

# CREATE IAM ROLE -- PROVIDER 1
resource "alks_iamrole" "test_role" {
  name                     = "TEST-DELETE"
  type                     = "AWS CodeBuild"
  include_default_policies = false
  enable_alks_access       = true
}

# CREATE IAM ROLE -- PROVIDER 2
resource "alks_iamrole" "test_role_nonprod" {
  provider                 = alks.second
  name                     = "TEST-DELETE"
  type                     = "AWS CodeBuild"
  include_default_policies = false
  enable_alks_access       = true
}
```

### Provider Configuration

Provider Options:

* `url` - (Required) The URL to your ALKS server. Also read from `ENV.ALKS_URL`
* `access_key` - (Optional) The access key from a valid STS session.  Also read from `ENV.ALKS_ACCESS_KEY_ID` and `ENV.AWS_ACCESS_KEY_ID`.
* `secret_key` - (Optional) The secret key from a valid STS session.  Also read from `ENV.ALKS_SECRET_ACCESS_KEY` and `ENV.AWS_SECRET_ACCESS_KEY`.
* `token` - (Optional) The session token from a valid STS session.  Also read from `ENV.ALKS_SESSION_TOKEN` and `ENV.AWS_SESSION_TOKEN`.
* `shared_credentials_file ` - (Optional) The the path to the shared credentials file. Also read from `ENV.AWS_SHARED_CREDENTIALS_FILE `.
* `profile` - (Optional) This is the AWS profile name as set in the shared credentials file. Also read from `ENV.AWS_PROFILE`.
* `assume_role` - (Optional) This is the role information to assume before making calling ALKS.  This feature works the same as the `assume_role` feature of the [AWS Terraform Provider](https://www.terraform.io/docs/providers/aws/#assume-role).
    * `role_arn` - (Required) The Role ARN to assume for calling the ALKS API.
    * `session_name` - (Optional) The session name to provide to AWS when creating STS credentials.  Please see the AWS SDK documentation for more information.
    * `external_id` - (Optional) The external identifier to provide to AWS when creating STS credentials.  Please see the AWS SDK documentation for more information.
    * `policy` - (Optional) This specifies additional policy restrictions to apply to the resulting STS credentials beyond any existing inline or managed policies.  Please see the AWS SDK documentation for more information.
    
### Resource Configuration

#### `alks_iamrole`

```tf
resource "alks_iamrole" "test_role" {
    name                     = "My_Test_Role"
    type                     = "Amazon EC2"
    include_default_policies = false
    enable_alks_access       = false
}
```

| Value                      | Type     | Forces New | Value Type | Description                                                                                                                                                                                                                                                       |
| -------------------------- | -------- | ---------- | ---------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `name`                     | Required | yes        | string     | The name of the IAM role to create. This parameter allows a string of characters consisting of upper and lowercase alphanumeric characters with no spaces. You can also include any of the following characters: =,.@-. Role names are not distinguished by case. |
| `type`                     | Required | yes        | string     | The role type to use. [Available Roles](https://gist.github.com/brianantonelli/5769deff6fd8f3ff30e40b844f0b1fb4)                                                                                                                                                  |
| `include_default_policies` | Required | yes        | bool       | Whether or not the default managed policies should be attached to the role.                                                                                                                                                                                       |
| `role_added_to_ip`         | Computed | n/a        | bool       | Indicates whether or not an instance profile role was created.                                                                                                                                                                                                    |
| `arn`                      | Computed | n/a        | string     | Provides the ARN of the role that was created.                                                                                                                                                                                                                    |
| `ip_arn`                   | Computed | n/a        | string     | If `role_added_to_ip` was `true` this will provide the ARN of the instance profile role.                                                                                                                                                                          |
| `enable_alks_access`       | Optional | yes        | bool       | If true, allows ALKS calls to be made by instance profiles or Lambda functions making use of this role.                                                                                                                                                           |

#### `alks_iamtrustrole`

```tf
resource "alks_iamtrustrole" "test_trust_role" {
    name                     = "My_Cross_Test_Role"
    type                     = "Cross Account"
    # type                   = "Inner Account"
    trust_arn                = "arn:aws:iam::123456789123:role/acct-managed/TestTrustRole"
    enable_alks_access       = false
}
```

| Value                | Type     | Forces New | Value Type | Description                                                                                                                                                                                                                                                       |
| -------------------- | -------- | ---------- | ---------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `name`               | Required | yes        | string     | The name of the IAM role to create. This parameter allows a string of characters consisting of upper and lowercase alphanumeric characters with no spaces. You can also include any of the following characters: =,.@-. Role names are not distinguished by case. |
| `type`               | Required | yes        | string     | The role type to use `Cross Account` or `Inner Account`.                                                                                                                                                                                                          |
| `trust_arn`          | Required | yes        | string     | account role arn to trust.                                                                                                                                                                                                                                        |
| `role_added_to_ip`   | Computed | n/a        | bool       | Indicates whether or not an instance profile role was created.                                                                                                                                                                                                    |
| `arn`                | Computed | n/a        | string     | Provides the ARN of the role that was created.                                                                                                                                                                                                                    |
| `ip_arn`             | Computed | n/a        | string     | If `role_added_to_ip` was `true` this will provide the ARN of the instance profile role.                                                                                                                                                                          |
| `enable_alks_access` | Optional | yes        | bool       | If `true`, allows ALKS calls to be made by instance profiles or Lambda functions making use of this role.                                                                                                                                                         |

### `alks_ltk`

```tf
resource "alks_ltk" "test_ltk_user" {
   iam_username             = "My_LTK_User_Name"
}
```

| Value          | Type     | Forces New | Value Type | Description                                                                                                                                                                                                                                                       |
| -------------- | -------- | ---------- | ---------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `iam_username` | Required | yes        | string     | The name of the IAM user to create. This parameter allows a string of characters consisting of upper and lowercase alphanumeric characters with no spaces. You can also include any of the following characters: =,.@-. User names are not distinguished by case. |
| `iam_user_arn` | Computed | n/a        | string     | The ARN associated with the LTK user.                                                                                                                                                                                                                             |
| `access_key`   | Computed | n/a        | string     | Generated access key for the LTK user. Note: This is saved in the state file, so please be aware of this.                                                                                                                                                         |
| `secret_key`   | Computed | n/a        | string     | Generated secret key for the LTK user. Note: This is saved in the state file, so please be aware of this.                                                                                                                                                         |

### Data Source Configuration
#### `alks_keys`
```tf
data "alks_keys" "account_keys" {
   providers: alks.my_alias
}
```

| Value          | Type     | Forces New | Value Type | Description                                                                                                                                                                                                                                                       |
| -------------- | -------- | ---------- | ---------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `access_key`   | Computed | n/a        | string     | Generated access key for the specified provider. If multiple providers, it takes the `provider` field. Otherwise uses the initial provider.                                                                                                                                                        |
| `secret_key`   | Computed | n/a        | string     | Generated secret key for the specified provider. If multiple providers, it takes the `provider` field. Otherwise uses the initial provider.                                                                                                                                                   |
| `session_token`| Computed | n/a        | string     | Generated session token for the specified provider. If multiple providers, it takes the `provider` field. Otherwise uses the initial provider.                                                                                                                                 |
| `account`   | Computed    | n/a        | string     | The account number of the returned keys.  
| `role`   | Computed       | n/a        | string     | The role from the returned keys.   

_Note: This does not take any arguments. See below._ 
- **How it works**: Whatever your default provider credentials are, will be used. If multiple providers have been configured, then one can point the data source to return keys for specific providers using `providers` field with a specific `alias`.


## Example

See [this example](examples/alks.tf) for a basic Terraform script which:

1. Creates an AWS provider and ALKS provider
2. Creates an IAM Role via the ALKS provider
3. Attaches a policy to the created role using the AWS provider

This example is intended to show how to combine a typical AWS Terraform script with the ALKS provider to automate the creation of IAM roles and other infrastructure.

## Building from Source

To build the ALKS provider, install [Go](http://www.golang.org/) (preferably version 1.14.4 or greater).

Clone this repository and `cd` into the cloned directory. All the necessary dependencies are vendored, so type `make build test` to build and test the project. If this exits with exit status `0`, then everything is working! Check your `examples` directory for an example Terraform script and the generated binary.

```bash
git clone https://github.com/Cox-Automotive/terraform-provider-alks.git
cd terraform-provider-alks
make build test
```

If you need any additional depedencies while developing, add the dependency by running `go get <dependency>` and then add it to the vendor folder by running `go mod vendor`.