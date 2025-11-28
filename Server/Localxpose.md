


```python

"""PyQt6 wrapper GUI for `loclx tunnel http` with persistence, auto-restart,
and system tray integration (SVG tray icon + minimize-to-tray).

Features:
    - Task list model of LocalXpose HTTP tunnels.
    - Per-task configuration mapped to CLI flags.
    - Start/stop tunnels via QProcess.
    - Command preview + live output log.
    - Config persistence using OS-specific app data directory:
        - Windows: %APPDATA%\LocalXposeTunnelManager\config.json
        - Linux:   ~/.config/localxpose_tunnel_manager/config.json
        - macOS:   ~/Library/Application Support/LocalXposeTunnelManager/config.json
    - Optional per-task "auto restart" on unexpected process exit.
    - SVG-based system tray icon:
        - App lives in tray when window is closed.
        - Tray menu with "Show/Hide" and "Quit".
        - Explicit "Minimize to tray" button in the UI.

Requirements:
    - Python 3.9+
    - PyQt6 (pip install PyQt6)
    - PyQt6-Qt6 includes SVG plugin (usual default).
    - LocalXpose CLI in PATH (binary named "loclx" or configure full path).

Run:
    python localxpose_gui.py
"""

from __future__ import annotations

import sys
import shlex
import json
import os
from dataclasses import dataclass, field, asdict
from pathlib import Path
from typing import List, Optional, Any, Dict

from PyQt6.QtCore import QProcess, Qt, QSize
from PyQt6.QtGui import QIcon, QPixmap, QAction
from PyQt6.QtSvgWidgets import QSvgRenderer
from PyQt6.QtWidgets import (
    QApplication,
    QWidget,
    QMainWindow,
    QListWidget,
    QListWidgetItem,
    QLineEdit,
    QLabel,
    QPushButton,
    QVBoxLayout,
    QHBoxLayout,
    QFormLayout,
    QCheckBox,
    QTextEdit,
    QMessageBox,
    QGroupBox,
    QSystemTrayIcon,
    QMenu,
)


# ---------------------------------------------------------------------------
# SVG icon
# ---------------------------------------------------------------------------

# Simple inline SVG icon: rounded rectangle with "LX" text
TRAY_ICON_SVG = b"""
<svg xmlns="http://www.w3.org/2000/svg" width="64" height="64">
  <defs>
    <linearGradient id="bg" x1="0" x2="1" y1="0" y2="1">
      <stop offset="0%" stop-color="#1976d2"/>
      <stop offset="100%" stop-color="#0d47a1"/>
    </linearGradient>
  </defs>
  <rect x="4" y="4" width="56" height="56" rx="12" ry="12" fill="url(#bg)"/>
  <text x="50%" y="52%" text-anchor="middle" dominant-baseline="middle"
        font-family="Segoe UI, Roboto, sans-serif"
        font-size="26" fill="#ffffff" font-weight="bold">
    LX
  </text>
</svg>
"""


def build_tray_icon(size: int = 32) -> QIcon:
    """Render the inline SVG to a QIcon at the requested size."""
    renderer = QSvgRenderer(TRAY_ICON_SVG)
    pixmap = QPixmap(size, size)
    pixmap.fill(Qt.GlobalColor.transparent)
    painter = None
    from PyQt6.QtGui import QPainter  # local import to avoid unused warning

    painter = QPainter(pixmap)
    renderer.render(painter)
    painter.end()
    icon = QIcon(pixmap)
    return icon


# ---------------------------------------------------------------------------
# Helpers: config directory / persistence
# ---------------------------------------------------------------------------


def get_config_dir() -> Path:
    """Return an OS-appropriate config directory for this app."""
    app_name = "LocalXposeTunnelManager"

    if sys.platform.startswith("win"):
        base = os.environ.get("APPDATA") or os.path.expanduser("~\\AppData\\Roaming")
        return Path(base) / app_name
    if sys.platform == "darwin":
        base = os.path.expanduser("~/Library/Application Support")
        return Path(base) / app_name

    base = os.path.expanduser("~/.config")
    return Path(base) / app_name


def get_config_path() -> Path:
    return get_config_dir() / "config.json"


# ---------------------------------------------------------------------------
# Data model
# ---------------------------------------------------------------------------


@dataclass
class TunnelConfig:
    """Data model for a LocalXpose HTTP tunnel configuration."""

    name: str = "New Tunnel"
    loclx_path: str = "loclx"
    to: str = "127.0.0.1:8080"
    region: str = ""
    subdomain: str = ""
    reserved_domain: str = ""
    https_to: str = ""
    crt_path: str = ""
    key_path: str = ""

    basic_auth: str = ""
    key_auth: str = ""
    rate_limit: str = ""  # numeric string

    request_headers: List[str] = field(default_factory=list)
    response_headers: List[str] = field(default_factory=list)
    prefix_path: str = ""
    https_redirect: bool = False
    ip_whitelist: List[str] = field(default_factory=list)

    file_server: str = ""  # path for --file-server

    auto_restart: bool = False  # restart on unexpected exit

    @classmethod
    def from_dict(cls, data: Dict[str, Any]) -> "TunnelConfig":
        """Create a TunnelConfig from a dict, tolerating missing/extra keys."""
        cfg = cls()
        for key, value in data.items():
            if not hasattr(cfg, key):
                continue
            setattr(cfg, key, value)
        if not isinstance(cfg.request_headers, list):
            cfg.request_headers = []
        if not isinstance(cfg.response_headers, list):
            cfg.response_headers = []
        if not isinstance(cfg.ip_whitelist, list):
            cfg.ip_whitelist = []
        return cfg


class TunnelTask:
    """Represents a runnable tunnel task (config + process)."""

    def __init__(self, config: Optional[TunnelConfig] = None) -> None:
        self.config: TunnelConfig = config or TunnelConfig()
        self.process: Optional[QProcess] = None
        self._stopping: bool = False  # distinguishes user stop vs unexpected exit

    def is_running(self) -> bool:
        return self.process is not None and self.process.state() != QProcess.ProcessState.NotRunning

    def build_command(self) -> List[str]:
        """Build the command-line argument list for this tunnel config."""
        cfg = self.config
        cmd: List[str] = [cfg.loclx_path, "tunnel", "http"]

        if cfg.to:
            cmd.extend(["--to", cfg.to])
        if cfg.region:
            cmd.extend(["--region", cfg.region])
        if cfg.subdomain:
            cmd.extend(["--subdomain", cfg.subdomain])
        if cfg.reserved_domain:
            cmd.extend(["--reserved-domain", cfg.reserved_domain])
        if cfg.https_to:
            cmd.extend(["--https-to", cfg.https_to])
        if cfg.crt_path:
            cmd.extend(["--crt", cfg.crt_path])
        if cfg.key_path:
            cmd.extend(["--key", cfg.key_path])

        if cfg.basic_auth:
            cmd.extend(["--basic-auth", cfg.basic_auth])
        if cfg.key_auth:
            cmd.extend(["--key-auth", cfg.key_auth])
        if cfg.rate_limit:
            cmd.extend(["--rate-limit", cfg.rate_limit])

        for ip in cfg.ip_whitelist:
            cleaned = ip.strip()
            if cleaned:
                cmd.extend(["--ip-whitelist", cleaned])

        for h in cfg.request_headers:
            cleaned = h.strip()
            if cleaned:
                cmd.extend(["--request-header", cleaned])

        for h in cfg.response_headers:
            cleaned = h.strip()
            if cleaned:
                cmd.extend(["--response-header", cleaned])

        if cfg.prefix_path:
            cmd.extend(["--prefix-path", cfg.prefix_path])

        if cfg.https_redirect:
            cmd.append("--https-redirect")

        if cfg.file_server:
            cmd.extend(["--file-server", cfg.file_server])

        return cmd


# ---------------------------------------------------------------------------
# Main window
# ---------------------------------------------------------------------------


class MainWindow(QMainWindow):
    """Main window providing a task list, detail editor, and tray integration."""

    def __init__(self) -> None:
        super().__init__()
        self.setWindowTitle("LocalXpose HTTP Tunnel Manager")
        self.resize(1100, 650)

        icon = build_tray_icon(32)
        self.setWindowIcon(icon)

        self.tasks: List[TunnelTask] = []

        central = QWidget(self)
        self.setCentralWidget(central)

        main_layout = QHBoxLayout(central)

        # Left: task list and controls
        self.task_list = QListWidget()
        self.task_list.currentRowChanged.connect(self.on_task_selection_changed)

        self.btn_add_task = QPushButton("Add")
        self.btn_duplicate_task = QPushButton("Duplicate")
        self.btn_remove_task = QPushButton("Remove")

        self.btn_add_task.clicked.connect(self.add_task)
        self.btn_duplicate_task.clicked.connect(self.duplicate_task)
        self.btn_remove_task.clicked.connect(self.remove_task)

        left_button_row = QHBoxLayout()
        left_button_row.addWidget(self.btn_add_task)
        left_button_row.addWidget(self.btn_duplicate_task)
        left_button_row.addWidget(self.btn_remove_task)

        left_layout = QVBoxLayout()
        left_layout.addWidget(QLabel("Tunnels"))
        left_layout.addWidget(self.task_list, 1)
        left_layout.addLayout(left_button_row)

        # Right: detail editor + run controls + command preview
        right_layout = QVBoxLayout()

        form_layout = QFormLayout()

        self.edit_name = QLineEdit()
        self.edit_loclx_path = QLineEdit("loclx")
        self.edit_to = QLineEdit("127.0.0.1:8080")
        self.edit_region = QLineEdit()
        self.edit_subdomain = QLineEdit()
        self.edit_reserved_domain = QLineEdit()
        self.edit_https_to = QLineEdit()
        self.edit_crt_path = QLineEdit()
        self.edit_key_path = QLineEdit()

        self.edit_basic_auth = QLineEdit()
        self.edit_key_auth = QLineEdit()
        self.edit_rate_limit = QLineEdit()

        self.edit_prefix_path = QLineEdit()
        self.check_https_redirect = QCheckBox("Enable HTTPS redirect (http -> https)")

        self.edit_file_server = QLineEdit()

        self.text_ip_whitelist = QTextEdit()
        self.text_ip_whitelist.setPlaceholderText("One IP or CIDR per line")

        self.text_request_headers = QTextEdit()
        self.text_request_headers.setPlaceholderText("One header per line, e.g. host:myapp.com")

        self.text_response_headers = QTextEdit()
        self.text_response_headers.setPlaceholderText("One header per line, e.g. role:admin")

        form_layout.addRow(QLabel("General"))
        form_layout.addRow("Name", self.edit_name)
        form_layout.addRow("Loclx path", self.edit_loclx_path)
        form_layout.addRow("To (host:port)", self.edit_to)
        form_layout.addRow("Region", self.edit_region)
        form_layout.addRow("Subdomain", self.edit_subdomain)
        form_layout.addRow("Reserved domain", self.edit_reserved_domain)
        form_layout.addRow("HTTPS to (host:port)", self.edit_https_to)
        form_layout.addRow("TLS cert path", self.edit_crt_path)
        form_layout.addRow("TLS key path", self.edit_key_path)

        form_layout.addRow(QLabel("Auth & limits"))
        form_layout.addRow("Basic auth (user:pass)", self.edit_basic_auth)
        form_layout.addRow("Key auth (X-Token)", self.edit_key_auth)
        form_layout.addRow("Rate limit (req/sec)", self.edit_rate_limit)

        form_layout.addRow(QLabel("Headers & routing"))
        form_layout.addRow("Prefix path", self.edit_prefix_path)
        form_layout.addRow(self.check_https_redirect)

        form_layout.addRow("IP whitelist", self.text_ip_whitelist)
        form_layout.addRow("Request headers", self.text_request_headers)
        form_layout.addRow("Response headers", self.text_response_headers)

        form_layout.addRow(QLabel("Apps"))
        form_layout.addRow("File server path", self.edit_file_server)

        right_layout.addLayout(form_layout, 3)

        # Run controls + auto-restart + minimize-to-tray
        run_box = QGroupBox("Run control")
        run_layout = QHBoxLayout(run_box)
        self.btn_start = QPushButton("Start tunnel")
        self.btn_stop = QPushButton("Stop tunnel")
        self.check_auto_restart = QCheckBox("Auto restart on exit")
        self.btn_minimize_tray = QPushButton("Minimize to tray")
        self.label_status = QLabel("Status: idle")

        self.btn_start.clicked.connect(self.start_selected_tunnel)
        self.btn_stop.clicked.connect(self.stop_selected_tunnel)
        self.check_auto_restart.stateChanged.connect(self._on_any_field_changed)
        self.btn_minimize_tray.clicked.connect(self.minimize_to_tray)

        run_layout.addWidget(self.btn_start)
        run_layout.addWidget(self.btn_stop)
        run_layout.addWidget(self.check_auto_restart)
        run_layout.addWidget(self.btn_minimize_tray)
        run_layout.addWidget(self.label_status, 1)

        right_layout.addWidget(run_box)

        # Command preview / log
        self.text_command = QTextEdit()
        self.text_command.setReadOnly(True)
        self.text_command.setPlaceholderText("Generated loclx command + log will appear here.")
        right_layout.addWidget(QLabel("Command preview / log"))
        right_layout.addWidget(self.text_command, 1)

        main_layout.addLayout(left_layout, 1)
        main_layout.addLayout(right_layout, 2)

        self._connect_edit_signals()
        self._create_tray_icon(icon)
        self.load_config_or_create_default()

    # ------------------------------------------------------------------
    # Tray icon handling
    # ------------------------------------------------------------------

    def _create_tray_icon(self, icon: QIcon) -> None:
        """Initialize system tray icon and menu."""
        self.tray_icon = QSystemTrayIcon(self)
        self.tray_icon.setIcon(icon)
        self.tray_icon.setToolTip("LocalXpose HTTP Tunnel Manager")
        self.tray_icon.activated.connect(self._on_tray_activated)

        menu = QMenu()

        self.action_show_hide = QAction("Show/Hide", self)
        self.action_quit = QAction("Quit", self)

        self.action_show_hide.triggered.connect(self._toggle_visibility_from_tray)
        self.action_quit.triggered.connect(self._quit_from_tray)

        menu.addAction(self.action_show_hide)
        menu.addSeparator()
        menu.addAction(self.action_quit)

        self.tray_icon.setContextMenu(menu)
        self.tray_icon.show()

    def _toggle_visibility_from_tray(self) -> None:
        if self.isVisible():
            self.hide()
        else:
            self.showNormal()
            self.raise_()
            self.activateWindow()

    def _quit_from_tray(self) -> None:
        self.save_config()
        QApplication.instance().quit()

    def _on_tray_activated(self, reason: QSystemTrayIcon.ActivationReason) -> None:
        if reason == QSystemTrayIcon.ActivationReason.Trigger:
            self._toggle_visibility_from_tray()

    def minimize_to_tray(self) -> None:
        if not self.tray_icon or not self.tray_icon.isVisible():
            return
        self.hide()
        self.tray_icon.showMessage(
            "LocalXpose Tunnel Manager",
            "Application is running in the system tray.",
            QSystemTrayIcon.MessageIcon.Information,
            3000,
        )

    # ------------------------------------------------------------------
    # Persistence
    # ------------------------------------------------------------------

    def load_config_or_create_default(self) -> None:
        cfg_path = get_config_path()
        if not cfg_path.exists():
            self.add_task()
            return

        try:
            with cfg_path.open("r", encoding="utf-8") as f:
                data = json.load(f)
        except Exception:
            self.add_task()
            return

        tunnels = data.get("tunnels") if isinstance(data, dict) else None
        if not isinstance(tunnels, list) or not tunnels:
            self.add_task()
            return

        for t_data in tunnels:
            if not isinstance(t_data, dict):
                continue
            config = TunnelConfig.from_dict(t_data)
            task = TunnelTask(config)
            self.tasks.append(task)
            item = QListWidgetItem(task.config.name)
            self.task_list.addItem(item)

        if self.tasks:
            self.task_list.setCurrentRow(0)
        else:
            self.add_task()

    def save_config(self) -> None:
        cfg_dir = get_config_dir()
        cfg_dir.mkdir(parents=True, exist_ok=True)
        cfg_path = cfg_dir / "config.json"

        tunnels_data = [asdict(task.config) for task in self.tasks]
        payload = {"tunnels": tunnels_data}
        try:
            with cfg_path.open("w", encoding="utf-8") as f:
                json.dump(payload, f, indent=2)
        except Exception:
            pass

    def closeEvent(self, event) -> None:  # type: ignore[override]
        """On window close, minimize to tray instead of quitting."""
        self.save_config()
        if hasattr(self, "tray_icon") and self.tray_icon.isVisible():
            event.ignore()
            self.minimize_to_tray()
        else:
            super().closeEvent(event)

    # ------------------------------------------------------------------
    # Task list management
    # ------------------------------------------------------------------

    def add_task(self) -> None:
        task = TunnelTask()
        self.tasks.append(task)

        item = QListWidgetItem(task.config.name)
        self.task_list.addItem(item)
        self.task_list.setCurrentItem(item)

    def duplicate_task(self) -> None:
        row = self.task_list.currentRow()
        if row < 0:
            return
        source_task = self.tasks[row]
        new_config_dict = asdict(source_task.config)
        new_config = TunnelConfig.from_dict(new_config_dict)
        new_config.name = f"{new_config.name} (copy)"
        task = TunnelTask(new_config)
        self.tasks.append(task)

        item = QListWidgetItem(task.config.name)
        self.task_list.addItem(item)
        self.task_list.setCurrentItem(item)

    def remove_task(self) -> None:
        row = self.task_list.currentRow()
        if row < 0:
            return
        task = self.tasks[row]
        if task.is_running():
            QMessageBox.warning(self, "Cannot remove", "Stop the tunnel before removing the task.")
            return
        self.tasks.pop(row)
        self.task_list.takeItem(row)

        if self.tasks:
            self.task_list.setCurrentRow(0)
        else:
            self.clear_editor()

        self.save_config()

    def on_task_selection_changed(self, row: int) -> None:
        if row < 0 or row >= len(self.tasks):
            self.clear_editor()
            return
        self.load_task_into_editor(self.tasks[row])

    # ------------------------------------------------------------------
    # Editor <-> model sync
    # ------------------------------------------------------------------

    def clear_editor(self) -> None:
        self.edit_name.clear()
        self.edit_loclx_path.setText("loclx")
        self.edit_to.clear()
        self.edit_region.clear()
        self.edit_subdomain.clear()
        self.edit_reserved_domain.clear()
        self.edit_https_to.clear()
        self.edit_crt_path.clear()
        self.edit_key_path.clear()
        self.edit_basic_auth.clear()
        self.edit_key_auth.clear()
        self.edit_rate_limit.clear()
        self.edit_prefix_path.clear()
        self.check_https_redirect.setChecked(False)
        self.edit_file_server.clear()
        self.text_ip_whitelist.clear()
        self.text_request_headers.clear()
        self.text_response_headers.clear()
        self.check_auto_restart.setChecked(False)
        self.text_command.clear()
        self.label_status.setText("Status: idle")

    def load_task_into_editor(self, task: TunnelTask) -> None:
        cfg = task.config

        self.edit_name.setText(cfg.name)
        self.edit_loclx_path.setText(cfg.loclx_path)
        self.edit_to.setText(cfg.to)
        self.edit_region.setText(cfg.region)
        self.edit_subdomain.setText(cfg.subdomain)
        self.edit_reserved_domain.setText(cfg.reserved_domain)
        self.edit_https_to.setText(cfg.https_to)
        self.edit_crt_path.setText(cfg.crt_path)
        self.edit_key_path.setText(cfg.key_path)

        self.edit_basic_auth.setText(cfg.basic_auth)
        self.edit_key_auth.setText(cfg.key_auth)
        self.edit_rate_limit.setText(cfg.rate_limit)

        self.edit_prefix_path.setText(cfg.prefix_path)
        self.check_https_redirect.setChecked(cfg.https_redirect)

        self.edit_file_server.setText(cfg.file_server)

        self.text_ip_whitelist.setPlainText("\n".join(cfg.ip_whitelist))
        self.text_request_headers.setPlainText("\n".join(cfg.request_headers))
        self.text_response_headers.setPlainText("\n".join(cfg.response_headers))

        self.check_auto_restart.setChecked(cfg.auto_restart)

        self.update_command_preview()
        self.update_status_label(task)

    def _connect_edit_signals(self) -> None:
        self.edit_name.textChanged.connect(self._on_any_field_changed)
        self.edit_loclx_path.textChanged.connect(self._on_any_field_changed)
        self.edit_to.textChanged.connect(self._on_any_field_changed)
        self.edit_region.textChanged.connect(self._on_any_field_changed)
        self.edit_subdomain.textChanged.connect(self._on_any_field_changed)
        self.edit_reserved_domain.textChanged.connect(self._on_any_field_changed)
        self.edit_https_to.textChanged.connect(self._on_any_field_changed)
        self.edit_crt_path.textChanged.connect(self._on_any_field_changed)
        self.edit_key_path.textChanged.connect(self._on_any_field_changed)
        self.edit_basic_auth.textChanged.connect(self._on_any_field_changed)
        self.edit_key_auth.textChanged.connect(self._on_any_field_changed)
        self.edit_rate_limit.textChanged.connect(self._on_any_field_changed)
        self.edit_prefix_path.textChanged.connect(self._on_any_field_changed)
        self.edit_file_server.textChanged.connect(self._on_any_field_changed)

        self.check_https_redirect.stateChanged.connect(self._on_any_field_changed)

        self.text_ip_whitelist.textChanged.connect(self._on_any_field_changed)
        self.text_request_headers.textChanged.connect(self._on_any_field_changed)
        self.text_response_headers.textChanged.connect(self._on_any_field_changed)

    def _on_any_field_changed(self) -> None:
        row = self.task_list.currentRow()
        if row < 0 or row >= len(self.tasks):
            return
        task = self.tasks[row]
        cfg = task.config

        cfg.name = self.edit_name.text().strip() or "Unnamed tunnel"
        cfg.loclx_path = self.edit_loclx_path.text().strip() or "loclx"
        cfg.to = self.edit_to.text().strip()
        cfg.region = self.edit_region.text().strip()
        cfg.subdomain = self.edit_subdomain.text().strip()
        cfg.reserved_domain = self.edit_reserved_domain.text().strip()
        cfg.https_to = self.edit_https_to.text().strip()
        cfg.crt_path = self.edit_crt_path.text().strip()
        cfg.key_path = self.edit_key_path.text().strip()

        cfg.basic_auth = self.edit_basic_auth.text().strip()
        cfg.key_auth = self.edit_key_auth.text().strip()
        cfg.rate_limit = self.edit_rate_limit.text().strip()

        cfg.prefix_path = self.edit_prefix_path.text().strip()
        cfg.https_redirect = self.check_https_redirect.isChecked()

        cfg.file_server = self.edit_file_server.text().strip()

        cfg.ip_whitelist = [
            line.strip()
            for line in self.text_ip_whitelist.toPlainText().splitlines()
            if line.strip()
        ]
        cfg.request_headers = [
            line.strip()
            for line in self.text_request_headers.toPlainText().splitlines()
            if line.strip()
        ]
        cfg.response_headers = [
            line.strip()
            for line in self.text_response_headers.toPlainText().splitlines()
            if line.strip()
        ]

        cfg.auto_restart = self.check_auto_restart.isChecked()

        item = self.task_list.item(row)
        if item is not None:
            item.setText(cfg.name)

        self.update_command_preview()
        self.save_config()

    def update_command_preview(self) -> None:
        row = self.task_list.currentRow()
        if row < 0 or row >= len(self.tasks):
            self.text_command.clear()
            return
        task = self.tasks[row]
        cmd_list = task.build_command()
        self.text_command.setPlainText(" ".join(shlex.quote(part) for part in cmd_list))

    # ------------------------------------------------------------------
    # Run control
    # ------------------------------------------------------------------

    def _start_task_process(self, task: TunnelTask) -> None:
        if task.is_running():
            return

        cmd_list = task.build_command()
        process = QProcess(self)
        process.setProgram(cmd_list[0])
        process.setArguments(cmd_list[1:])

        process.readyReadStandardOutput.connect(lambda: self._on_process_output(process))
        process.readyReadStandardError.connect(lambda: self._on_process_output(process))
        process.finished.connect(lambda code, status: self._on_process_finished(task, code, status))

        try:
            process.start()
        except Exception as exc:  # noqa: BLE001
            QMessageBox.critical(self, "Failed to start", f"Could not start loclx: {exc}")
            return

        task.process = process
        task._stopping = False
        self.update_status_label(task)

    def start_selected_tunnel(self) -> None:
        row = self.task_list.currentRow()
        if row < 0 or row >= len(self.tasks):
            return
        task = self.tasks[row]

        if task.is_running():
            QMessageBox.information(self, "Already running", "The selected tunnel is already running.")
            return

        self._start_task_process(task)

    def stop_selected_tunnel(self) -> None:
        row = self.task_list.currentRow()
        if row < 0 or row >= len(self.tasks):
            return
        task = self.tasks[row]
        if not task.is_running():
            return

        task._stopping = True
        assert task.process is not None
        task.process.terminate()
        task.process.waitForFinished(3000)
        if task.process.state() != QProcess.ProcessState.NotRunning:
            task.process.kill()
        self.update_status_label(task)

    def update_status_label(self, task: TunnelTask) -> None:
        status = "running" if task.is_running() else "idle"
        self.label_status.setText(f"Status: {status}")

    def _on_process_output(self, process: QProcess) -> None:
        data_out = bytes(process.readAllStandardOutput()).decode(errors="ignore")
        data_err = bytes(process.readAllStandardError()).decode(errors="ignore")

        text = self.text_command.toPlainText()
        if data_out:
            text += "\n[STDOUT]\n" + data_out
        if data_err:
            text += "\n[STDERR]\n" + data_err
        self.text_command.setPlainText(text)
        self.text_command.verticalScrollBar().setValue(self.text_command.verticalScrollBar().maximum())

    def _on_process_finished(self, task: TunnelTask, exit_code: int, _status: QProcess.ExitStatus) -> None:
        msg = f"Tunnel exited with code {exit_code}."
        text = self.text_command.toPlainText() + "\n" + msg
        self.text_command.setPlainText(text)
        self.text_command.verticalScrollBar().setValue(self.text_command.verticalScrollBar().maximum())

        restarting = False
        if not task._stopping and task.config.auto_restart:
            restarting = True
            self.text_command.append("[INFO] Auto-restarting tunnel...")
            self._start_task_process(task)
        else:
            task.process = None
            task._stopping = False
            self.update_status_label(task)

        if not restarting:
            task.process = None
            task._stopping = False
            self.update_status_label(task)


# ---------------------------------------------------------------------------
# Entry point
# ---------------------------------------------------------------------------


def main() -> None:
    app = QApplication(sys.argv)
    window = MainWindow()
    window.show()
    sys.exit(app.exec())


if __name__ == "__main__":
    main()
```



end