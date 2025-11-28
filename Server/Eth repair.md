


```python




#!/usr/bin/env python3
"""
fuzzy_eth_repair.py

Heuristic ("fuzzy") repair script for a Linux Ethernet interface (default: eth0).

It:
- Probes link state, IP address, routing, and DNS.
- Builds a fuzzy diagnosis (weighted suspicion scores).
- Applies safe repair actions in order of likelihood.
"""

from __future__ import annotations

import argparse
import dataclasses
import enum
import shlex
import subprocess
import sys
from typing import Dict, List, Optional, Tuple


@dataclasses.dataclass
class CommandResult:
    cmd: List[str]
    returncode: int
    stdout: str
    stderr: str


class Suspicion(enum.Enum):
    INTERFACE_MISSING = "interface_missing"
    LINK_DOWN = "link_down"
    NO_IPV4 = "no_ipv4"
    NO_ROUTE = "no_route"
    NO_INTERNET = "no_internet"
    DNS_BROKEN = "dns_broken"


SUSPICION_LABELS: Dict[Suspicion, str] = {
    Suspicion.INTERFACE_MISSING: "Interface missing",
    Suspicion.LINK_DOWN: "Link down",
    Suspicion.NO_IPV4: "No IPv4 address",
    Suspicion.NO_ROUTE: "No default route",
    Suspicion.NO_INTERNET: "Internet unreachable (ICMP)",
    Suspicion.DNS_BROKEN: "DNS resolution failing",
}


@dataclasses.dataclass
class Diagnosis:
    suspicion_scores: Dict[Suspicion, float]

    def sorted_scores(self) -> List[Tuple[Suspicion, float]]:
        return sorted(
            self.suspicion_scores.items(),
            key=lambda kv: kv[1],
            reverse=True,
        )


def run_cmd(
    cmd: List[str],
    timeout: int = 5,
    check: bool = False,
) -> CommandResult:
    """
    Run a command and capture stdout/stderr.
    """
    try:
        proc = subprocess.run(
            cmd,
            stdin=subprocess.DEVNULL,
            stdout=subprocess.PIPE,
            stderr=subprocess.PIPE,
            timeout=timeout,
            text=True,
        )
    except Exception as exc:
        return CommandResult(
            cmd=cmd,
            returncode=255,
            stdout="",
            stderr=str(exc),
        )

    if check and proc.returncode != 0:
        raise subprocess.CalledProcessError(
            proc.returncode,
            cmd,
            proc.stdout,
            proc.stderr,
        )

    return CommandResult(
        cmd=cmd,
        returncode=proc.returncode,
        stdout=proc.stdout,
        stderr=proc.stderr,
    )


def log(msg: str) -> None:
    """
    Simple log to stdout.
    """
    print(msg, flush=True)


def debug(msg: str, verbose: bool) -> None:
    """
    Conditional debug logging.
    """
    if verbose:
        log(f"[DEBUG] {msg}")


def cmd_str(cmd: List[str]) -> str:
    """
    Human-readable command string.
    """
    return " ".join(shlex.quote(part) for part in cmd)


def interface_exists(iface: str, verbose: bool) -> bool:
    """
    Determine if the interface exists according to `ip link`.
    """
    res = run_cmd(["ip", "link", "show", "dev", iface])
    debug(f"ip link show: rc={res.returncode}", verbose)
    return res.returncode == 0


def interface_link_up(iface: str, verbose: bool) -> bool:
    """
    Check if interface link is marked as UP by `ip link`.
    """
    res = run_cmd(["ip", "link", "show", "dev", iface])
    if res.returncode != 0:
        debug(
            f"ip link show failed: rc={res.returncode}, err={res.stderr}",
            verbose,
        )
        return False

    # Example flags: "<BROADCAST,MULTICAST,UP,LOWER_UP>"
    if "<UP," in res.stdout or ",UP," in res.stdout or ",UP>" in res.stdout:
        return True
    return False


def interface_has_ipv4(iface: str, verbose: bool) -> bool:
    """
    Check if interface has at least one IPv4 address.
    """
    res = run_cmd(["ip", "-4", "addr", "show", "dev", iface])
    if res.returncode != 0:
        debug(
            f"ip -4 addr show failed: rc={res.returncode}, err={res.stderr}",
            verbose,
        )
        return False
    for line in res.stdout.splitlines():
        line = line.strip()
        if line.startswith("inet "):
            return True
    return False


def has_default_route(verbose: bool) -> bool:
    """
    Check for a default route in the main routing table.
    """
    res = run_cmd(["ip", "route", "show", "default"])
    if res.returncode != 0:
        debug(
            f"ip route show default failed: rc={res.returncode}, "
            f"err={res.stderr}",
            verbose,
        )
        return False

    for line in res.stdout.splitlines():
        if line.startswith("default "):
            return True
    return False


def ping_host(
    host: str,
    count: int = 1,
    timeout: int = 3,
    verbose: bool = False,
) -> bool:
    """
    ICMP ping using system `ping`.
    """
    res = run_cmd(
        [
            "ping",
            "-c",
            str(count),
            "-w",
            str(timeout),
            host,
        ],
        timeout=timeout + 1,
    )
    debug(
        f"ping {host}: rc={res.returncode}, out={res.stdout}, "
        f"err={res.stderr}",
        verbose,
    )
    return res.returncode == 0


def dns_resolves(
    name: str = "deb.debian.org",
    verbose: bool = False,
) -> bool:
    """
    Try to resolve a hostname using `getent hosts`.
    """
    res = run_cmd(["getent", "hosts", name])
    debug(
        f"getent hosts {name}: rc={res.returncode}, out={res.stdout}, "
        f"err={res.stderr}",
        verbose,
    )
    if res.returncode != 0:
        return False
    return bool(res.stdout.strip())


def detect_network_managers(verbose: bool) -> Dict[str, bool]:
    """
    Detect presence of common network management tools/services.
    """
    managers = {
        "NetworkManager": False,
        "systemd-networkd": False,
        "ifupdown": False,
    }

    nm = run_cmd(["systemctl", "is-active", "NetworkManager"])
    managers["NetworkManager"] = nm.returncode == 0

    sn = run_cmd(["systemctl", "is-active", "systemd-networkd"])
    managers["systemd-networkd"] = sn.returncode == 0

    if_res = run_cmd(["which", "ifup"])
    managers["ifupdown"] = if_res.returncode == 0

    debug(f"Detected managers: {managers}", verbose)

    return managers


def fuzzy_diagnose(
    iface: str,
    verbose: bool,
) -> Diagnosis:
    """
    Build a fuzzy diagnosis from all probes.
    """
    scores: Dict[Suspicion, float] = {
        Suspicion.INTERFACE_MISSING: 0.0,
        Suspicion.LINK_DOWN: 0.0,
        Suspicion.NO_IPV4: 0.0,
        Suspicion.NO_ROUTE: 0.0,
        Suspicion.NO_INTERNET: 0.0,
        Suspicion.DNS_BROKEN: 0.0,
    }

    exists = interface_exists(iface, verbose)
    link_up = interface_link_up(iface, verbose) if exists else False
    has_ip = interface_has_ipv4(iface, verbose) if exists else False
    default_route = has_default_route(verbose)
    can_ping_ip = ping_host("8.8.8.8", verbose=verbose)
    can_resolve = dns_resolves(verbose=verbose)

    if not exists:
        scores[Suspicion.INTERFACE_MISSING] = 1.0
        return Diagnosis(scores)

    if not link_up:
        scores[Suspicion.LINK_DOWN] = 0.8

    if not has_ip:
        scores[Suspicion.NO_IPV4] = 0.7

    if not default_route:
        scores[Suspicion.NO_ROUTE] = 0.6

    if not can_ping_ip:
        scores[Suspicion.NO_INTERNET] = 0.6

    if can_ping_ip and not can_resolve:
        scores[Suspicion.DNS_BROKEN] = 0.9
    elif not can_resolve:
        scores[Suspicion.DNS_BROKEN] = 0.4

    return Diagnosis(scores)


def apply_action(
    desc: str,
    cmd: List[str],
    dry_run: bool,
    verbose: bool,
) -> bool:
    """
    Run a repair action (or just log it in dry-run mode).
    """
    log(f"[ACTION] {desc}")
    log(f"         {cmd_str(cmd)}")
    if dry_run:
        return True

    res = run_cmd(cmd, timeout=15)
    debug(
        f"action rc={res.returncode}, out={res.stdout}, err={res.stderr}",
        verbose,
    )
    if res.returncode != 0:
        log(
            "[WARN] Action failed "
            f"(rc={res.returncode}): {cmd_str(cmd)}",
        )
        return False
    return True


def repair_interface_missing(
    iface: str,
    dry_run: bool,
    verbose: bool,
) -> None:
    """
    For a missing interface, there is no safe generic fix.
    """
    log(
        "[INFO] Interface does not exist. This usually means a driver or "
        "hardware issue.",
    )
    log(
        "[HINT] Check: lspci / lsusb, kernel modules, dmesg, or hypervisor "
        "settings.",
    )


def repair_link_down(
    iface: str,
    dry_run: bool,
    verbose: bool,
) -> None:
    """
    Attempt to bring interface up.
    """
    apply_action(
        f"Bring {iface} link up via ip",
        ["sudo", "ip", "link", "set", iface, "up"],
        dry_run,
        verbose,
    )


def repair_no_ipv4(
    iface: str,
    managers: Dict[str, bool],
    dry_run: bool,
    verbose: bool,
) -> None:
    """
    Try to obtain an IPv4 lease.
    """
    if managers.get("NetworkManager", False):
        apply_action(
            f"Ask NetworkManager to bring {iface} up",
            ["sudo", "nmcli", "device", "connect", iface],
            dry_run,
            verbose,
        )
        return

    if managers.get("systemd-networkd", False):
        apply_action(
            "Restart systemd-networkd",
            ["sudo", "systemctl", "restart", "systemd-networkd"],
            dry_run,
            verbose,
        )
        return

    if managers.get("ifupdown", False):
        apply_action(
            f"ifup {iface}",
            ["sudo", "ifup", iface],
            dry_run,
            verbose,
        )
        return

    apply_action(
        f"Run dhclient on {iface}",
        ["sudo", "dhclient", "-v", iface],
        dry_run,
        verbose,
    )


def repair_no_route(
    dry_run: bool,
    verbose: bool,
) -> None:
    """
    Try to nudge routing by restarting the main network manager.
    """
    managers = detect_network_managers(verbose)
    if managers.get("NetworkManager", False):
        apply_action(
            "Restart NetworkManager",
            ["sudo", "systemctl", "restart", "NetworkManager"],
            dry_run,
            verbose,
        )
        return

    if managers.get("systemd-networkd", False):
        apply_action(
            "Restart systemd-networkd",
            ["sudo", "systemctl", "restart", "systemd-networkd"],
            dry_run,
            verbose,
        )
        return

    if managers.get("ifupdown", False):
        apply_action(
            "Restart networking service (ifupdown)",
            ["sudo", "systemctl", "restart", "networking"],
            dry_run,
            verbose,
        )
        return

    log(
        "[INFO] No known manager for default route; "
        "you may need to add it manually.",
    )


def repair_dns(
    allow_resolv_conf_edit: bool,
    dry_run: bool,
    verbose: bool,
) -> None:
    """
    Try to fix DNS in a conservative way first.
    """
    res = run_cmd(["systemctl", "is-active", "systemd-resolved"])
    if res.returncode == 0:
        apply_action(
            "Restart systemd-resolved",
            ["sudo", "systemctl", "restart", "systemd-resolved"],
            dry_run,
            verbose,
        )
        return

    if not allow_resolv_conf_edit:
        log(
            "[INFO] DNS appears broken, but resolv.conf editing is disabled. "
            "Check /etc/resolv.conf and your router.",
        )
        return

    apply_action(
        "Backup /etc/resolv.conf",
        ["sudo", "cp", "/etc/resolv.conf", "/etc/resolv.conf.bak"],
        dry_run,
        verbose,
    )

    apply_action(
        "Write temporary resolver to /etc/resolv.conf "
        "(1.1.1.1 / 8.8.8.8)",
        [
            "sudo",
            "bash",
            "-c",
            (
                "printf '%s\n' "
                "'nameserver 1.1.1.1' "
                "'nameserver 8.8.8.8' "
                "> /etc/resolv.conf"
            ),
        ],
        dry_run,
        verbose,
    )


def repair_no_internet(
    dry_run: bool,
    verbose: bool,
) -> None:
    """
    Generic response for no ICMP reachability.
    """
    managers = detect_network_managers(verbose)
    if managers.get("NetworkManager", False):
        apply_action(
            "Restart NetworkManager",
            ["sudo", "systemctl", "restart", "NetworkManager"],
            dry_run,
            verbose,
        )
        return

    if managers.get("systemd-networkd", False):
        apply_action(
            "Restart systemd-networkd",
            ["sudo", "systemctl", "restart", "systemd-networkd"],
            dry_run,
            verbose,
        )
        return

    if managers.get("ifupdown", False):
        apply_action(
            "Restart networking service (ifupdown)",
            ["sudo", "systemctl", "restart", "networking"],
            dry_run,
            verbose,
        )
        return

    log(
        "[INFO] Internet reachability still failing; "
        "check router, cabling, or ISP.",
    )


def perform_repairs(
    iface: str,
    diagnosis: Diagnosis,
    dry_run: bool,
    verbose: bool,
    allow_resolv_conf_edit: bool,
) -> None:
    """
    Apply repairs in descending order of suspicion.
    """
    managers = detect_network_managers(verbose)

    log("")
    log("=== Fuzzy diagnosis ===")
    for susp, score in diagnosis.sorted_scores():
        if score <= 0.0:
            continue
        label = SUSPICION_LABELS[susp]
        log(f"{label:20s}: {score:.2f}")

    log("")
    log("=== Repair phase ===")

    if diagnosis.suspicion_scores.get(Suspicion.INTERFACE_MISSING, 0.0) > 0.5:
        repair_interface_missing(iface, dry_run, verbose)
        return

    if diagnosis.suspicion_scores.get(Suspicion.LINK_DOWN, 0.0) > 0.4:
        repair_link_down(iface, dry_run, verbose)

    if diagnosis.suspicion_scores.get(Suspicion.NO_IPV4, 0.0) > 0.4:
        repair_no_ipv4(iface, managers, dry_run, verbose)

    if diagnosis.suspicion_scores.get(Suspicion.NO_ROUTE, 0.0) > 0.4:
        repair_no_route(dry_run, verbose)

    if diagnosis.suspicion_scores.get(Suspicion.NO_INTERNET, 0.0) > 0.4:
        repair_no_internet(dry_run, verbose)

    if diagnosis.suspicion_scores.get(Suspicion.DNS_BROKEN, 0.0) > 0.4:
        repair_dns(allow_resolv_conf_edit, dry_run, verbose)


def parse_args(argv: Optional[List[str]] = None) -> argparse.Namespace:
    """
    Argument parser.
    """
    parser = argparse.ArgumentParser(
        description=(
            "Fuzzy heuristic repair for a Linux Ethernet interface "
            "(default: eth0)."
        ),
    )
    parser.add_argument(
        "-i",
        "--interface",
        default="eth0",
        help="Interface to attempt to repair (default: eth0).",
    )
    parser.add_argument(
        "--dry-run",
        action="store_true",
        help="Do not run any mutating commands, only log them.",
    )
    parser.add_argument(
        "--verbose",
        action="store_true",
        help="Enable verbose debug output.",
    )
    parser.add_argument(
        "--allow-resolv-conf-edit",
        action="store_true",
        help=(
            "Allow the script to overwrite /etc/resolv.conf with public "
            "DNS (backup created first)."
        ),
    )
    return parser.parse_args(argv)


def main(argv: Optional[List[str]] = None) -> int:
    """
    Entry point.
    """
    args = parse_args(argv)
    iface: str = args.interface
    dry_run: bool = args.dry_run
    verbose: bool = args.verbose
    allow_resolv_conf_edit: bool = args.allow_resolv_conf_edit

    log(f"[INFO] Fuzzy repair for interface: {iface}")
    if dry_run:
        log("[INFO] Dry-run mode enabled (no changes will be made).")

    diagnosis = fuzzy_diagnose(iface, verbose)
    perform_repairs(
        iface=iface,
        diagnosis=diagnosis,
        dry_run=dry_run,
        verbose=verbose,
        allow_resolv_conf_edit=allow_resolv_conf_edit,
    )

    log("")
    log("[INFO] Repair phase completed.")
    log("[INFO] Re-check connectivity after a few seconds.")

    return 0


if __name__ == "__main__":
    raise SystemExit(main())




```





end

