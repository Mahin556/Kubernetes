A network namespace provides an isolated networking stack for processes.

Each namespace has its own interfaces, routing tables, iptables rules, ARP table, and sockets.

By default, everything runs in the root (default) namespace.

When you create a new namespace, it’s like a separate mini-computer’s network stack inside your Linux host.

```bash
sudo apt install iproute2 -y   # or yum install iproute -y
```
```bash
ip netns list #List all namespaces in /var/run/netns/
```
* By default, no custom namespaces are listed.
* But Kubernetes, Docker, or containerd create them internally (you can check with ip netns inside /var/run/netns/).

```bash
#########################################
# 🧱 Step 1: Create Network Namespaces
#########################################

ip netns add ns1             # Create first network namespace (like Pod 1)
ip netns add ns2             # Create second network namespace (like Pod 2)
ip netns list                # Verify namespaces exist (they're files in /var/run/netns/)

ls /var/run/netns/           # Shows namespace entries created by iproute2

#########################################
# 🌐 Step 2: Inspect Network Stack Inside a Namespace
#########################################

ip netns exec ns1 ip addr    # Show interfaces in ns1 (only loopback 'lo' exists by default)

#########################################
# 🔌 Step 3: Create a Virtual Ethernet (veth) Pair
#########################################

ip link add veth-ns1 type veth peer name veth-br   # Create a veth pair (veth-ns1 ↔ veth-br)
ip link show type veth                            # Verify both ends exist

#########################################
# 🔗 Step 4: Move One End into Namespace
#########################################

ip link set veth-ns1 netns ns1    # Move one end into ns1; veth-br stays in root namespace
ip netns exec ns1 ip link         # Verify inside ns1 — you’ll see veth-ns1@ifX

#########################################
# 📡 Step 5: Assign IP Addresses
#########################################

ip addr add 10.0.0.1/24 dev veth-br                     # Assign IP to host end
ip netns exec ns1 ip addr add 10.0.0.2/24 dev veth-ns1  # Assign IP to ns1’s interface

#########################################
# ⚙️ Step 6: Bring Interfaces UP
#########################################

ip link set veth-br up                     # Activate host-side interface
ip netns exec ns1 ip link set veth-ns1 up  # Activate ns1-side interface
ip netns exec ns1 ip link set lo up        # Bring loopback up inside ns1

#########################################
# 🧭 Step 7: Test Connectivity Between ns1 and Host
#########################################

ip netns exec ns1 ping 10.0.0.1 -c 3       # Test packet flow over veth pair (works both ways)

#########################################
# 🌉 Step 8: Connect Multiple Namespaces Using a Linux Bridge
#########################################

ip link add name br0 type bridge           # Create Linux bridge (acts as virtual switch)
ip link set br0 up                         # Activate bridge

# Create veth pairs to connect ns1 and ns2 to bridge
ip link add veth-ns1 type veth peer name veth-br1
ip link add veth-ns2 type veth peer name veth-br2

# Move one end of each pair into its namespace
ip link set veth-ns1 netns ns1
ip link set veth-ns2 netns ns2

# Attach bridge-side ends to bridge (like connecting cables to switch)
ip link set veth-br1 master br0
ip link set veth-br2 master br0

#########################################
# 📡 Step 9: Assign IP Addresses to Namespaces
#########################################

ip netns exec ns1 ip addr add 192.168.10.1/24 dev veth-ns1
ip netns exec ns2 ip addr add 192.168.10.2/24 dev veth-ns2

#########################################
# ⚙️ Step 10: Bring All Interfaces UP
#########################################

ip link set veth-br1 up
ip link set veth-br2 up
ip link set br0 up

ip netns exec ns1 ip link set veth-ns1 up
ip netns exec ns1 ip link set lo up
ip netns exec ns2 ip link set veth-ns2 up
ip netns exec ns2 ip link set lo up

#########################################
# 🧭 Step 11: Test Connectivity Between ns1 and ns2
#########################################

ip netns exec ns1 ping 192.168.10.2 -c 3   # Works via bridge br0 (acts like switch)

#########################################
# 🌍 Step 12: Enable Internet Access in Namespace
#########################################

# Give ns1 a default route to the bridge gateway (host)
ip netns exec ns1 ip route add default via 10.0.0.1

# Enable Linux IP forwarding globally
sysctl -w net.ipv4.ip_forward=1

# Masquerade outbound packets (NAT) from the namespace to the Internet
# Replace 'eth0' with your actual outbound interface (use `ip route get 8.8.8.8` to check)
iptables -t nat -A POSTROUTING -s 10.0.0.0/24 -o eth0 -j MASQUERADE

# Test internet connectivity from inside ns1
ip netns exec ns1 ping 8.8.8.8 -c 3        # Should work via NAT on host

#########################################
# ⚡ Step 13: Run Commands Inside Namespace (like a container shell)
#########################################

ip netns exec ns1 bash                     # Open shell inside ns1 namespace
# Inside: you can run ping, curl, ip addr, etc. in isolated environment

# Or run one-liner commands directly:
ip netns exec ns1 curl http://example.com

#########################################
# 🧹 Step 14: Cleanup Everything
#########################################

ip netns del ns1                           # Delete first namespace
ip netns del ns2                           # Delete second namespace
ip link del br0                            # Remove the bridge

#########################################
# 🔍 Optional: Inspect State of Network
#########################################

ip link                                     # List all interfaces (including bridges/veths)
ip link show type bridge                    # Show bridge interfaces
brctl show                                  # Alternative tool to display bridge topology
ip netns exec ns1 ip route                  # Show routes inside namespace
ip netns exec ns1 ip neigh                  # Show ARP (neighbor) table inside ns1

#########################################
# 🧠 Theory Recap — How Kubernetes Uses These Concepts
#########################################

# When Kubernetes creates a Pod:
# 1️⃣ Container runtime (containerd/CRI-O) calls `ip netns add` → new Pod namespace.
# 2️⃣ CNI plugin runs:
#     - `ip link add` to create veth pair
#     - Moves one end to Pod namespace
#     - Attaches host end to Linux bridge (cni0) or overlay interface
#     - Assigns IP (like 10.244.x.x)
#     - Adds routes using `ip route`
# 3️⃣ kube-proxy adds iptables or IPVS rules for Service routing.
# 4️⃣ conntrack tracks NATed connections for stateful communication.
# 5️⃣ sysctl, ip_forward, and NAT ensure external connectivity.

# 👉 The commands above replicate the full Linux-level foundation Kubernetes automates for Pod networking.

```