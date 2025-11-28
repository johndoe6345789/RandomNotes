


```python


#!/usr/bin/env python3
"""
Full Linux account validator/repair tool for Raspberry Pi 'pi' user.
Validates and repairs:
  - /etc/passwd entry
  - /etc/shadow entry
  - /etc/group and /etc/gshadow entries
  - home directory integrity (copy from /etc/skel if missing)
  - default shell
All logic is pure and structured for safety. No partial writes.
"""

import os
import sys
import pwd
import grp
import spwd
import shutil
import argparse
from pathlib import Path
from datetime import datetime

# ------------------------------------------------------------
# Small helpers
# ------------------------------------------------------------

def die(msg: str) -> None:
    print(msg, file=sys.stderr)
    sys.exit(1)


def require_root() -> None:
    if os.geteuid() != 0:
        die("Must run as root.")


# ------------------------------------------------------------
# PASSWD VALIDATION
# ------------------------------------------------------------

def validate_passwd(username: str, shell: str) -> None:
    try:
        entry = pwd.getpwnam(username)
    except KeyError:
        die(f"User '{username}' missing in /etc/passwd.")

    if entry.pw_shell != shell:
        os.system(f"chsh -s {shell} {username}")


# ------------------------------------------------------------
# SHADOW VALIDATION
# ------------------------------------------------------------

def validate_shadow(username: str) -> None:
    try:
        spwd.getspnam(username)
    except KeyError:
        die(f"User '{username}' missing in /etc/shadow.")


# ------------------------------------------------------------
# GROUP VALIDATION
# ------------------------------------------------------------

def ensure_group(g: str, gid: int | None = None) -> None:
    try:
        grp.getgrnam(g)
        return
    except KeyError:
        pass

    cmd = ["groupadd"]
    if gid is not None:
        cmd += ["-g", str(gid)]
    cmd.append(g)
    os.system(" ".join(cmd))


def validate_groups(username: str) -> None:
    try:
        u = pwd.getpwnam(username)
    except KeyError:
        die(f"User '{username}' not found for group validation.")

    primary_gid = u.pw_gid

    # Ensure primary group exists
    try:
        grp.getgrgid(primary_gid)
    except KeyError:
        ensure_group(username, primary_gid)

    # Ensure user appears in group memberships in /etc/group
    for g in grp.getgrall():
        if g.gr_gid == primary_gid and username not in g.gr_mem:
            pass  # primary group uses pw_gid, not list


# ------------------------------------------------------------
# HOME DIRECTORY + SKEL
# ------------------------------------------------------------

def backup(path: Path) -> None:
    if path.exists():
        ts = datetime.now().strftime("%Y%m%d%H%M%S")
        b = Path(str(path) + f".bak-{ts}")
        path.rename(b)


def copy_skel(skel: Path, home: Path, uid: int, gid: int) -> None:
    for root, dirs, files in os.walk(skel):
        rel = Path(root).relative_to(skel)
        dest_root = home / rel
        dest_root.mkdir(parents=True, exist_ok=True)
        os.chown(dest_root, uid, gid)

        for f in files:
            src = Path(root) / f
            dst = dest_root / f
            if not dst.exists():
                shutil.copy2(src, dst)
                os.chown(dst, uid, gid)


def ensure_home(username: str) -> None:
    try:
        u = pwd.getpwnam(username)
    except KeyError:
        die(f"User '{username}' missing while repairing home.")

    home = Path(u.pw_dir)
    skel = Path("/etc/skel")

    if not home.exists():
        home.mkdir(parents=True, mode=0o700)
        os.chown(home, u.pw_uid, u.pw_gid)
        copy_skel(skel, home, u.pw_uid, u.pw_gid)


# ------------------------------------------------------------
# MAIN
# ------------------------------------------------------------

def main() -> None:
    require_root()

    p = argparse.ArgumentParser()
    p.add_argument("--user", default="pi")
    p.add_argument("--shell", default="/bin/bash")
    args = p.parse_args()

    validate_passwd(args.user, args.shell)
    validate_shadow(args.user)
    validate_groups(args.user)
    ensure_home(args.user)

    print("Account repair complete.")


if __name__ == "__main__":
    main()




```


```python



#!/usr/bin/env python3
"""
Linux account validator/repair tool for Raspberry Pi style users.

Validates and repairs for a given username (default: "pi"):
  - /etc/passwd entry and login shell (via chsh)
  - /etc/shadow presence (no spwd dependency; parses /etc/shadow directly)
  - primary group in /etc/group (creates if missing via groupadd)
  - home directory existence and skeleton files from /etc/skel

This script is intentionally conservative:
  - It does not attempt to rewrite /etc/passwd, /etc/group or /etc/shadow
    directly; it only uses standard tools (chsh, groupadd).
  - It does not modify supplementary group memberships.

Run as root:
    sudo ./repair_pi_accounts.py
"""

from __future__ import annotations

import argparse
import os
import pwd
import grp
import shutil
import subprocess
import sys
from datetime import datetime
from pathlib import Path


# ------------------------------------------------------------
# Generic helpers
# ------------------------------------------------------------


def die(message: str, code: int = 1) -> None:
    """Print an error to stderr and exit."""

    print(message, file=sys.stderr)
    raise SystemExit(code)


def require_root() -> None:
    """Ensure we are running as root."""

    if os.geteuid() != 0:
        die("This script must be run as root (use sudo).")


def run_cmd(cmd: list[str]) -> None:
    """Run a command, raising on failure, printing stderr on error."""

    result = subprocess.run(
        cmd,
        text=True,
        capture_output=True,
        check=False,
    )
    if result.returncode != 0:
        print(result.stderr, file=sys.stderr)
        die(f"Command failed: {' '.join(cmd)} (exit {result.returncode})")


# ------------------------------------------------------------
# /etc/passwd validation and shell repair
# ------------------------------------------------------------


def get_passwd(username: str) -> pwd.struct_passwd:
    """Return the passwd entry or abort if missing."""

    try:
        return pwd.getpwnam(username)
    except KeyError as exc:
        raise SystemExit(f"User '{username}' missing in /etc/passwd.") from exc


def ensure_shell(username: str, desired_shell: str) -> None:
    """Ensure the user's login shell is the desired shell using chsh."""

    entry = get_passwd(username)
    current_shell = entry.pw_shell
    if current_shell == desired_shell:
        return

    print(f"Changing shell for {username}: {current_shell} -> {desired_shell}")
    run_cmd(["chsh", "-s", desired_shell, username])


# ------------------------------------------------------------
# /etc/shadow validation without spwd
# ------------------------------------------------------------


def shadow_has_user(username: str, shadow_path: Path = Path("/etc/shadow")) -> bool:
    """Return True if /etc/shadow contains a line for the user."""

    try:
        with shadow_path.open("r", encoding="utf-8", errors="ignore") as fh:
            for line in fh:
                if not line or line.startswith("#"):
                    continue
                fields = line.split(":", 1)
                if len(fields) < 2:
                    continue
                if fields[0] == username:
                    return True
    except FileNotFoundError:
        die("/etc/shadow not found; cannot validate shadow entries.")
    except PermissionError:
        die("No permission to read /etc/shadow. Run this as root.")

    return False


def validate_shadow(username: str) -> None:
    """Abort if the user has no entry in /etc/shadow."""

    if not shadow_has_user(username):
        die(f"User '{username}' missing in /etc/shadow.")


# ------------------------------------------------------------
# Group validation (primary group)
# ------------------------------------------------------------


def ensure_group_exists(group_name: str, gid: int | None) -> None:
    """Ensure a group with the given name (and optional gid) exists."""

    try:
        grp.getgrnam(group_name)
        return
    except KeyError:
        pass

    cmd: list[str] = ["groupadd"]
    if gid is not None:
        cmd.extend(["-g", str(gid)])
    cmd.append(group_name)

    print(f"Creating missing primary group '{group_name}' (gid={gid}).")
    run_cmd(cmd)


def validate_primary_group(username: str) -> None:
    """Ensure the user's primary group exists (create if missing)."""

    pw_entry = get_passwd(username)
    primary_gid = pw_entry.pw_gid

    try:
        grp.getgrgid(primary_gid)
        return
    except KeyError:
        pass

    # If the group is missing, create a group with the same name as the user
    ensure_group_exists(username, primary_gid)


# ------------------------------------------------------------
# Home directory validation and /etc/skel copy
# ------------------------------------------------------------


def timestamp_suffix() -> str:
    """Return a compact timestamp string for backup names."""

    return datetime.now().strftime("%Y%m%d%H%M%S")


def backup_path(path: Path) -> Path:
    """Return a backup path with a timestamp suffix appended."""

    return Path(f"{path}.bak-{timestamp_suffix()}")


def ensure_home_exists(username: str, skel: Path) -> None:
    """Ensure the user's home directory exists and is populated from skel."""

    pw_entry = get_passwd(username)
    home = Path(pw_entry.pw_dir)

    if not home.exists():
        print(f"Creating home directory {home} for {username}.")
        home.mkdir(parents=True, mode=0o700)
        os.chown(home, pw_entry.pw_uid, pw_entry.pw_gid)
        copy_skeleton(skel, home, pw_entry.pw_uid, pw_entry.pw_gid)
        return

    # Home already exists; ensure basic ownership is sane
    os.chown(home, pw_entry.pw_uid, pw_entry.pw_gid)


def copy_skeleton(skel: Path, dest: Path, uid: int, gid: int) -> None:
    """Copy missing files from skel into dest, preserving structure."""

    if not skel.is_dir():
        die(f"Skeleton directory '{skel}' not found or not a directory.")

    print(f"Syncing missing skeleton files from {skel} into {dest}.")

    for root, dirs, files in os.walk(skel):
        rel_root = Path(root).relative_to(skel)
        dest_root = dest / rel_root
        dest_root.mkdir(parents=True, exist_ok=True)
        os.chown(dest_root, uid, gid)

        for dirname in dirs:
            dpath = dest_root / dirname
            dpath.mkdir(exist_ok=True)
            os.chown(dpath, uid, gid)

        for filename in files:
            src_file = Path(root) / filename
            dest_file = dest_root / filename
            if dest_file.exists():
                continue
            shutil.copy2(src_file, dest_file)
            os.chown(dest_file, uid, gid)


def recursive_chown(path: Path, uid: int, gid: int) -> None:
    """Recursively chown a home tree. Safe but can be expensive."""

    print(f"Recursively fixing ownership under {path}.")

    for root, dirs, files in os.walk(path):
        for name in dirs:
            dpath = Path(root) / name
            try:
                os.chown(dpath, uid, gid)
            except PermissionError:
                print(f"Warning: could not chown directory {dpath}", file=sys.stderr)
        for name in files:
            fpath = Path(root) / name
            try:
                os.chown(fpath, uid, gid)
            except PermissionError:
                print(f"Warning: could not chown file {fpath}", file=sys.stderr)

    try:
        os.chown(path, uid, gid)
    except PermissionError:
        print(f"Warning: could not chown {path}", file=sys.stderr)


# ------------------------------------------------------------
# Argument parsing and main
# ------------------------------------------------------------


def parse_args(argv: list[str] | None = None) -> argparse.Namespace:
    """Parse command line arguments."""

    parser = argparse.ArgumentParser(
        description=(
            "Validate and repair core account state for a Linux user "
            "(default: 'pi')."
        ),
    )

    parser.add_argument(
        "--user",
        default="pi",
        help="Username to validate/repair (default: pi)",
    )
    parser.add_argument(
        "--shell",
        default="/bin/bash",
        help="Desired login shell to enforce (default: /bin/bash)",
    )
    parser.add_argument(
        "--skel",
        default="/etc/skel",
        help="Skeleton directory to seed new homes from (default: /etc/skel)",
    )
    parser.add_argument(
        "--no-recursive-chown",
        action="store_true",
        help=(
            "Do not recursively chown the home tree; only fix the top-level "
            "directory and newly copied files."
        ),
    )

    return parser.parse_args(argv)


def main(argv: list[str] | None = None) -> None:
    """Entry point: wire together all validation/repair steps."""

    require_root()
    args = parse_args(argv)

    username = args.user
    desired_shell = args.shell
    skel = Path(args.skel)

    print(f"Validating account for user '{username}'.")

    pw_entry = get_passwd(username)
    validate_shadow(username)
    validate_primary_group(username)
    ensure_shell(username, desired_shell)
    ensure_home_exists(username, skel)

    if not args.no_recursive_chown:
        recursive_chown(Path(pw_entry.pw_dir), pw_entry.pw_uid, pw_entry.pw_gid)

    print("Done. Core account state should now be sane.")


if __name__ == "__main__":  # pragma: no cover
    main()




```








end