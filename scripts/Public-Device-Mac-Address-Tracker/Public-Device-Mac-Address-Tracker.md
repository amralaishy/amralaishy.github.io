
# ==========================================================
# MAC Finder Tool – Interactive (Cisco + FortiGate)
# ==========================================================
import sys
import re
import time
import getpass
import ipaddress
import subprocess
import platform
import paramiko
from concurrent.futures import ThreadPoolExecutor, as_completed
from netmiko import ConnectHandler

# ================== COMMON ==================
def normalize_mac_raw(mac):
    mac = re.sub(r'[^0-9a-fA-F]', '', mac).lower()
    if len(mac) != 12:
        raise ValueError("Invalid MAC address")
    return mac


def mac_variants(mac):
    raw = normalize_mac_raw(mac)
    return {
        f"{raw[0:4]}.{raw[4:8]}.{raw[8:12]}",
        raw,
        ":".join(raw[i:i+2] for i in range(0, 12, 2)),
        "-".join(raw[i:i+2] for i in range(0, 12, 2))
    }


def ping_ip(ip):
    system = platform.system().lower()
    cmd = ["ping", "-n", "1", "-w", "800", str(ip)] if system == "windows" \
          else ["ping", "-c", "1", "-W", "1", str(ip)]
    return subprocess.run(cmd, stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL).returncode == 0


def ping_sweep(cidr, workers=60):
    net = ipaddress.ip_network(cidr, strict=False)
    live = []
    with ThreadPoolExecutor(max_workers=workers) as pool:
        futures = {pool.submit(ping_ip, ip): ip for ip in net.hosts()}
        for f in as_completed(futures):
            if f.result():
                live.append(str(futures[f]))
    return live


# ================== CISCO ==================
def connect_cisco(ip, username, password):
    return ConnectHandler(
        device_type="cisco_ios",
        host=ip,
        username=username,
        password=password,
        fast_cli=True
    )


def get_hostname(conn):
    out = conn.send_command("show run | i ^hostname")
    return out.split()[-1] if out else "UNKNOWN"


def is_access_port(conn, iface):
    out = conn.send_command(f"show interface {iface} switchport")
    if not out:
        return False
    if "Operational Mode: trunk" in out:
        return False
    return "Access Mode VLAN" in out


def search_mac_cisco(ip, username, password, mac_inputs):
    try:
        conn = connect_cisco(ip, username, password)
        for mac in mac_inputs:
            out = conn.send_command(f"show mac address-table | include {mac}")
            if not out.strip():
                continue
            for line in out.splitlines():
                m = re.search(r"(\d+)\s+" + re.escape(mac) + r"\s+\S+\s+(\S+)", line)
                if not m:
                    continue
                vlan, iface = m.groups()
                if iface.lower() in {"cpu", "yes"}:
                    continue
                if not re.match(r"^(Fa|Gi|Te|Eth)", iface):
                    continue
                if not is_access_port(conn, iface):
                    continue

                hostname = get_hostname(conn)
                conn.disconnect()
                return {
                    "hostname": hostname,
                    "ip": ip,
                    "interface": iface,
                    "vlan": vlan,
                    "mac": mac
                }
        conn.disconnect()
    except:
        pass
    return None


def parallel_search_cisco(ips, username, password, mac_inputs):
    with ThreadPoolExecutor(max_workers=20) as pool:
        futures = [pool.submit(search_mac_cisco, ip, username, password, mac_inputs) for ip in ips]
        for f in as_completed(futures):
            if f.result():
                return f.result()
    return None


# ================== FORTIGATE ==================
def search_single_switch(ip, username, password, serial, mac_formatted):
    client = paramiko.SSHClient()
    client.set_missing_host_key_policy(paramiko.AutoAddPolicy())
    try:
        client.connect(ip, username=username, password=password, timeout=12)
        shell = client.invoke_shell()
        time.sleep(1)
        shell.send("config vdom\nedit root\n\n")
        time.sleep(1)
        shell.send(f"diagnose switch-controller mac-cache show {serial}\n")
        time.sleep(4)

        output = ""
        while shell.recv_ready():
            output += shell.recv(65535).decode(errors="ignore")
            time.sleep(0.2)

        output = re.sub(r'\x1B\[[0-?]*[ -/]*[@-~]', '', output)
        m = re.search(
            r'(\d+)\s+\d+\s+' + re.escape(mac_formatted) + r'\s+\d+\s+(\S+)',
            output,
            re.IGNORECASE
        )
        if m:
            return {"serial": serial, "vlan": m.group(1), "port": m.group(2)}
    except:
        return None
    finally:
        client.close()


def get_switch_name(ip, username, password, serial):
    client = paramiko.SSHClient()
    client.set_missing_host_key_policy(paramiko.AutoAddPolicy())
    try:
        client.connect(ip, username=username, password=password, timeout=10)
        shell = client.invoke_shell()
        time.sleep(1)
        shell.send("config vdom\nedit root\n\n")
        time.sleep(1)
        shell.send(f"show switch-controller managed-switch {serial}\n")
        time.sleep(3)

        output = ""
        while shell.recv_ready():
            output += shell.recv(8192).decode(errors="ignore")
            time.sleep(0.2)

        m = re.search(r'set name\s+"([^"]+)"', output)
        return m.group(1) if m else "No name configured"
    except:
        return "Unknown"
    finally:
        client.close()


def search_mac_fortigate(ip, username, password, mac_input):
    mac_clean = re.sub(r'[^0-9A-F]', '', mac_input.upper())
    mac_formatted = ':'.join(mac_clean[i:i+2] for i in range(0, 12, 2))

    client = paramiko.SSHClient()
    client.set_missing_host_key_policy(paramiko.AutoAddPolicy())
    client.connect(ip, username=username, password=password, timeout=12)
    shell = client.invoke_shell()
    time.sleep(1)

    shell.send("config vdom\nedit root\n\n")
    time.sleep(1)
    shell.send("execute switch-controller get-conn-status\n")
    time.sleep(4)

    output = ""
    while shell.recv_ready():
        output += shell.recv(65535).decode(errors="ignore")

    serials = list(set(re.findall(r'([A-Z0-9]{10,})\s', output)))
    client.close()

    with ThreadPoolExecutor(max_workers=5) as pool:
        futures = {
            pool.submit(search_single_switch, ip, username, password, s, mac_formatted): s
            for s in serials
        }
        for f in as_completed(futures):
            res = f.result()
            if res:
                res["switch_name"] = get_switch_name(ip, username, password, res["serial"])
                return res
    return None


# ================== MAIN ==================
def main():
    while True:
        print("""
1) Cisco Network
2) FortiGate + FortiSwitch
""")
        choice = input("Select option: ").strip()

        mac = input("MAC Address: ").strip()

        if choice == "1":
            cidr = input("Cisco network range (CIDR): ").strip()
            username = input("Cisco username: ").strip()
            password = getpass.getpass("Cisco password: ")

            mac_inputs = mac_variants(mac)
            print("\n🔍 Scanning network...\n")
            ips = ping_sweep(cidr)
            result = parallel_search_cisco(ips, username, password, mac_inputs)

            if result:
                print("\n" + "═" * 70)
                print(f" MAC      : {mac.upper()}")
                print(f" Switch   : {result['hostname']}")
                print(f" IP       : {result['ip']}")
                print(f" Interface: {result['interface']}")
                print(f" VLAN     : {result['vlan']}")
                print("═" * 70)
            else:
                print("\n❌ MAC not found")

        elif choice == "2":
            fg_ip = input("FortiGate IP: ").strip()
            username = input("FortiGate username: ").strip()
            password = getpass.getpass("FortiGate password: ")

            result = search_mac_fortigate(fg_ip, username, password, mac)
            if result:
                print("\n" + "═" * 70)
                print(f" MAC        : {mac.upper()}")
                print(f" Switch     : {result['switch_name']}")
                print(f" Serial     : {result['serial']}")
                print(f" Port       : {result['port']}")
                print(f" VLAN       : {result['vlan']}")
                print("═" * 70)
            else:
                print("\n❌ MAC not found")

        else:
            break

        if input("\nSearch another MAC? (y/n): ").lower() != "y":
            break


if __name__ == "__main__":
    main()
