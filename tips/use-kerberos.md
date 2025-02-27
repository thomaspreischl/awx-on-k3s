<!-- omit in toc -->
# Use Kerberos authentication to connect to Windows hosts

If you use Active Directory users to run job templates against Windows hosts that are domain members, Kerberos authentication is usually required for WinRM.

There is [an official documentation to use Kerberos authentication in Ansible Automation Controller](https://docs.ansible.com/automation-controller/latest/html/administration/kerberos_auth.html), but it is not suitable for AWX.

This page shows you how to use Kerberos authentication for running job templates in AWX.

<!-- omit in toc -->
## Table of Contents

- [Example environment for this guide](#example-environment-for-this-guide)
- [Procedure](#procedure)
  - [Setting up your Windows host](#setting-up-your-windows-host)
    - [Enable WinRM](#enable-winrm)
    - [Configure group or permissions for the domain user](#configure-group-or-permissions-for-the-domain-user)
  - [Setting up Kubernetes](#setting-up-kubernetes)
    - [Create `krb5.conf`](#create-krb5conf)
    - [Create ConfigMap for `krb5.conf`](#create-configmap-for-krb5conf)
  - [Setting up AWX](#setting-up-awx)
    - [Create Container Group](#create-container-group)
    - [Create Credential](#create-credential)
    - [Create Inventory](#create-inventory)
    - [Configure Job Template](#configure-job-template)
- [Testing](#testing)
- [Troubleshooting](#troubleshooting)
  - [Playbook for investigation](#playbook-for-investigation)
  - [Ensure your `krb5.conf` is mounted](#ensure-your-krb5conf-is-mounted)
  - [Ensure your KDC is accessible from the EE](#ensure-your-kdc-is-accessible-from-the-ee)
  - [Ensure `kinit` can be succeeded manually](#ensure-kinit-can-be-succeeded-manually)
  - [Common issues and workarounds](#common-issues-and-workarounds)
    - [Error creating pod](#error-creating-pod)
    - [kinit: Cannot find KDC for realm "\<DOMAINNAME\>" while getting initial credentials](#kinit-cannot-find-kdc-for-realm-domainname-while-getting-initial-credentials)
    - [kerberos: the specified credentials were rejected by the server](#kerberos-the-specified-credentials-were-rejected-by-the-server)
    - [kerberos: Access is denied. Bad HTTP response returned from server. Code 500](#kerberos-access-is-denied-bad-http-response-returned-from-server-code-500)
- [Alternative solution (not recommended)](#alternative-solution-not-recommended)

## Example environment for this guide

This is my example environment for this guide. Replace these values in this guide as appropriate for your environment.

| Key | Value |
| - | - |
| Domain | `kurokobo.internal` |
| Domain Controller | `kuro-ad01.kurokobo.internal` |
| KDC Server | `kuro-ad01.kurokobo.internal` |
| Ansible Target Host | `kuro-win01.kurokobo.internal` |
| Ansible User | `awx@kurokobo.internal` |

## Procedure

To use Kerberos authentication in your AWX, following tasks are required.

1. Setting up your Windows host
   - Enable WinRM that uses Kerberos authentication
   - Allow specific domain user to connect via WinRM
2. Setting up your Kubernetes
   - Create `krb5.conf` as ConfigMap on your Kubernetes cluster
3. Setting up your AWX
   - Create Container Group with custom pod spec that mounts `krb5.conf` to make Kerberos authentication to be used in your EE
   - Create Credential for the domain user
   - Create Inventory for the Windows hosts
   - Configure your Job Template

### Setting up your Windows host

Enable WinRM on your Windows host and allow specific domain user to connect to your host via WinRM.

#### Enable WinRM

Refer [the Ansible documentation](https://docs.ansible.com/ansible/latest/user_guide/windows_setup.html) and enable WinRM on your Windows host. Running `winrm quickconfig` on your Windows host is the simplest way, but GPO can also be used to enable WinRM.

Ensure that your WinRM Listener is enabled for HTTP. Note that HTTP is safe to use with Kerberos authentication since Kerberos has its own encryption method and all messages will be encrypted over HTTP. Therefore WinRM Listener _for HTTPS_ is not mandatory.

```powershell
> winrm enumerate winrm/config/Listener
Listener
    Address = *
    Transport = HTTP
    Port = 5985
    Hostname
    Enabled = true
    URLPrefix = wsman
    CertificateThumbprint
    ListeningOn = 127.0.0.1, ...
```

Then ensure Kerberos authentication is enabled for WinRM.

```powershell
> winrm get winrm/config/Service
Service
    RootSDDL = ...
    MaxConcurrentOperations = 4294967295
    MaxConcurrentOperationsPerUser = 1500
    EnumerationTimeoutms = 240000
    MaxConnections = 300
    MaxPacketRetrievalTimeSeconds = 120
    AllowUnencrypted = false
    Auth
        Basic = true
        Kerberos = true     👈👈👈
        Negotiate = true
        Certificate = false
        CredSSP = false
        CbtHardeningLevel = Relaxed
    DefaultPorts
        HTTP = 5985
        HTTPS = 5986
    IPv4Filter = *
    IPv6Filter = *
    EnableCompatibilityHttpListener = false
    EnableCompatibilityHttpsListener = false
    CertificateThumbprint
    AllowRemoteAccess = true
```

#### Configure group or permissions for the domain user

Additionally, you have to care about the group or permissions of the domain user that used for WinRM.

WinRM is configured by default to only allow connections by users in the local `Administrators` group. In this guide, a domain user `awx@kurokobo.internal` will be used to connect to Windows hosts via WinRM.

Therefore this user have to be joined local `Administrators`, or have permissions for `Read` and `Execute` for WinRM. Default permissions for WinRM can be changed by `winrm configSDDL default`. Refer [the Ansible documentation](https://docs.ansible.com/ansible/latest/user_guide/windows_winrm.html#non-administrator-accounts) for more detail.

### Setting up Kubernetes

Create `krb5.conf` and add it as ConfigMap to your Kubernetes cluster.

#### Create `krb5.conf`

Create new file `krb5.conf` on the host that `kubectl` for your Kubernetes cluster can be used. This file will be added as ConfigMap in your Kubernetes cluster in the later step.

There are some official documentation about `krb5.conf`:

- Ansible documentation
  - [Windows Remote Management - Configuring Host Kerberos](https://docs.ansible.com/ansible/latest/user_guide/windows_winrm.html#configuring-host-kerberos)
- Ansible Automation Controller documentation
  - [23. User Authentication with Kerberos](https://docs.ansible.com/automation-controller/latest/html/administration/kerberos_auth.html)

This is my minimal example. Note that some domain names under `[realms]` and `[domain_realm]` are capitalized.

```ini
[realms]
  KUROKOBO.INTERNAL = {
    kdc = kuro-ad01.kurokobo.internal
    admin_server = kuro-ad01.kurokobo.internal
  }

[domain_realm]
  .kurokobo.internal = KUROKOBO.INTERNAL
  kurokobo.internal = KUROKOBO.INTERNAL
```

#### Create ConfigMap for `krb5.conf`

Create new ConfigMap on your Kubernetes cluster using `krb5.conf` that you have created.

```bash
kubectl -n awx create configmap awx-kerberos-config --from-file=krb5.conf
```

This command creates new ConfigMap called `awx-kerberos-config` in the namespace `awx`. Specify the path to your `krb5.conf` as `--from-file`.

Note that the namespace has to be the name of the namespace that your EE will be launched on. If you've deployed your AWX using to my guide, the namespace is `awx`.

Ensure new ConfigMap is created on your Kubernetes cluster.

```bash
$ kubectl -n awx get configmap awx-kerberos-config -o yaml
apiVersion: v1
data:
  krb5.conf: |-
    [realms]
      KUROKOBO.INTERNAL = {
        kdc = kuro-ad01.kurokobo.internal
        admin_server = kuro-ad01.kurokobo.internal
      }

    [domain_realm]
      .kurokobo.internal = KUROKOBO.INTERNAL
      kurokobo.internal = KUROKOBO.INTERNAL
kind: ConfigMap
metadata:
  ...
  name: awx-kerberos-config
  namespace: awx
  ...
```

### Setting up AWX

Create new Container Group, Credential, and Inventory in your AWX.

#### Create Container Group

Create Container Group with custom pod spec that mounts `krb5.conf` to make Kerberos authentication to be used in your EE.

1. Open AWX UI and open `Instance Groups` under `Administration`, then press `Add` > `Add container group`.
2. Enter `Name` as you like (e.g. `kerberos`) and toggle `Customize pod specification`.
3. Put following YAML string to `Custom pod spec` and press `Save`

   ```yaml
   apiVersion: v1
   kind: Pod
   metadata:
     namespace: awx
   spec:
     serviceAccountName: default
     automountServiceAccountToken: false
     containers:
       - image: 'quay.io/ansible/awx-ee:latest'
         name: worker
         args:
           - ansible-runner
           - worker
           - '--private-data-dir=/runner'
         resources:
           requests:
             cpu: 250m
             memory: 100Mi
         volumeMounts:
           - name: awx-kerberos-volume
             mountPath: /etc/krb5.conf
             subPath: krb5.conf
     volumes:
       - name: awx-kerberos-volume
         configMap:
           name: awx-kerberos-config
   ```

This pod spec means that your ConfigMap including your `krb5.conf` will be mounted as `/etc/krb5.conf` in the EE based on this Container Group. Of course this pod spec is just a working example and you can modify this to suit your requirements. Refer [my guide for Container Group](../containergroup) for detail.

#### Create Credential

Create new Credential for your domain user. In this guide, the domain user `awx@kurokobo.internal` will be used to connect to Windows hosts via WinRM, so the Credential for `awx@kurokobo.internal` have to be created.

1. Open AWX UI and open `Credentials` under `Resources`, then press `Add`.
2. Enter `Name` as you like (e.g. `Domain User`) and select `Machine` as `Credential Type`.
3. Enter `Username` and `Password` for your domain user in `<username>@<DOMAINNAME>` format. Note that _the domain name have to be capitalized_ like `awx@KUROKOBO.INTERNAL`.

It's very important that the domain name in the `Username` have to be capitalized, since this username will be passed as-is to `kinit`. If you specify your domain name in lower-case, `kinit` will fail because `kinit` cannot find KDC for realm in lower-case.

Alternatively the name of the realm can be passed through `ansible_winrm_realm` variable, but my recommendation is specify realm in Credential as the part of `Username` in upper-case.

#### Create Inventory

Create Inventory in the standard way. The important points are as follows.

- The name of the `Host` in your Inventory have to be specified as FQDN, like `kuro-win01.kurokobo.internal`, instead of IP address. This is mandatory requirement.
- You should add following variables in the Inventory as host variables or group variables.

  ```yaml
  ---
  ansible_connection: winrm
  ansible_winrm_transport: kerberos
  ansible_port: 5985
  ```

#### Configure Job Template

Create or configure your Job Template. The important points are as follows.

- Specify Inventory that includes your Windows hosts in FQDN.
- Specify Credential that includes the username in `<username>@<DOMAINNAME>` format. The domain name in the username have to be capitalized.
- Specify Instance Group that has custom pod spec that mounts your ConfigMap as `/etc/krb5.conf`.

## Testing

You can test connectivity via WinRM using Kerberos by using `ansible.windows.win_ping` module. This is an example playbook.

```yaml
---
- name: Test Kerberos Authentication
  hosts: kuro-win01.kurokobo.internal
  gather_facts: false
  tasks:

    - name: Ensure windows host is reachable
      ansible.windows.win_ping:
```

If the `Verbosity` for the Job Template is configured `4 (Connection Debug)` and if your Kerberos authentication is successfully established, the job ends with success and the log shows that `kinit` is called to connect to Windows hosts.

```text
TASK [Ensure windows host is reachable] ****************************************
...
<kuro-win01.kurokobo.internal> ESTABLISH WINRM CONNECTION FOR USER: awx@KUROKOBO.INTERNAL on PORT 5985 TO kuro-win01.kurokobo.internal
calling kinit with pexpect for principal awx@KUROKOBO.INTERNAL     👈👈👈
...
ok: [kuro-win01.kurokobo.internal] => {
    "changed": false,
    "invocation": {
        "module_args": {
            "data": "pong"
        }
    },
    "ping": "pong"
}
```

## Troubleshooting

The Kerberos authentication including `kinit` will be invoked on every Job Template running. This means that `kinit` will be invoked in EE, so if we want to investigate Kerberos related issues, we have to dig into the EE.

### Playbook for investigation

Run this playbook as a Job on the Container Group.

```yaml
---
- name: Debug Kerberos Authentication
  hosts: localhost
  gather_facts: false
  tasks:

    - name: Ensure /etc/krb5.conf is mounted
      ansible.builtin.debug:
        msg: "{{ lookup( 'file', '/etc/krb5.conf' ) }}"

    - name: Pause for specified minutes for debugging
      ansible.builtin.pause:
        minutes: 10
```

You can dig into the EE during `ansible.builtin.pause` is working by following commands.

1. Launch the Job with the playbook above.
   - If your Job exits with `Failed` immediately, your custom pod spec for your Container Group or ConfigMap for your `krb5.conf` might be wrong.
2. Invoke `kubectl -n <namespace> get pod` on your Kubernetes cluster
3. Gather the pod name that starts with `automation-job-*`

```bash
$ kubectl -n awx get pod
NAME                                               READY   STATUS    RESTARTS   AGE
awx-postgres-0                                     1/1     Running   0          41h
awx-76445c946f-btfzz                               4/4     Running   0          41h
awx-operator-controller-manager-7594795b6b-565wm   2/2     Running   0          41h
automation-job-42-tdvs5                            1/1     Running   0          4s     👈👈👈
```

Now you can access `bash` inside the EE by `kubectl -n <namespace> exec -it <pod name> -- bash`:

```bash
$ kubectl -n awx exec -it automation-job-42-tdvs5 -- bash
bash-4.4$     👈👈👈
```

Then proceed investigation.

### Ensure your `krb5.conf` is mounted

If your Container Group and ConfigMap are configured correctly, you can get your `krb5.conf` as `/etc/krb5.conf` inside the EE.

```bash
bash-4.4$ cat /etc/krb5.conf
[realms]
  KUROKOBO.INTERNAL = {
    kdc = kuro-ad01.kurokobo.internal
    admin_server = kuro-ad01.kurokobo.internal
  }

[domain_realm]
  .kurokobo.internal = KUROKOBO.INTERNAL
  kurokobo.internal = KUROKOBO.INTERNAL
```

If your `krb5.conf` is missing, ensure your custom pod spec for Container Group and ConfigMap for your `krb5.conf` are correct.

### Ensure your KDC is accessible from the EE

Ensure your KDC server can be reachable. There is no command such as `ping` or `nslookup`, downloading and using  Busybox is helpful.

```bash
# Download busybox and make it executable
bash-4.4$ curl -o busybox https://busybox.net/downloads/binaries/1.35.0-x86_64-linux-musl/busybox
bash-4.4$ chmod +x busybox
```

Then test name resolution and network reachability.

```bash
# Ensure your domain name can be resolved
bash-4.4$ ./busybox nslookup kurokobo.internal
Server:         10.43.0.10
Address:        10.43.0.10:53

Name:   kurokobo.internal
Address: ...
```

```bash
# Ensure hostname of your KDC can be resolved
bash-4.4$ ./busybox nslookup kuro-ad01.kurokobo.internal
Server:         10.43.0.10
Address:        10.43.0.10:53

Name:   kuro-ad01.kurokobo.internal
Address: ...
```

```bash
# Ensure the port 88 on your KDC can be opened
bash-4.4$ ./busybox nc -v -w 1 kuro-ad01.kurokobo.internal 88
kuro-ad01.kurokobo.internal (...:88) open
```

### Ensure `kinit` can be succeeded manually

You can test Kerberos authentication by using `kinit` manually inside the EE.

```bash
# Ensure that there is no error while passing <username>@<DOMAINNAME> and password
# Note that the domain name for kinit have to be capitalized
bash-4.4$ kinit awx@KUROKOBO.INTERNAL
Password for awx@KUROKOBO.INTERNAL:
```

```bash
# Ensure new ticket has been issued after kinit
bash-4.4$ klist                      
Ticket cache: FILE:/tmp/krb5cc_1000
Default principal: awx@KUROKOBO.INTERNAL

Valid starting     Expires            Service principal
07/02/22 12:32:28  07/02/22 22:32:28  krbtgt/KUROKOBO.INTERNAL@KUROKOBO.INTERNAL
        renew until 07/03/22 12:32:21
```

### Common issues and workarounds

Some common issues during this guide and workaround for those errors.

The ["Troubleshooting Kerberos" section in Ansible documentation](https://docs.ansible.com/ansible/latest/user_guide/windows_winrm.html#troubleshooting-kerberos) can also be helpful.

#### Error creating pod

The job had been failed immediately after running the job. The log shows following.

```text
Error creating pod: container failed with exit code 128: failed to create containerd task: ...
```

This is usually caused by misconfigured custom pod spec of your Container Group or ConfigMap for your `krb5.conf`.

#### kinit: Cannot find KDC for realm "\<DOMAINNAME\>" while getting initial credentials

`kinit` inside the EE or job failed with following error.

```bash
bash-4.4$ kinit <username>@<DOMAINNAME>
kinit: Cannot find KDC for realm "<DOMAINNAME>" while getting initial credentials
```

```text
TASK [Ensure windows host is reachable] ****************************************
fatal: [...]: UNREACHABLE! => {
    "changed": false,
    "msg": "Kerberos auth failure for principal awx@kurokobo.internal with pexpect: Cannot find KDC for realm \"<DOMAINNAME>\" while getting initial credentials",
    "unreachable": true
}
```

If this occurred, ensure:

- `/etc/krb5.conf` is correctly configured
- Your KDC hostname can be resolved
- Your KDC can be accessed from EE
- The username for `kinit` is correct. Especially, note that the domain name in the username have to be capitalized like `awx@KUROKOBO.INTERNAL`
- If manually invoked `kinit` is succeeded but `kinit` inside the job failed, ensure the username in your Credential in AWX is correct. Note that the domain name in the username have to be capitalized like `awx@KUROKOBO.INTERNAL`

#### kerberos: the specified credentials were rejected by the server

The job failed with following error.

```text
TASK [Ensure windows host is reachable] ****************************************
fatal: [...]: UNREACHABLE! => {
    "changed": false,
    "msg": "kerberos: the specified credentials were rejected by the server",
    "unreachable": true
}
```

Ensure your domain user that used to connect to WinRM on the target host is the member of local `Administrators` group on the target host, or has permissions for `Read` and `Execute` for WinRM.

#### kerberos: Access is denied. Bad HTTP response returned from server. Code 500

The job failed with following error.

```text
TASK [Ensure windows host is reachable] ****************************************
fatal: [...]: UNREACHABLE! => {
    "changed": false,
    "msg": "kerberos: Access is denied.  (extended fault data: {'transport_message': 'Bad HTTP response returned from server. Code 500', 'http_status_code': 500, 'wsmanfault_code': '5', 'fault_code': 's:Sender', 'fault_subcode': 'w:AccessDenied'})",
    "unreachable": true
}
```

Ensure your domain user that used to connect to WinRM on the target host is the member of local `Administrators` group on the target host, or has permissions for `Read` and `Execute` for WinRM. In this case, `Execute` might be missing.

## Alternative solution (not recommended)

To replace `/etc/krb5.conf` in EE with your customized `krb5.conf`, you can also use `AWX_ISOLATION_SHOW_PATHS` settings in AWX. This is a setting to expose any path on the host to EE. If this setting is activated, it's no longer required to create a Container Group on AWX or ConfigMap on Kubernetes, that described in this guide.

However, this feature will internally mount `krb5.conf` via `hostPath`, so a customized `krb5.conf` must be placed on all Kubernetes nodes where the EE will run.

Also, side-effects and security concerns must be taken into consideration, as all EE jobs running on AWX will mount `krb5.conf` via `hostPath`, weather the job is for Windows hosts or not.

Therefore, I don't recommend this method in this guide.

If you want to use this feature, you can do so by following these steps.

1. Place your `krb5.conf` on any path on your Kubernetes node, e.g. `/data/kerberos/krb5.conf`, instead of creating a Container Group on AWX or ConfigMap on Kubernetes
2. Enable `Expose host paths for Container Groups` in AWX under `Settings` > `Job settings`.
   - This equals to set `AWX_MOUNT_ISOLATED_PATHS_ON_K8S` to `true`.
3. Add `/data/kerberos/krb5.conf:/etc/krb5.conf:O` to `Paths to expose to isolated jobs` in AWX under `Settings` > `Job settings`.
   - This equals to append string to `AWX_ISOLATION_SHOW_PATHS`.
4. Run your Job Template that without any Container Group.
