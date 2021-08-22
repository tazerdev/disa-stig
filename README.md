# Ansible DISA STIG roles for RHEL8

This repository contains an ansible role for auditing and remediating a RHEL8 system in accordance with the DISA STIG. It is intended to be executed both during the kickstart post-install process as well as after first boot.

Red Hat provides extensive documentation for extracting, customizing, and repackaging the installation ISO as well as using supplementary media for additional files.

[DISA STIG Document Library](https://public.cyber.mil/stigs/downloads/)

[RHEL 8 ISO Customization](https://access.redhat.com/solutions/60959)

[RHEL 8 Advanced Installation](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/performing_an_advanced_rhel_installation/index)

What's different about this STIG playbook than the many others that are already out there? Not too terribly much. One difference is that it's designed to be executed during kickstart and to be used after first-boot for auditing. None of the interesting variables have been extracted, primarily because modifications should be made in a layer higher. For instance, this playbook will set selinux to enforcing but you can revert that with a role that's run afterwards. This will be done prior to the first boot. So, in a worst case scenario, your system is configured to the stig.

## Assumptions

* Must perform installation in FIPS mode.
* Must have DISA STIG compliant file system layout.

## Generating the initial role

The DISA STIG can be exported to a CSV file. Using a tool like Microsoft Excel or LibreOffice Calc you can use a macro to create the initial playbook using the following macro:

```Excel
="- name: " & CHAR(34) & G2 & CHAR(34) & "EOL" & "  ansible.builtin.shell: |" & "EOL" & "    /bin/false" & "EOL" & "  register: " & SUBSTITUTE(F2,"-","_") & "_result" & "EOL" & "  changed_when: " & SUBSTITUTE(F2,"-","_") & "_result.rc|int > 0" & "EOL" & "  failed_when: false" & "EOL" & "  check_mode: false" & "EOL" & "  tags:" & "EOL" & "    - rhel8-disa-stig-v1r3" & "EOL" & "    - " & D2 & "EOL" & "    - " & E2 & "EOL" & "    - " & F2 & "EOL" & "    - " & B2 & "EOL" & "    - " & C2 & "EOL" & "    - checkonly" & "EOL" & "    - shell" & "EOL" & "    - notimplemented" & "EOL"
```

* Export all fields
* Remove extraneous \~\~UNCLASSIFIED\~\~ lines.
* Insert a new left-most column
* Paste in the above formula over the entire column
* Copy and paste the calculated column into main.yaml
* Search and replace EOL with a newline character (using CHAR(10) doesn't work when copying and pasting multiple cells)
* Replace "sudo" with 'sudo'

You should end up with a playbook with tasks that look like this:

```yaml
- name: "RHEL 8 must be a vendor-supported release."
  shell: ,
    /bin/false
  register: RHEL_08_010000_result
  changed_when: RHEL_08_010000_result.rc|int > 0
  failed_when: false
  check_mode: false
  tags:
    - rhel8-disa-stig-v1r3
    - SRG-OS-000480-GPOS-00227
    - SV-230221r743913_rule
    - RHEL-08-010000
    - V-230221
    - high
    - checkonly
    - shell
    - notimplemented
```

## Shell task fields explanation

The default set of options for tasks which use the shell module are configured in such a way that these tasks never fail, rather, they only 'change.' The remainder of tasks will use native ansible modules for the sake of idempotency.

- **name**: Taken directly from the DISA STIG. We're using double quotes because some descriptions contain apostrophes.

- **module**: Since we want a fully functional playbook but we don't have any controls yet defined, we're going to run /bin/false by default to indicate we're not in compliance with the STIG. We'll change the module on a case by case basis for each control we implement.

- **register**: Register the result of the shell command.

- **failed_when**: Since this playbook will also be used for auditing post-provisioning we don't want the controls to fail, thus we set it to false. We just want to report that something on the system needs to be changed, thus a changed task is necessarily a finding.

- **changed_when**: Here we set the condition when our shell command is determined to be non-compliant.

- **check_mode**: We always want to run shell modules in check mode, which is how we will audit, so this is set to false.

## Tags

The high/medium/low, SRG-*, SV-*, RHEL-*, and V-* tags are all taken from the STIG. The remaining tags are described here:

- **rhel8-disa-stig-v1r3**: This tag is used to indicate which revision this control was written against. As new DISA STIG revisions come out it's useful to review and implement the changes and them update this tag on all fields to indicate the entire playbook is compliant with the latest revision.

- **checkonly**: Until we actually implement a given control, we define it a something that's only performing a check. No modifications are being made.

- **shell**: Where approrpiate we'll use a tag to logically group controls. By default all controls use the shell module and we want to keep track of those, so we start with the shell tag.

- **notimplemented**: Until we actually implement the control, all tasks are notimplemented.

## Running the Playbook

We always want to run it in check mode first:

```Bash
ansible-pull --checkout=main --directory=/tmp --url=https://github.com/tazerdev/disa-stig.git --check --skip-tags=notimplemented
```
