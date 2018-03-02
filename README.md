# AWS STS Role

This Ansible role allows a user to assume a given role, generating temporary security credentials that can be used to assume the role.

## Requirements

- Python 2.7
- PIP package manager (**easy_install pip**)
- Ansible 2.2 or greater (**pip install ansible**)
- AWS CLI (**pip install awscli**)

## Setup

To use this role, you must add the rola as Ansible Galaxy requirement to your Ansible playbook project.

To set this role up as an Ansible Galaxy requirement, first create a `requirements.yml` file in a subfolder called `roles` and add an entry for this role.  See the [Ansible Galaxy documentation](http://docs.ansible.com/ansible/galaxy.html#installing-multiple-roles-from-a-file) for more details.

```
# Example requirements.yml file
- src: git@github.com:Casecommons/aws-sts.git
  scm: git
  version: 0.1.0
  name: aws-sts
```

Once you have created `roles/requirements.yml`, you can install the role using the `ansible-galaxy` command line tool.

```
$ ansible-galaxy install --role-file=roles/requirements.yml --roles-path=./roles/ --force
$ git commit -a -m "Added aws-sts 0.1.0 role"
```

To update the role version, simply update the `requirements.yml` file and re-install the role as demonstrated above.

## Usage

### Inputs

The STS role relies on a top level `Sts` object that uses the following inputs:

- `Sts.Profile` (Conditional) - defines the local AWS profile that credentials should be obtained from.
- `Sts.Role` (Optional) - defines the ARN of the role to assume.  If configured, this will override the role associated with the detected/configured profile.
- `Sts.SessionName` (Optional) - a name for the temporary session token that is generated
- `Sts.Disabled` (Optional) - disables the assume role action of this role.  Useful for long running playbooks that would be affected by the duration (maximum 60 minutes) of using STS token.  You must set this flag whenever you use explicit AWS credentials in your local environment (e.g. you have configured access key, secret access key and session token as environment variables) and **you don't want to attempt to assume a role**.
- `Sts.Region` (Optional) - the target AWS region.  Alternatively you can set the AWS region using the `AWS_DEFAULT_REGION` environment variable.  If this is not in your environment, the playbook defaults to `us-west-2`.
- AWS credentials - you must configure the environment such that your credentials are available to assume the role.  For example, you can set an access key and secret key via environment variables, or configure a profile via environment variables, or rely on an EC2 instance profile if running in AWS.  A dictionary called `Sts.Credentials` is output by this module, which sets up the appropriate configuration with AWS STS token settings.

### Outputs

If the assume role operation is successful, the temporary credentials issued by AWS are registered to `Sts.Credentials`:

- `Sts.Credentials.AWS_ACCESS_KEY`
- `Sts.Credentials.AWS_ACCESS_KEY_ID`
- `Sts.Credentials.AWS_SECRET_KEY`
- `Sts.Credentials.AWS_SECRET_ACCESS_KEY`
- `Sts.Credentials.AWS_SECURITY_TOKEN`
- `Sts.Credentials.AWS_DEFAULT_REGION`

You can configure `Sts.Credentials` as the environment for subsequent plays to operate under the privileges of the assumed role:

```
AWS_DEFAULT_REGION: "{{ Sts.Credentials.AWS_DEFAULT_REGION }}"
AWS_ACCESS_KEY: "{{ Sts.Credentials.AWS_ACCESS_KEY }}"
AWS_ACCESS_KEY_ID: "{{ Sts.Credentials.AWS_ACCESS_KEY_ID }}"
AWS_SECRET_KEY: "{{ Sts.Credentials.AWS_SECRET_KEY }}"
AWS_SECRET_ACCESS_KEY: "{{ Sts.Credentials.AWS_SECRET_ACCESS_KEY` }}"
AWS_SECURITY_TOKEN: "{{ Sts.Credentials.AWS_SECURITY_TOKEN` }}"
```

You should call this role from a dedicated play, and then define your subsequent playbook tasks in separate plays.  This allows the `sts_creds` or individual `sts_session` variables to be used to configure the environment of your remaining tasks.

### Role Assumption Behavior

By default this role attempts to determine an STS role to assume using the following rules in order of precedence:

1. Use the role defined by `Sts.Role`
2. Lookup the role defined in the specified AWS profile if `Sts.Profile` is configured
3. Lookup the role defined in the specified AWS profile if the `AWS_PROFILE` environment variable is set

This role also attempts to determine your AWS region using the following rules in order of precedence:

1.  Use the value of `Sts.Region` if defined
2.  Use the value of the `AWS_DEFAULT_REGION` or `AWS_REGION` environment variables if set
3.  Use the region defined in your current AWS profile if configured
4.  Return an error if none of the above methods can determine your region

In order to assume the role, this role attempts to determine your AWS credentials using the following rules in order of precedence:

1. Use the AWS profile defined by `Sts.Profile`
2. Use the AWS profile in the specified AWS profile is the `AWS_PROFILE` environment variable is set
3. Use standard AWS environment variables if configured

## Examples

The following demonstrates how to call this role from a playbook:

```yaml
- name: Assume Role Play
  hosts: localhost
  connection: local
  gather_facts: no
  vars:
    Sts:
      Role: arn:aws:iam::123456789:role/admin
      SessionName: testAssumeRole
      Region: us-west-2
  roles:
    - aws-sts
```

The following shows the recommended play configuration to use the temporary credentials issued:

```yaml
...
...
- name: My Playbook
  hosts: localhost
  connection: local
  environment: "{{ Sts.Credentials }}"
  tasks:
   - ...
   - ...
...
...
```

## Testing

You can run the included [`test.yml`](./test.yml) playbook to test this role directly.

To test the role you need to provide the role that you want to assume as demonstrated below:

```
$ ansible-playbook test.yml -e Sts.Role=arn:aws:iam::429614120872:role/admin
PLAY [Assume Role] *************************************************************

TASK [set_fact] ****************************************************************
ok: [localhost]

TASK [checking if sts functions are sts_disabled] ******************************
skipping: [localhost]

TASK [setting empty sts_session result] ****************************************
skipping: [localhost]

TASK [setting sts_creds if legacy AWS credentials are present (e.g. for Ansible Tower)] ***
skipping: [localhost]

TASK [assume sts role] *********************************************************
ok: [localhost]

TASK [set sts facts] ***********************************************************
ok: [localhost]

TASK [set sts facts] ***********************************************************
ok: [localhost]

TASK [debug] *******************************************************************
skipping: [localhost]

TASK [debug] *******************************************************************
ok: [localhost] => {
    "msg": {
        "AWS_ACCESS_KEY": "xxxx",
        "AWS_ACCESS_KEY_ID": "xxxx",
        "AWS_DEFAULT_REGION": "ap-southeast-2",
        "AWS_SECRET_ACCESS_KEY": "xxxx",
        "AWS_SECRET_KEY": "xxxx",
        "AWS_SECURITY_TOKEN": "xxxx"
    }
}

PLAY RECAP *********************************************************************
localhost                  : ok=5    changed=0    unreachable=0    failed=0

```

## Release Notes

### Version 2.4.1

- Improve region handling

### Version 2.4.0

- Ansible 2.4 support
- Add profile detection feature

### Version 1.0

- Switch to dot notation syntax

### Version 0.1.3

- Add Licensing Information

### Version 0.1.2

- Fix incorrect sts_session references

### Version 0.1.1

- Update command to use non abbreviated options.

### Version 0.1.0

- First Release

## License

Copyright (C) 2017.  Case Commons, Inc.

This program is free software: you can redistribute it and/or modify it under
the terms of the GNU Affero General Public License as published by the Free
Software Foundation, either version 3 of the License, or (at your option) any
later version.

This program is distributed in the hope that it will be useful, but WITHOUT ANY
WARRANTY; without even the implied warranty of MERCHANTABILITY or FITNESS FOR A
PARTICULAR PURPOSE. See the GNU Affero General Public License for more details.

See www.gnu.org/licenses/agpl.html
