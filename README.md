# OpenDaylight Practical Implementation Guide

> **Setup:** Open **2 terminals** — one for OpenDaylight, one for Mininet. You can rename the tabs in Lubuntu for convenience.

---

## Prerequisites

```bash
sudo apt update
```

---

## Part 1: OpenDaylight Setup

### Step 1 — Download OpenDaylight (Oxygen Release)

```bash
wget https://nexus.opendaylight.org/content/repositories/opendaylight.release/org/opendaylight/integration/karaf/0.8.4/karaf-0.8.4.zip
```

> 💡 You can also find this file uploaded on GitHub if the direct download fails.

---

### Step 2 — Unzip the Downloaded File

```bash
unzip "karaf-0.8.4.zip"
```

> ⚠️ The quotes around the filename are important.

---

### Step 3 — Navigate into the Directory

```bash
cd ~/karaf-0.8.4
```

---

### Step 4 — Install Java 8 JDK

> **If you have broken/ghost Java files**, clean them first:
> ```bash
> sudo rm -rf /usr/lib/jvm/*
> sudo apt-get purge "*openjdk*" -y
> sudo apt-get autoremove -y
> ```
> Then proceed with a fresh install below.

```bash
sudo apt-get install openjdk-8-jdk -y
```

---

### Step 5 — Set Permanent Java Path

Open your `.bashrc` file:

```bash
nano ~/.bashrc
```

Add this line at the **bottom** of the file:

```bash
export JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64
```

Save with `Ctrl+O` → `Enter`, then exit with `Ctrl+X`. Apply the changes:

```bash
source ~/.bashrc
```

---

### Step 6 — Install & Load Networking Drivers

Install Open vSwitch and Mininet:

```bash
sudo apt install openvswitch-switch mininet -y
```

Load the kernel driver:

```bash
sudo modprobe openvswitch
```

Verify it loaded successfully:

```bash
lsmod | grep openvswitch
```

> ✅ **Expected output:** A line starting with `openvswitch` followed by numbers and dependent modules like `nf_nat`, `nf_conntrack`.

---

### Step 7 — Launch OpenDaylight Controller

> Run this from inside the `karaf-0.8.4` directory only.

```bash
./bin/karaf
```

---

### Step 8 — Install Required Features (inside Karaf shell)

```bash
feature:install odl-restconf odl-l2switch-switch odl-mdsal-apidocs
```

> ⏳ This will take **2–3 minutes**. Wait for it to complete before moving on.

---

## Part 2: Mininet Setup

> Switch to the **Mininet terminal** for the following steps.

### Step 9 — Install Mininet & Clean Up

```bash
sudo apt install mininet -y
```

Always clear previous failed attempts first:

```bash
sudo mn -c
```

---

### Step 10 — Launch Mininet Topology

Since both OpenDaylight and Mininet are running on the same machine, use `127.0.0.1` (localhost):

```bash
sudo mn --topo linear,3 --mac --controller remote,ip=127.0.0.1,port=6633 --switch ovs,protocols=OpenFlow13
```

> ⚠️ Type this command exactly as shown to avoid errors.

---

### Step 11 — Verify the Setup

Once you see the `mininet>` prompt, run the following checks:

**Check hardware connections** (ensure switches are connected to ODL):

```
mininet> sh ovs-vsctl show
```

Look for: `Bridge s1`, `Bridge s2`, `Bridge s3`, and `is_connected: true`.

**Test connectivity:**

```
mininet> pingall
```

> ✅ Expected result: **0% packet loss**

---

## Part 3: Proof of SDN (Practical Submission)

> The Brain (OpenDaylight) is successfully controlling the Body (Mininet switches). Now collect the proof.

---

### Proof 1 — View Flow Tables (Southbound API Proof)

OpenDaylight pushes "Flow Rules" to the switches when pings occur. View them:

```
mininet> sh ovs-ofctl -O OpenFlow13 dump-flows s1
```

> Replace `s1` with `s2` or `s3` to inspect other switches.

**What you're seeing:**
- Lines containing `cookie`, `duration`, and `actions` show the controller's instructions.
- `odl-l2switch-switch` learned MAC addresses from the first ping and pushed output rules to the switches.
- This is the **Southbound Interface** (OpenFlow 1.3) in action — the controller programming "dumb" switches.

> ⚠️ Note: `dpctl dump-flows` may throw an error due to OpenFlow version mismatch. Use the `ovs-ofctl` command above instead.

---

### Proof 2 — Query Topology via REST API (Northbound API Proof)

Open a **third terminal** and run:

```bash
curl -u admin:admin http://localhost:8181/restconf/operational/network-topology:network-topology
```

**What to look for:** A JSON response containing `node-id`, `s1`, `s2`, and `s3`.

This proves that an external application can read the network state from the controller via the **Northbound API**.

**This completes the full SDN loop:**

| Layer | Component | Role |
|---|---|---|
| Data Plane | Mininet | Created the hosts |
| Southbound | OpenFlow | Connected switches to the controller |
| Control Plane | OpenDaylight | Learned the path, enabled pings |
| Northbound | RESTCONF | Exposed network state to the outside world |

---

### Proof 3 — Test Network Agility (Dynamic SDN Proof)

SDN is designed to be flexible. Simulate a link failure and observe the controller react:

```
mininet> link s1 s2 down
```

```
mininet> pingall
```
*(Some hosts will be unreachable — expected.)*

```
mininet> link s1 s2 up
```

```
mininet> pingall
```

> ✅ Everything should work again. This proves the controller **dynamically manages topology changes** in real-time.

---

## Part 4: Cleanup

### Step 1 — Exit Mininet

```
mininet> exit
```

### Step 2 — Clean Virtual Hardware

```bash
sudo mn -c
```

### Step 3 — Stop OpenDaylight

In the Karaf terminal:

```
opendaylight-user@root> logout
```

---

## Key Concepts (Viva / Theory)

| Concept | Explanation |
|---|---|
| **Network Agility** | The SDN controller detects topology changes and updates flow tables automatically in real-time |
| **Separation of Concerns** | OVS handles traffic forwarding; OpenDaylight decides *where* traffic goes |
| **Scalability** | One Karaf instance can manage thousands of switches in a real-world deployment |
