# monitoring-tracert-route

#!/usr/bin/env python3
import subprocess
import socket
import os
import time
from datetime import datetime

# Warna terminal
class Color:
    GREEN = "\033[92m"
    YELLOW = "\033[93m"
    RED = "\033[91m"
    CYAN = "\033[96m"
    RESET = "\033[0m"
    BOLD = "\033[1m"

# Daftar target (hop) untuk dipantau
ROUTE_TARGETS = [
    "192.168.1.1",    # Router lokal
    "8.8.8.8",        # Google DNS
    "1.1.1.1",        # Cloudflare
    "www.netacad.com" # Cisco NetAcad
]

# Informasi tambahan tentang IP / domain
IP_INFO = {
    "192.168.1.1": "Router lokal (Gateway utama)",
    "8.8.8.8": "Google Public DNS",
    "1.1.1.1": "Cloudflare DNS",
    "www.netacad.com": "Cisco Networking Academy Server"
}

def ping_target(target):
    """Ping target dan ambil rata-rata latensinya"""
    try:
        result = subprocess.run(
            ["ping", "-c", "3", "-W", "1", target],
            stdout=subprocess.PIPE, stderr=subprocess.PIPE, text=True
        )
        if result.returncode == 0:
            for line in result.stdout.split("\n"):
                if "rtt min/avg/max" in line:
                    avg_time = line.split("/")[4]
                    return float(avg_time)
            return 0.0
        else:
            return None
    except Exception:
        return None

def get_hostname(ip):
    """Dapatkan hostname dari IP"""
    try:
        return socket.gethostbyaddr(ip)[0]
    except Exception:
        return "-"

def status_label(latency):
    """Menentukan status stabilitas berdasarkan latency"""
    if latency is None:
        return f"{Color.RED}Tidak dapat dilewati (Timeout){Color.RESET}"
    elif latency < 50:
        return f"{Color.GREEN}Stabil{Color.RESET}"
    elif latency < 150:
        return f"{Color.YELLOW}Sedikit lambat{Color.RESET}"
    else:
        return f"{Color.RED}Tidak stabil{Color.RESET}"

def print_header():
    os.system("clear")
    print(f"{Color.BOLD}{Color.CYAN}╔══════════════════════════════════════════════════════╗")
    print(f"║         PING ROUTE ANALYZER + IP INFORMATION         ║")
    print(f"╚══════════════════════════════════════════════════════╝{Color.RESET}")
    print(f"Waktu: {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}")
    print(f"{'-'*90}")

def display_results(results):
    print(f"{Color.BOLD}Hop | Target             | Hostname / Info                    | Latency (ms) | Status{Color.RESET}")
    print(f"{'-'*90}")
    good = 0
    bad = 0
    for i, (target, latency, status, info, hostname) in enumerate(results, start=1):
        latency_display = f"{latency:.2f}" if latency else "-"
        print(f"{i:<3} | {target:<18} | {hostname[:30]:<30} | {latency_display:<12} | {status}")
        print(f"      └─ Info: {info}")
        if latency is None:
            bad += 1
        else:
            good += 1
    print(f"{'-'*90}")
    print(f"\n{Color.BOLD}Rangkuman:{Color.RESET}")
    print(f"  {Color.GREEN}Stabil   : {good} hop{Color.RESET}")
    print(f"  {Color.RED}Bermasalah: {bad} hop{Color.RESET}")
    if bad == 0:
        print(f"\n{Color.GREEN}✅ Semua route dapat dilewati dan stabil!{Color.RESET}")
    else:
        print(f"\n{Color.RED}⚠️  Beberapa route tidak dapat dilewati atau tidak stabil.{Color.RESET}")

def main():
    interval_input = input("Masukkan interval pengecekan (detik, default 10): ").strip()
    interval = int(interval_input) if interval_input.isdigit() else 10

    while True:
        print_header()
        print(f"[INFO] Memeriksa stabilitas route dan informasi IP...\n")
        results = []
        for target in ROUTE_TARGETS:
            latency = ping_target(target)
            status = status_label(latency)
            info = IP_INFO.get(target, "Tidak ada informasi spesifik")
            hostname = get_hostname(target) if not target.startswith("www.") else target
            results.append((target, latency, status, info, hostname))
        display_results(results)
        print(f"\nPengecekan ulang dalam {interval} detik...\n")
        time.sleep(interval)

if __name__ == "__main__":
    main()
