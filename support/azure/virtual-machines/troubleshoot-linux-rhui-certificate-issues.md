---
title: Troubleshoot common certificate issues in RHUI
description: Troubleshoot common Red Hat Update Infrastructure certificate issues in Azure that are caused by expired or missing TLS or SSL certificates.
author: msaenzbosupport
ms.author: msaenzbo
ms.reviewer: divargas-msft
editor: v-jsitser
ms.date: 2/9/2023
ms.service: virtual-machines
ms.subservice: redhat
---

# Troubleshoot RHUI certificate issues in Azure

This article discusses common issues in the Red Hat Update Infrastructure (RHUI) that are caused by expired or missing Transport Layer Security (TLS) or Secure Sockets Layer (SSL) certificates.

## Prerequisites

- SSH access to the corresponding [Red Hat Enterprise Linux (RHEL) Pay-As-You-Go (PAYG) virtual machine (VM) in Azure](/azure/virtual-machines/workloads/redhat/redhat-rhui).

- Root privileges.

> [!IMPORTANT]
> RHUI is intended for only pay-as-you-go images. Are you using custom or golden images (also known as "bring-your-own-subscription (BYOS)") instead? In that case, the system has to be attached to Red Hat Subscription Manager (RHSM) or Satellite in order to receive updates. For more information, see [How to register and subscribe an RHEL system to the Red Hat Customer Portal using RHSM](https://access.redhat.com/solutions/253273).

## Cause 1: RHUI client certificate is expired

The Azure RHUI certificates typically expire every two years. If you use an older RHEL VM image, such as RHEL version 7.4 (image URN: `RedHat:RHEL:7.4:7.4.2018010506`), you experience degraded connectivity to RHUI because of a now-expired TLS/SSL client certificate. For example, you might receive one of the following error messages:

> - "SSL peer rejected your certificate as expired"
>
> - "Error: Cannot retrieve repository metadata (*repomd.xml*) for repository:_... Please verify its path and try again"

You have to apply a process to avoid certificate expiration in old images or images that were created just before a certificate expiration date.

### Solution 1: Update the RHUI client package

To access RHEL repositories on pay-as-you-go systems in cloud environments, use RHUI. As a cloud provider, Azure can create and publish newer client configuration RPM versions at any time, such as for the following tasks:

- Providing access to a new repository.
- Renewing certificates.
- Making any other packaging changes.

In this situation, you have to install the new RHUI package in the system. This package has the renewed certificate. Run the [yum](https://access.redhat.com/articles/yum-cheat-sheet) command to update the RHUI package:

```bash
sudo yum update -y --disablerepo='*' --enablerepo='*microsoft*'
```

The `sudo yum update` command might also update the client certificate package (depending on your RHEL version). This is true even if the command output contains the same expired SSL certificate errors that you see for other repositories. If this update is successful, you have to restore normal connectivity to other RHUI repositories so that you can run `sudo yum update` successfully a second time.

If you receive a "404" error when you run `yum update`, try to run the following commands to refresh your yum cache:

```bash
sudo yum clean all
sudo yum makecache
```

## Cause 2: RHUI certificate is missing

The Azure Red Hat Linux VM has the RHUI Azure package already installed. However, the certificate is missing from the */etc/pki/rhui/product/* directory.

If the RHUI certificate was removed from the VM by mistake, the following error message appears when you try to install or update a package:

```output
# yum install <package-name>
 Red Hat Enterprise Linux X for x86_64 - XXXX  0.0  B/s |   0  B     00:00  
 Errors during downloading metadata for repository 'rhel-X-for-x86_64-XXXX-eus-rhui-rpms':  
 \- **Curl error (58): Problem with the local SSL certificate** for https\://rhui-3.microsoft.com/pulp/repos/content/eus/rhel8/rhui/X.X/x86_64/XXXXX/os/repodata/repomd.xml [could not load PEM client certificate, OpenSSL error error:02001002:system library:fopen:No such file or directory, (no key found, wrong pass phrase, or wrong file format?)]
```

### Solution 2: Reinstall the EUS, non-EUS, or SAP RHUI package

Reinstall the corresponding RHUI package to regenerate the missing certificates in the correct location.

All the commands in the following steps should be run by using root privileges or by specifying `sudo`:

1. Verify that the `rhui-azure` (`EUS`, `non-EUS`, or `SAP/E4S`) package is installed by using the following command:

   ```bash
   sudo rpm -qa | grep -i azure
   rhui-azure-rhelX-<>-X.X-XXX.noarch
   ```

   For more information about EUS or non-EUS RHUI packages, see the linked sections of the following articles.

   | Package type | Link |
   |---|---|
   | EUS RHUI packages | [Red Hat images connected to EUS repositories](/azure/virtual-machines/workloads/redhat/redhat-rhui#images-connected-to-eus-repositories) |
   | Non-EUS RHUI packages | [Red Hat images connected to non-EUS repositories](/azure/virtual-machines/workloads/redhat/redhat-rhui#images-connected-to-non-eus-repositories) |
   | Update Services for SAP Solutions subscriptions (SAP/E4S) RHUI packages | [Red Hat images connected to SAP/E4S repositories](/azure/virtual-machines/workloads/redhat/redhat-images#update-services-for-sap) |

2. Verify that the certificate exists:

   ```bash
   sudo ls -l /etc/pki/rhui/product/
   ```

   > [!NOTE]  
   > In this scenario, you discover that the file is missing.

3. Reinstall the corresponding `rhui-azure` package by running the `yum reinstall` command:

   ```bash
   sudo yum reinstall $(rpm -qa | grep -i rhui-azure) --disablerepo=* --enablerepo="*microsoft*"
   ```

4. If the `EUS` or `E4S` repo is installed, lock the `releasever` variable:

   ```bash
   sudo echo $(. /etc/os-release && echo $VERSION_ID) > /etc/yum/vars/releasever
   ```

5. Verify that the certificate exists by running the `ls` command again. The certificate file should be listed now:

   ```bash
   sudo ls -l /etc/pki/rhui/product/
   ```

## Cause 3: RHUI package is missing

The RHUI EUS, non-EUS, or SAP/E4S package is missing from the Red Hat VM, but the repository configuration files still exist in the */etc/yum.repos.d/* directory.

When you try to install or update a package, you receive the following error message:

```output
# yum install <package-name>  
Red Hat Enterprise Linux X for x86_64 - XXXX  0.0  B/s |   0  B     00:00  
 Errors during downloading metadata for repository 'rhel-X-for-x86_64-XXXX-XXX-rhui-rpms':  
- **Curl error (58): Problem with the local SSL certificate** for https\://rhui-3.microsoft.com/pulp/repos/content/eus/rhel8/rhui/X.X/x86_64/XXXXX/os/repodata/repomd.xml [could not load PEM client certificate, OpenSSL error error:02001002:system library:fopen:No such file or directory, (no key found, wrong pass phrase, or wrong file format?)]
```

### Solution 3: Install the EUS, non-EUS, or SAP/E4S RHUI package

Install the missing RHUI package for EUS, non-EUS, or SAP/E4S.

All the following commands should be run by using root privileges or by specifying `sudo`.

#### EUS RHUI package installation

#### [RHEL 7._x_ - EUS](#tab/rhel7-eus)

1. Run the `yum install` command to install the `rhui-azure-rhel7-eus` package:

   ```bash
   sudo yum --config='https://rhelimage.blob.core.windows.net/repositories/rhui-microsoft-azure-rhel7-eus.config' install 'rhui-azure-rhel7-eus'
   ```

2. Lock the `releasever` variable:

   ```bash
   sudo echo $(. /etc/os-release && echo $VERSION_ID) > /etc/yum/vars/releasever
   ```

3. Verify that the corresponding repositories are available and show no errors by running the `yum repolist` command:

   ```bash
   sudo yum repolist all
   ```

#### [RHEL 8._x_ - EUS](#tab/rhel8-eus)

1. Download the EUS repository configuration file by running the [wget](https://www.gnu.org/software/wget/) command:

   ```bash
   sudo wget https://rhelimage.blob.core.windows.net/repositories/rhui-microsoft-azure-rhel8-eus.config
   ```

2. Install the `rhui-azure-rhel8-eus` package by running the [dnf](https://www.linuxfordevices.com/tutorials/centos/dnf-command) installation command:

   ```bash
   sudo dnf --config=rhui-microsoft-azure-rhel8-eus.config install rhui-azure-rhel8-eus
   ```

3. Lock the `releasever` variable:

   ```bash
   sudo echo $(. /etc/os-release && echo $VERSION_ID) > /etc/yum/vars/releasever
   ```

4. Verify that the corresponding repositories are available and show no errors by running the `dnf repolist` command:

   ```bash
   sudo dnf repolist all
   ```

---

#### Non-EUS RHUI package installation

#### [RHEL 7._x_ - non-EUS](#tab/rhel7-noneus)

1. Run the `yum install` command to install the `rhui-azure` package:

   ```bash
   sudo yum --config='https://rhelimage.blob.core.windows.net/repositories/rhui-microsoft-azure-rhel7.config' install 'rhui-azure-rhel7'
   ```

2. Verify that the corresponding repositories are available and show no errors by running the `yum repolist` command:

   ```bash
   sudo yum repolist all
   ```

#### [RHEL 8._x_ - non-EUS](#tab/rhel8-noneus)

1. Download the `EUS` repository configuration file by running the `wget` command:

   ```bash
   sudo wget https://rhelimage.blob.core.windows.net/repositories/rhui-microsoft-azure-rhel8.config
   ```

2. Install the `rhui-azure-rhel8` package by running the `yum install` command:

   ```bash
   sudo yum --config=rhui-microsoft-azure-rhel8.config install rhui-azure-rhel8
   ```

3. Verify that the corresponding repositories are available and show no errors by running the `dnf repolist` command:

   ```bash
   sudo dnf repolist all
   ```

#### [RHEL 9._x_ - non-EUS](#tab/rhel9-noneus)

1. Download the `EUS` repository configuration file by running the `wget` command:

   ```bash
   sudo wget https://rhelimage.blob.core.windows.net/repositories/rhui-microsoft-azure-rhel9.config
   ```

2. Install the `rhui-azure-rhel9` package by running the `yum install` command:

   ```bash
   sudo yum --config=rhui-microsoft-azure-rhel9.config install rhui-azure-rhel9
   ```

3. Verify that the corresponding repositories are available and show no errors by running the `dnf repolist` command:

   ```bash
   sudo dnf repolist all
   ```

---

#### SAP/E4S RHUI package installation

Select the tab of an SAP image type to see the corresponding instructions.

#### [RHEL 7._x_ - RHEL-SAP-APPS](#tab/rhel7-rhel-sap-apps)

The following steps apply if the OS version is *earlier than RHEL 7.9* and the VM was created by using the `RHEL-SAP-APPS` offer image.

1. Create the */root/repo.config* file:

   ```bash
   sudo vi /root/repo.config 
   ```

2. Add the required client configuration RPM repositories for `EUS` and `SAPAPPS` to the */root/repo.config* file:

   ```console
   [rhui-microsoft-azure-rhel7-eus]
   name=Microsoft Azure RPMs for Red Hat Enterprise Linux 7 (EUS)
   baseurl=https://rhui-1.microsoft.com/pulp/repos/microsoft-azure-rhel7-eus
           https://rhui-2.microsoft.com/pulp/repos/microsoft-azure-rhel7-eus
           https://rhui-3.microsoft.com/pulp/repos/microsoft-azure-rhel7-eus
   enabled=1
   gpgcheck=1
   gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-microsoft-azure-release
   sslverify=1
   
   
   [microsoft-azure-rhel7-sap]
   name=Microsoft Azure RPMs for RHEL7 SAP
   baseurl=https://rhui-1.microsoft.com/pulp/repos/microsoft-azure-rhel7-sap
           https://rhui-2.microsoft.com/pulp/repos/microsoft-azure-rhel7-sap
           https://rhui-3.microsoft.com/pulp/repos/microsoft-azure-rhel7-sap
   enabled=1
   gpgcheck=1
   gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-microsoft-azure-release
   sslverify=1
   ```

3. Install the `sap-apps-azure-rhel7` package by running the `yum install` command:

   ```bash
   sudo yum --config /root/repo.config install sap-apps-azure-rhel7
   ```

   > [!NOTE]  
   > This also installs the `rhui-azure-rhel7-eus` package as a dependency.

4. Lock the `releasever` variable by running the following command:

   ```bash
   sudo echo $(. /etc/os-release && echo $VERSION_ID) > /etc/yum/vars/releasever
   ```

5. Verify that the corresponding repositories are available and show no errors by running the `yum repolist` command:

   ```bash
   sudo yum repolist all
   ```

#### [RHEL 7.9 - RHEL-SAP-APPS](#tab/rhel79-rhel-sap-apps)

The following steps apply if the OS version is *RHEL 7.9* and the VM was created by using the `RHEL-SAP-APPS` offer image.

1. Create the */root/repo.config* file:

   ```bash
   sudo vi /root/repo.config 
   ```

2. Add the required client configuration RPM repositories for `non-EUS` and `SAPAPPS` to the */root/repo.config* file:

   ```console
   [rhui-microsoft-azure-rhel7]
   name=Microsoft Azure RPMs for Red Hat Enterprise Linux 7
   baseurl=https://rhui-1.microsoft.com/pulp/repos/microsoft-azure-rhel7
           https://rhui-2.microsoft.com/pulp/repos/microsoft-azure-rhel7
           https://rhui-3.microsoft.com/pulp/repos/microsoft-azure-rhel7
   enabled=1
   gpgcheck=1
   gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-microsoft-azure-release
   sslverify=1
   
   [rhui-microsoft-azure-rhel7-base-sap-apps] 
   name=Microsoft Azure RPMs for Red Hat Enterprise Linux 7 (rhel7-base-sap-apps) 
   baseurl=https://rhui-1.microsoft.com/pulp/repos/microsoft-azure-rhel7-base-sap-apps 
           https://rhui-2.microsoft.com/pulp/repos/microsoft-azure-rhel7-base-sap-apps 
           https://rhui-3.microsoft.com/pulp/repos/microsoft-azure-rhel7-base-sap-apps 
   enabled=1 
   gpgcheck=1 
   gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-microsoft-azure-release 
   sslverify=1 
   ```

3. Install the `rhui-azure-rhel7-base-sap-apps` package by running the `yum install` command:

   ```bash
   sudo yum --config /root/repo.config install rhui-azure-rhel7-base-sap-apps
   ```

   > [!NOTE]  
   > This also installs the `rhui-azure-rhel7` package as a dependency.

4. Verify that the corresponding repositories are available and show no errors by running the `yum repolist` command:

   ```bash
   sudo yum repolist all
   ```

#### [RHEL 7._x_ - RHEL-SAP (old offer)](#tab/rhel7-RHEL-SAP-old-offer)

The following steps apply if the OS version is *earlier than RHEL 7.9* and the VM was created by using the `RHEL-SAP` offer image.

Images from the `RHEL-SAP (RHEL for SAP)` offer that were created *before* December 2019 are connected to `EUS`, `SAP`, and `SAP-HA` repositories. However, these specific images are no longer supported. Therefore, no new client package for `SAP-HA` will be published. Azure will support the `RHEL-SAP-HA` offers that can now be used.

To install the `SAP-HANA` repositories in this specific scenario, install the `E4S` repositories that are provided in the `RHEL-SAP-HA` offers by following these steps:

1. Create the */root/repo.config* file:

   ```bash
   sudo vi /root/repo.config 
   ```

2. Add the required client configuration RPM repositories for `E4S` to the */root/repo.config* file:

   ```console
   [microsoft-azure-rhel7-e4s]
   name=Microsoft Azure RPMs for Red Hat Enterprise Linux 7 Updates for SAP solutions (E4S)
   baseurl=https://rhui-1.microsoft.com/pulp/repos/microsoft-azure-rhel7-e4s
           https://rhui-2.microsoft.com/pulp/repos/microsoft-azure-rhel7-e4s
           https://rhui-3.microsoft.com/pulp/repos/microsoft-azure-rhel7-e4s
   enabled=1
   gpgcheck=1
   gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-microsoft-azure-release
   sslverify=1
   ```

3. Install the `rhui-azure-rhel7-e4s` package by running the `yum install` command:

   ```bash
   sudo yum --config /root/repo.config install rhui-azure-rhel7-e4s
   ```

4. Lock the `releasever` variable by running the following command:

   ```bash
   sudo echo $(. /etc/os-release && echo $VERSION_ID) > /etc/yum/vars/releasever
   ```

5. Verify that the corresponding repositories are available and show no errors by running the `yum repolist` command:

   ```bash
   sudo yum repolist all
   ```

#### [RHEL 7._x_ - RHEL-SAP-HA (E4S)](#tab/rhel7-rhel-sap-ha-e4s)

The following steps apply if the OS version is *earlier than RHEL 7.9* and the VM was created by using the `RHEL-SAP` or `RHEL-SAP-HA` offer image.

The images from the following offers that were created *after* December 2019 are connected to `E4S` repositories:

- RHEL-SAP (RHEL for SAP)
- RHEL-SAP-HA (RHEL for SAP with High Availability and Update Services)

1. Create the */root/repo.config* file:

   ```bash
   sudo vi /root/repo.config 
   ```

2. Add the required client configuration RPM repositories for `E4S` to the */root/repo.config* file:

   ```console
   [microsoft-azure-rhel7-e4s]
   name=Microsoft Azure RPMs for Red Hat Enterprise Linux 7 Updates for SAP solutions (E4S)
   baseurl=https://rhui-1.microsoft.com/pulp/repos/microsoft-azure-rhel7-e4s
           https://rhui-2.microsoft.com/pulp/repos/microsoft-azure-rhel7-e4s
           https://rhui-3.microsoft.com/pulp/repos/microsoft-azure-rhel7-e4s
   enabled=1
   gpgcheck=1
   gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-microsoft-azure-release
   sslverify=1
   ```

3. Install the `rhui-azure-rhel7-e4s` package by running the `yum install` command:

   ```bash
   sudo yum --config /root/repo.config install rhui-azure-rhel7-e4s
   ```

4. Lock the `releasever` variable by running the following command:

   ```bash
   sudo echo $(. /etc/os-release && echo $VERSION_ID) > /etc/yum/vars/releasever
   ```

5. Verify that the corresponding repositories are available and show no errors by running the `yum repolist` command:

   ```bash
   sudo yum repolist all
   ```

#### [RHEL 7.9 - RHEL-SAP-HA](#tab/rhe79-rhel-sap-ha)

The following steps apply if the OS version is *RHEL 7.9* and the VM was created by using the `RHEL-SAP-HA` offer image.

1. Create the */root/repo.config* file:

   ```bash
   sudo vi /root/repo.config 
   ```

2. Add the required client configuration RPM repositories for `non-EUS` and `microsoft-azure-rhel7-base-sap-ha` to the */root/repo.config* file:

   ```console
   [rhui-microsoft-azure-rhel7]
   name=Microsoft Azure RPMs for Red Hat Enterprise Linux 7
   baseurl=https://rhui-1.microsoft.com/pulp/repos/microsoft-azure-rhel7
           https://rhui-2.microsoft.com/pulp/repos/microsoft-azure-rhel7
           https://rhui-3.microsoft.com/pulp/repos/microsoft-azure-rhel7
   enabled=1
   gpgcheck=1
   gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-microsoft-azure-release
   sslverify=1
   
   [microsoft-azure-rhel7-base-sap-ha] 
   name=Microsoft Azure RPMs for Red Hat Enterprise Linux 7 (rhel7-base-sap-ha) 
   baseurl=https://rhui-1.microsoft.com/pulp/repos/microsoft-azure-rhel7-base-sap-ha 
           https://rhui-2.microsoft.com/pulp/repos/microsoft-azure-rhel7-base-sap-ha 
           https://rhui-3.microsoft.com/pulp/repos/microsoft-azure-rhel7-base-sap-ha 
   enabled=1 
   gpgcheck=1 
   gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-microsoft-azure-release 
   sslverify=1
   ```

3. Install the `rhui-azure-rhel7-base-sap-ha` package by running the `yum install` command:

   ```bash
   sudo yum --config /root/repo.config install rhui-azure-rhel7-base-sap-ha
   ```

   > [!NOTE]  
   > This also installs the `rhui-azure-rhel7` package as a dependency.

4. Verify that the corresponding repositories are available and show no errors by running the `yum repolist` command:

   ```bash
   sudo yum repolist all
   ```

#### [RHEL 8._x_ - RHEL-SAP-APPS](#tab/rhel8-rhel-sap-apps)

The following steps apply if the OS version is *earlier than the latest version available* supported by SAP for `RHEL 8.X` and the VM was created by using the `RHEL-SAP-APPS` offer image.

1. Create the */root/repo.config* file:

   ```bash
   sudo vi /root/repo.config 
   ```

2. Add the required client configuration RPM repositories for `EUS` and `SAP-APPS` to the */root/repo.config* file:

   ```console
   [microsoft-azure-rhel8-sapapps]
   baseurl=https://rhui-1.microsoft.com/pulp/repos/microsoft-azure-rhel8-sapapps
   name=Apps
   enabled=1
   gpgcheck=1
   gpgkey=https://rhelimage.blob.core.windows.net/repositories/RPM-GPG-KEY-microsoft-azure-release
   sslverify=1
   
   [microsoft-azure-rhel8-eus]
   baseurl=https://rhui-1.microsoft.com/pulp/repos/microsoft-azure-rhel8-eus
   name=EUS
   enabled=1
   gpgcheck=1
   gpgkey=https://rhelimage.blob.core.windows.net/repositories/RPM-GPG-KEY-microsoft-azure-release
   sslverify=1
   ```

3. Install the `rhui-azure-rhel8-sapapps` package by running the `dnf install` command:

   ```bash
   sudo dnf --config /root/repo.config install rhui-azure-rhel8-sapapps
   ```

   > [!NOTE]  
   > This also installs the `rhui-azure-rhel8-eus` package as a dependency.

4. Lock the `releasever` variable by running the following command:

   ```bash
   sudo echo $(. /etc/os-release && echo $VERSION_ID) > /etc/yum/vars/releaseverv
   ```

5. Verify that the corresponding repositories are available and show no errors by running the `dnf repolist` command:

   ```bash
   sudo dnf repolist all
   ```

#### [RHEL 8._x_ - RHEL-SAP-HA (E4S)](#tab/rhel8-rhel-sap-ha-e4s)

The following steps apply if the OS version is *earlier than the latest version available* supported by SAP for `RHEL 8.X` and the VM was created by using the `RHEL-SAP-HA` offer image.

1. Create the */root/repo.config* file:

   ```bash
   sudo vi /root/repo.config 
   ```

2. Add the required client configuration RPM repositories for `SAP HANA` to the */root/repo.config* file:

   ```console
   [microsoft-azure-rhel8-sap-ha]
   name=Microsoft Azure RPMs for RHEL8 High Availability SAP Solutions
   baseurl=https://rhui-1.microsoft.com/pulp/repos/microsoft-azure-rhel8-sap-ha
           https://rhui-2.microsoft.com/pulp/repos/microsoft-azure-rhel8-sap-ha
           https://rhui-3.microsoft.com/pulp/repos/microsoft-azure-rhel8-sap-ha
   enabled=1
   gpgcheck=1
   gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-microsoft-azure-release
   ```

3. Install the `rhui-azure-rhel8-sap-ha` package by running the `dnf install` command:

   ```bash
   sudo dnf --config /root/repo.config install rhui-azure-rhel8-sap-ha
   ```

4. Lock the `releasever` variable by running the following command:

   ```bash
   sudo echo $(. /etc/os-release && echo $VERSION_ID) > /etc/yum/vars/releaseverv
   ```

5. Verify that the corresponding repositories are available and show no errors by running the `dnf repolist` command:

   ```bash
   sudo dnf repolist all
   ```

---

## Cause 4: SSL CA certificate is missing

The *ca-bundle.crt* certificate file was manually deleted, was corrupted, or is outdated.

You might receive an error message that resembles the following output when you try to run `yum` commands:

```output
# yum repolist  
 Loaded plugins: langpacks, product-id, search-disabled-repos  
 rhui-rhel-X-server-dotnet-rhui FAILED  
 https\://rhui-3.microsoft.com/pulp/repos//content/dist/rhel/rhui/server/X/XServer/x86_64/dotnet/1/os/repodata/70b2edf9a115dffa42d4dd66ba77e77bc3cad45d1143ed02df72ea58c92b59b5-primary.sqlite.bz2: [Errno 14] **curl#77 - "Problem with the SSL CA cert (path? access rights?)"**  
 Trying other mirror.
```

### Solution 4: Update or reinstall the CA certificates package

1. Download the latest *ca-certificates-XXXX.X.XX-XX.elX_X.noarch.rpm* package from another VM that has repository access and the same Red Hat version and release. Then, copy the package to the affected VM:

   ```bash
   sudo yumdownloader ca-certificates
   sudo scp ca-certificates-XXXX.X.XX-XX.elX_X.noarch.rpm <user-name>@<affected-VM-IP-address>:/tmp
   ```

   > [!NOTE]  
   > Make sure that you replace the corresponding user and IP address placeholders. Also, make sure that you replace the package name, *ca-certificates-XXXX.X.XX-XX.elX_X.noarch.rpm*, accordingly.

2. Update, install, or reinstall the `ca-certificate` package after it's copied to the affected VM:

   1. Check whether the package is already installed:

      ```bash
      sudo rpm -qa | grep "ca-certificates"
      ```

      - If the package is missing, install it by running the `yum install` command:

        ```bash
        sudo yum install ca-certificates-*.noarch.rpm --disablerepo=*
        ```

      - If the package is still installed, run the `yum reinstall` command to reinstall it:

        ```bash
        sudo yum reinstall ca-certificates-*.noarch.rpm --disablerepo=*
        ```

   1. To regenerate or update the corresponding certificates, run the [update-ca-trust](https://www.systutorials.com/docs/linux/man/8-update-ca-trust/) command:

      ```bash
      sudo update-ca-trust
      ```

[!INCLUDE [Third-party disclaimer](../../includes/third-party-disclaimer.md)]

[!INCLUDE [Azure Help Support](../../includes/azure-help-support.md)]
