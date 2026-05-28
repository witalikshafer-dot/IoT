
import tkinter as tk
from tkinter import ttk, messagebox, filedialog
import random, math, time, csv, json, threading
from datetime import datetime
from collections import deque

import matplotlib
matplotlib.use("TkAgg")
import matplotlib.pyplot as plt
from matplotlib.backends.backend_tkagg import FigureCanvasTkAgg
from matplotlib.figure import Figure
import numpy as np

LIGHT = dict(
    bg="#f5f4f0", surface="#ffffff", surface2="#f0eeea", surface3="#e8e6e0",
    text="#1a1a18", text2="#6b6a65", text3="#9b9a95",
    border="#d0cec8",
    ok="#3B6D11", ok_bg="#EAF3DE",
    warn="#854F0B", warn_bg="#FAEEDA",
    err="#A32D2D", err_bg="#FCEBEB",
    info="#0C447C", info_bg="#E6F1FB",
    chart_bg="#ffffff",
)
DARK = dict(
    bg="#111110", surface="#1c1c1a", surface2="#242422", surface3="#2e2e2b",
    text="#e8e6e1", text2="#888781", text3="#555450",
    border="#333330",
    ok="#97C459", ok_bg="#1a2e0a",
    warn="#EF9F27", warn_bg="#2a1e05",
    err="#F09595", err_bg="#2a0a0a",
    info="#85B7EB", info_bg="#042C53",
    chart_bg="#1c1c1a",
)

SENSOR_COLORS = ["#D85A30","#378ADD","#639922","#7F77DD",
                 "#EF9F27","#D4537E","#1D9E75","#993C1D"]

SENSORS_DEF = [
    dict(name="Температура",  location="Сервер A",  room="Серверна",
         unit="°C",  min_v=18, max_v=85, warn=70, err=80, normal=(20,45)),
    dict(name="Вологість",    location="Склад B",   room="Склад",
         unit="%",   min_v=20, max_v=95, warn=80, err=90, normal=(30,70)),
    dict(name="CO₂",          location="Офіс C",    room="Офіс",
         unit="ppm", min_v=350,max_v=2000,warn=1000,err=1500,normal=(350,800)),
    dict(name="Тиск",         location="Цех D",     room="Цех",
         unit="hPa", min_v=960,max_v=1040,warn=1025,err=1030,normal=(990,1020)),
    dict(name="Освітленість", location="Офіс C",    room="Офіс",
         unit="lx",  min_v=0,  max_v=1000,warn=800, err=950, normal=(100,600)),
    dict(name="Рух",          location="Коридор E", room="Коридор",
         unit="",    min_v=0,  max_v=1,   warn=1,   err=1,   normal=(0,0)),
    dict(name="Вібрація",     location="Цех D",     room="Цех",
         unit="g",   min_v=0,  max_v=10,  warn=5,   err=8,   normal=(0,2)),
    dict(name="Задимленість", location="Склад B",   room="Склад",
         unit="%",   min_v=0,  max_v=100, warn=15,  err=30,  normal=(0,5)),
]

USERS = {
    "admin":    dict(password="admin123", role="admin",    label="Адміністратор"),
    "operator": dict(password="oper456",  role="operator", label="Оператор"),
    "viewer":   dict(password="view789",  role="viewer",   label="Переглядач"),
}

MAX_LOG = 500
HISTORY_LEN = 60   

def rnd(a, b): return round(random.uniform(a, b), 2)
def gen_series(s, n=24):
    v = rnd(*s["normal"])
    out = []
    for _ in range(n):
        if s["unit"] == "":
            out.append(1.0 if random.random() < 0.05 else 0.0)
        else:
            v += (random.random() - 0.48) * (s["max_v"] - s["min_v"]) * 0.04
            v = max(s["min_v"], min(s["max_v"], v))
            out.append(round(v, 2))
    return out

def sensor_status(val, s):
    if s["unit"] == "":
        return "Спрацював" if val >= 1 else "Норма"
    if val >= s["err"]:  return "Критично"
    if val >= s["warn"]: return "Увага"
    return "Норма"

def status_color(val, s, theme):
    st = sensor_status(val, s)
    if st == "Критично":           return theme["err"]
    if st in ("Увага","Спрацював"): return theme["warn"]
    return theme["ok"]

class IoTApp:
    def __init__(self, root):
        self.root = root
        self.root.title("IoT Monitor Ultimate")
        self.root.geometry("1200x780")
        self.root.minsize(900, 600)

        self.dark_mode = False
        self.theme = LIGHT
        self.current_user = None
        self.session_start = None

        self.sensors = []
        for i, s in enumerate(SENSORS_DEF):
            d = dict(**s)
            d["color"] = SENSOR_COLORS[i]
            d["current"] = rnd(*s["normal"])
            d["prev"]    = d["current"]
            d["series"]  = gen_series(s, 24)
            d["history"] = deque([d["current"]] * HISTORY_LEN, maxlen=HISTORY_LEN)
            self.sensors.append(d)

        self.data_log = []
        self.access_log = []
        self.rules = []
        self.rule_log = []

        self._build_login()

    def _build_login(self):
        self.login_frame = tk.Frame(self.root)
        self.login_frame.pack(fill="both", expand=True)
        self._style_bg(self.login_frame)

        box = tk.Frame(self.login_frame, width=320)
        box.place(relx=0.5, rely=0.5, anchor="center")
        self._style_bg(box)

        tk.Label(box, text="IoT Monitor", font=("Courier New", 16, "bold")).pack(pady=(0,4))
        tk.Label(box, text="Увійдіть для доступу до системи",
                 font=("Courier New", 10)).pack(pady=(0,16))

        for lbl, attr, show in [("Логін","_login_user",""), ("Пароль","_login_pass","•")]:
            f = tk.Frame(box); f.pack(fill="x", pady=4)
            tk.Label(f, text=lbl, font=("Courier New", 10)).pack(anchor="w")
            e = tk.Entry(f, show=show, font=("Georgia", 13), relief="flat",
                         highlightthickness=1)
            e.pack(fill="x", ipady=4)
            setattr(self, attr, e)

        self._login_err = tk.Label(box, text="", font=("Courier New", 10), fg="#A32D2D")
        self._login_err.pack()

        btn = tk.Button(box, text="УВІЙТИ", font=("Courier New", 11),
                        command=self._do_login, relief="flat",
                        padx=12, pady=6)
        btn.pack(fill="x", pady=8)

        hint = "admin / admin123 — повний доступ\noperator / oper456 — оператор\nviewer / view789 — тільки перегляд"
        tk.Label(box, text=hint, font=("Courier New", 9), justify="left").pack()

        self._login_pass.bind("<Return>", lambda e: self._do_login())
        self._apply_theme_to_frame(self.login_frame)

    def _do_login(self):
        u = self._login_user.get().strip()
        p = self._login_pass.get()
        if u in USERS and USERS[u]["password"] == p:
            self.current_user = dict(username=u, **USERS[u])
            self.session_start = datetime.now()
            self._add_access_log("Вхід", "Успішна автентифікація")
            self.login_frame.destroy()
            self._build_main()
            self._start_live_update()
        else:
            self._login_err.config(text="Невірний логін або пароль")
            self._add_access_log("Невдала спроба", f"Користувач: {u}", failed=True)

    def _do_logout(self):
        self._add_access_log("Вихід", "Сесія завершена")
        self.current_user = None
        if hasattr(self, "_live_job"):
            self.root.after_cancel(self._live_job)
        self.main_frame.destroy()
        self._build_login()

    def _build_main(self):
        self.main_frame = tk.Frame(self.root)
        self.main_frame.pack(fill="both", expand=True)

        self._build_nav()

        self.content = tk.Frame(self.main_frame)
        self.content.pack(fill="both", expand=True)

        self.pages = {}
        self._build_dashboard()
        self._build_compare()
        self._build_forecast()
        self._build_automation()
        self._build_thresholds()
        self._build_export()
        self._build_access_log()

        self._show_page("dashboard")
        self._apply_theme_to_frame(self.main_frame)

    def _build_nav(self):
        nav = tk.Frame(self.main_frame, height=40)
        nav.pack(fill="x")
        nav.pack_propagate(False)
        self.nav_frame = nav

        self._nav_btns = {}
        tabs = [
            ("dashboard",   "📊 Дашборд",      True),
            ("compare",     "📋 Порівняння",    True),
            ("forecast",    "📈 Прогноз",       True),
            ("automation",  "⚡ Автоматизація", self.current_user["role"] == "admin"),
            ("thresholds",  "⚙ Пороги",        self.current_user["role"] == "admin"),
            ("export",      "💾 Експорт",       True),
            ("access_log",  "🛡 Журнал",        self.current_user["role"] == "admin"),
        ]
        for key, label, visible in tabs:
            if not visible: continue
            b = tk.Button(nav, text=label, font=("Courier New", 9),
                          relief="flat", padx=8,
                          command=lambda k=key: self._show_page(k))
            b.pack(side="left", fill="y")
            self._nav_btns[key] = b

        right = tk.Frame(nav)
        right.pack(side="right")

        role_lbl = tk.Label(right, text=self.current_user["label"],
                            font=("Courier New", 9))
        role_lbl.pack(side="left", padx=4)

        self._clock_lbl = tk.Label(right, text="", font=("Courier New", 9))
        self._clock_lbl.pack(side="left", padx=8)

        tk.Button(right, text="🌙 Тема", font=("Courier New", 9),
                  relief="flat", padx=6,
                  command=self._toggle_theme).pack(side="left", padx=2)

        tk.Button(right, text="Вийти", font=("Courier New", 9),
                  relief="flat", padx=6,
                  command=self._do_logout).pack(side="left", padx=4)

    def _show_page(self, name):
        for k, f in self.pages.items():
            f.pack_forget()
        self.pages[name].pack(fill="both", expand=True)
        self.current_page = name
        for k, b in self._nav_btns.items():
            b.config(relief="sunken" if k == name else "flat")
        if name == "dashboard":  self._render_dashboard()
        if name == "compare":    self._render_compare()
        if name == "forecast":   self._render_forecast()
        if name == "export":     self._render_export()
        if name == "access_log": self._render_access_log()

    def _build_dashboard(self):
        frame = tk.Frame(self.content)
        self.pages["dashboard"] = frame

        top = tk.Frame(frame)
        top.pack(fill="x", padx=12, pady=8)
        tk.Label(top, text="IoT — Моніторинг датчиків",
                 font=("Courier New", 13)).pack(side="left")
        self._live_lbl = tk.Label(top, text="● Живий режим",
                                  font=("Courier New", 9))
        self._live_lbl.pack(side="left", padx=12)
        tk.Button(top, text="⟳ Оновити", font=("Courier New", 9),
                  relief="flat", padx=6, command=self._refresh).pack(side="right")

        self._metrics_frame = tk.Frame(frame)
        self._metrics_frame.pack(fill="x", padx=12, pady=4)

        ctrl = tk.Frame(frame)
        ctrl.pack(fill="x", padx=12, pady=2)
        tk.Label(ctrl, text="Датчик:", font=("Courier New", 9)).pack(side="left")
        self._sensor_var = tk.StringVar()
        names = [s["name"] for s in self.sensors]
        self._sensor_cb = ttk.Combobox(ctrl, textvariable=self._sensor_var,
                                       values=names, width=16, state="readonly")
        self._sensor_cb.current(0)
        self._sensor_cb.pack(side="left", padx=4)
        self._sensor_cb.bind("<<ComboboxSelected>>", lambda e: self._render_dashboard())

        fig_frame = tk.LabelFrame(frame, text="Графік показників",
                                  font=("Courier New", 9))
        fig_frame.pack(fill="both", expand=True, padx=12, pady=4)
        self._main_fig = Figure(figsize=(8, 2.5), dpi=90)
        self._main_ax  = self._main_fig.add_subplot(111)
        self._main_canvas = FigureCanvasTkAgg(self._main_fig, fig_frame)
        self._main_canvas.get_tk_widget().pack(fill="both", expand=True)

        bot = tk.Frame(frame)
        bot.pack(fill="x", padx=12, pady=4)

        self._sensor_list_frame = tk.LabelFrame(bot, text="Статус датчиків",
                                                font=("Courier New", 9))
        self._sensor_list_frame.pack(side="left", fill="both", expand=True, padx=(0,4))

        self._alerts_frame = tk.LabelFrame(bot, text="Сповіщення",
                                           font=("Courier New", 9))
        self._alerts_frame.pack(side="left", fill="both", expand=True)

    def _render_dashboard(self):
        t = self.theme
        for w in self._metrics_frame.winfo_children():
            w.destroy()
        cols = max(1, self._metrics_frame.winfo_width() // 150) or 4
        for i, s in enumerate(self.sensors):
            card = tk.Frame(self._metrics_frame, bd=1, relief="flat",
                            bg=t["surface"], padx=8, pady=6)
            card.grid(row=i//cols, column=i%cols, padx=4, pady=4, sticky="nsew")
            st = sensor_status(s["current"], s)
            stc = status_color(s["current"], s, t)
            val_str = ("ON" if s["current"] >= 1 else "OFF") if s["unit"] == "" \
                      else str(s["current"])
            tk.Label(card, text=s["name"], font=("Courier New", 8),
                     fg=t["text2"], bg=t["surface"]).pack(anchor="w")
            tk.Label(card, text=s["location"], font=("Courier New", 7),
                     fg=t["text3"], bg=t["surface"]).pack(anchor="w")
            tk.Label(card, text=f"{val_str} {s['unit']}",
                     font=("Georgia", 16), fg=t["text"], bg=t["surface"]).pack(anchor="w")
            delta = round(s["current"] - s["prev"], 2)
            sign = "▲" if delta > 0 else ("▼" if delta < 0 else "—")
            dcol = t["ok"] if delta <= 0 else t["err"]
            tk.Label(card, text=f"{sign} {abs(delta)} {s['unit']}",
                     font=("Courier New", 8), fg=dcol, bg=t["surface"]).pack(anchor="w")
            tk.Label(card, text=st, font=("Courier New", 8),
                     fg=stc, bg=t["surface"]).pack(anchor="w")

        sel = self._sensor_var.get() or self.sensors[0]["name"]
        s = next((x for x in self.sensors if x["name"] == sel), self.sensors[0])
        ax = self._main_ax
        ax.clear()
        hist = list(s["history"])
        ax.plot(hist, color=s["color"], linewidth=1.5)
        ax.axhline(s["warn"], color=self.theme["warn"], linestyle="--",
                   linewidth=0.8, alpha=0.7, label="Увага")
        ax.axhline(s["err"],  color=self.theme["err"],  linestyle="--",
                   linewidth=0.8, alpha=0.7, label="Критично")
        ax.set_facecolor(t["chart_bg"])
        self._main_fig.patch.set_facecolor(t["chart_bg"])
        ax.tick_params(colors=t["text2"], labelsize=7)
        for spine in ax.spines.values():
            spine.set_edgecolor(t["border"])
        ax.set_title(f"{s['name']} ({s['unit']})", color=t["text2"],
                     fontsize=9, fontfamily="monospace")
        ax.legend(fontsize=7, facecolor=t["chart_bg"], labelcolor=t["text2"])
        self._main_canvas.draw()

        for w in self._sensor_list_frame.winfo_children(): w.destroy()
        for s in self.sensors:
            row = tk.Frame(self._sensor_list_frame, bg=t["surface"])
            row.pack(fill="x", pady=1)
            stc = status_color(s["current"], s, t)
            val_str = ("ON" if s["current"] >= 1 else "OFF") if s["unit"] == "" \
                      else f"{s['current']} {s['unit']}"
            tk.Label(row, text=f"■ {s['name']}",
                     font=("Courier New", 9), fg=s["color"], bg=t["surface"]).pack(side="left", padx=2)
            tk.Label(row, text=s["location"],
                     font=("Courier New", 8), fg=t["text3"], bg=t["surface"]).pack(side="left", padx=4)
            tk.Label(row, text=val_str,
                     font=("Courier New", 9), fg=t["text"], bg=t["surface"]).pack(side="right", padx=4)
            tk.Label(row, text=sensor_status(s["current"], s),
                     font=("Courier New", 8), fg=stc, bg=t["surface"]).pack(side="right", padx=2)

        for w in self._alerts_frame.winfo_children(): w.destroy()
        any_alert = False
        for s in self.sensors:
            st = sensor_status(s["current"], s)
            if st in ("Критично", "Увага", "Спрацював"):
                any_alert = True
                stc = status_color(s["current"], s, t)
                val_str = ("ON" if s["current"] >= 1 else "OFF") if s["unit"] == "" \
                          else f"{s['current']} {s['unit']}"
                msg = f"{'🔴' if st=='Критично' else '🟡'} {s['name']}: {val_str} — {st}"
                tk.Label(self._alerts_frame, text=msg,
                         font=("Courier New", 9), fg=stc,
                         bg=t["warn_bg"] if st == "Увага" else t["err_bg"],
                         anchor="w").pack(fill="x", padx=4, pady=1)
        if not any_alert:
            tk.Label(self._alerts_frame, text="✓ Всі датчики в нормі",
                     font=("Courier New", 9), fg=t["ok"],
                     bg=t["ok_bg"]).pack(fill="x", padx=4, pady=2)

    def _build_compare(self):
        frame = tk.Frame(self.content)
        self.pages["compare"] = frame

        tk.Label(frame, text="Порівняння датчиків",
                 font=("Courier New", 13)).pack(padx=12, pady=8, anchor="w")

        self._cmp_fig = Figure(figsize=(8, 3.5), dpi=90)
        self._cmp_ax  = self._cmp_fig.add_subplot(111)
        canvas = FigureCanvasTkAgg(self._cmp_fig, frame)
        canvas.get_tk_widget().pack(fill="both", expand=True, padx=12, pady=4)
        self._cmp_canvas = canvas

        self._room_frame = tk.LabelFrame(frame, text="Середнє по кімнатах",
                                         font=("Courier New", 9))
        self._room_frame.pack(fill="x", padx=12, pady=4)

    def _render_compare(self):
        t = self.theme
        ax = self._cmp_ax
        ax.clear()
        hours = list(range(24))
        for s in self.sensors:
            if s["unit"] == "": continue
            ax.plot(hours, s["series"], color=s["color"], linewidth=1.2,
                    label=s["name"], alpha=0.8)
        ax.set_facecolor(t["chart_bg"])
        self._cmp_fig.patch.set_facecolor(t["chart_bg"])
        ax.tick_params(colors=t["text2"], labelsize=7)
        for sp in ax.spines.values(): sp.set_edgecolor(t["border"])
        ax.set_xlabel("Година", color=t["text2"], fontsize=8)
        ax.legend(fontsize=7, facecolor=t["chart_bg"], labelcolor=t["text2"])
        ax.set_title("Всі датчики за 24 год (нормовані)", color=t["text2"],
                     fontsize=9, fontfamily="monospace")
        self._cmp_canvas.draw()

        for w in self._room_frame.winfo_children(): w.destroy()
        rooms = {}
        for s in self.sensors:
            if s["unit"] == "": continue
            rooms.setdefault(s["room"], []).append(s["current"])
        for room, vals in rooms.items():
            avg = round(sum(vals)/len(vals), 1)
            row = tk.Frame(self._room_frame, bg=t["surface"])
            row.pack(fill="x", pady=2, padx=4)
            tk.Label(row, text=f"{room}:", width=14, anchor="w",
                     font=("Courier New", 9), fg=t["text2"],
                     bg=t["surface"]).pack(side="left")
            tk.Label(row, text=f"{avg}",
                     font=("Courier New", 9), fg=t["text"],
                     bg=t["surface"]).pack(side="left", padx=8)

    def _build_forecast(self):
        frame = tk.Frame(self.content)
        self.pages["forecast"] = frame

        top = tk.Frame(frame)
        top.pack(fill="x", padx=12, pady=8)
        tk.Label(top, text="Прогноз на наступні години",
                 font=("Courier New", 13)).pack(side="left")

        ctrl = tk.Frame(frame)
        ctrl.pack(fill="x", padx=12, pady=2)
        tk.Label(ctrl, text="Датчик:", font=("Courier New", 9)).pack(side="left")
        self._fc_sensor_var = tk.StringVar()
        names = [s["name"] for s in self.sensors if s["unit"] != ""]
        self._fc_cb = ttk.Combobox(ctrl, textvariable=self._fc_sensor_var,
                                   values=names, width=16, state="readonly")
        self._fc_cb.current(0)
        self._fc_cb.pack(side="left", padx=4)

        tk.Label(ctrl, text="Горизонт:", font=("Courier New", 9)).pack(side="left", padx=8)
        self._fc_horizon_var = tk.StringVar(value="6")
        ttk.Combobox(ctrl, textvariable=self._fc_horizon_var,
                     values=["3","6","12","24"], width=5, state="readonly"
                     ).pack(side="left")

        tk.Label(ctrl, text="Метод:", font=("Courier New", 9)).pack(side="left", padx=8)
        self._fc_method_var = tk.StringVar(value="ma")
        ttk.Combobox(ctrl, textvariable=self._fc_method_var,
                     values=["linear","ma","exp","holt"],
                     width=8, state="readonly").pack(side="left")

        tk.Button(ctrl, text="▶ Розрахувати", font=("Courier New", 9),
                  relief="flat", padx=8, command=self._render_forecast).pack(side="left", padx=12)

        fig_frame = tk.LabelFrame(frame, text="Факт + прогноз",
                                  font=("Courier New", 9))
        fig_frame.pack(fill="both", expand=True, padx=12, pady=4)
        self._fc_fig = Figure(figsize=(8, 3), dpi=90)
        self._fc_ax  = self._fc_fig.add_subplot(111)
        self._fc_canvas = FigureCanvasTkAgg(self._fc_fig, fig_frame)
        self._fc_canvas.get_tk_widget().pack(fill="both", expand=True)

        self._fc_table_frame = tk.LabelFrame(frame, text="Прогнозні значення",
                                             font=("Courier New", 9))
        self._fc_table_frame.pack(fill="x", padx=12, pady=4)

    def _render_forecast(self):
        t = self.theme
        sel = self._fc_sensor_var.get()
        s = next((x for x in self.sensors if x["name"] == sel), self.sensors[0])
        horizon = int(self._fc_horizon_var.get())
        method  = self._fc_method_var.get()

        hist = s["series"] 
        def forecast_linear(data, n):
            x = np.arange(len(data))
            m, b = np.polyfit(x, data, 1)
            return [round(m*(len(data)+i)+b, 2) for i in range(n)]
        def forecast_ma(data, n, w=5):
            last = np.mean(data[-w:])
            return [round(last + (np.random.randn()*0.2), 2) for _ in range(n)]
        def forecast_exp(data, n, a=0.3):
            v = data[-1]
            out = []
            for _ in range(n):
                v = a*data[-1] + (1-a)*v
                out.append(round(v, 2))
            return out
        def forecast_holt(data, n, a=0.3, b=0.1):
            l, tr = data[-1], data[-1]-data[-2] if len(data)>1 else 0
            out = []
            for _ in range(n):
                l2 = a*data[-1] + (1-a)*(l+tr)
                tr = b*(l2-l) + (1-b)*tr
                l = l2
                out.append(round(l+tr, 2))
            return out

        fc_map = {"linear": forecast_linear, "ma": forecast_ma,
                  "exp": forecast_exp, "holt": forecast_holt}
        predicted = fc_map[method](hist, horizon)

        ax = self._fc_ax
        ax.clear()
        x_hist = list(range(len(hist)))
        x_fc   = list(range(len(hist), len(hist)+horizon))
        ax.plot(x_hist, hist, color=s["color"], linewidth=1.5, label="Факт")
        ax.plot(x_fc, predicted, color="#7F77DD", linewidth=1.5,
                linestyle="--", label="Прогноз")
        ax.axvline(len(hist)-1, color=t["border"], linewidth=0.8, linestyle=":")
        ax.set_facecolor(t["chart_bg"])
        self._fc_fig.patch.set_facecolor(t["chart_bg"])
        ax.tick_params(colors=t["text2"], labelsize=7)
        for sp in ax.spines.values(): sp.set_edgecolor(t["border"])
        ax.set_title(f"{s['name']} — прогноз ({method})", color=t["text2"],
                     fontsize=9, fontfamily="monospace")
        ax.legend(fontsize=7, facecolor=t["chart_bg"], labelcolor=t["text2"])
        self._fc_canvas.draw()

        for w in self._fc_table_frame.winfo_children(): w.destroy()
        header = tk.Frame(self._fc_table_frame, bg=t["surface2"])
        header.pack(fill="x")
        for h in ("Час +", "Значення", "Статус"):
            tk.Label(header, text=h, font=("Courier New", 8),
                     fg=t["text2"], bg=t["surface2"],
                     width=12).pack(side="left", padx=4)
        for i, v in enumerate(predicted):
            row = tk.Frame(self._fc_table_frame, bg=t["surface"])
            row.pack(fill="x")
            stc = status_color(v, s, t)
            for txt, w in [(f"+{i+1}г", 12),(f"{v} {s['unit']}", 12),
                           (sensor_status(v,s), 12)]:
                clr = stc if "Статус" in txt else t["text"]
                tk.Label(row, text=txt, font=("Courier New", 8),
                         fg=t["text"], bg=t["surface"], width=w).pack(side="left", padx=4)

    def _build_automation(self):
        frame = tk.Frame(self.content)
        self.pages["automation"] = frame

        tk.Label(frame, text="Сценарії автоматизації",
                 font=("Courier New", 13)).pack(padx=12, pady=8, anchor="w")

        add_frame = tk.LabelFrame(frame, text="Додати правило", font=("Courier New", 9))
        add_frame.pack(fill="x", padx=12, pady=4)

        row1 = tk.Frame(add_frame)
        row1.pack(fill="x", padx=6, pady=4)
        tk.Label(row1, text="Якщо:", font=("Courier New", 9)).pack(side="left")
        self._rule_sensor_var = tk.StringVar()
        names = [s["name"] for s in self.sensors]
        ttk.Combobox(row1, textvariable=self._rule_sensor_var, values=names,
                     width=14, state="readonly").pack(side="left", padx=4)
        self._rule_op_var = tk.StringVar(value=">")
        ttk.Combobox(row1, textvariable=self._rule_op_var,
                     values=[">","<","="], width=4, state="readonly").pack(side="left")
        self._rule_val_entry = tk.Entry(row1, width=8, font=("Georgia", 11))
        self._rule_val_entry.pack(side="left", padx=4)
        tk.Label(row1, text="→ Дія:", font=("Courier New", 9)).pack(side="left", padx=4)
        self._rule_action_var = tk.StringVar(value="notify")
        ttk.Combobox(row1, textvariable=self._rule_action_var,
                     values=["notify","log","alarm","email"],
                     width=10, state="readonly").pack(side="left")
        self._rule_msg_entry = tk.Entry(row1, width=22, font=("Georgia", 11))
        self._rule_msg_entry.insert(0, "Текст повідомлення")
        self._rule_msg_entry.pack(side="left", padx=4)
        tk.Button(row1, text="+ Додати", font=("Courier New", 9),
                  relief="flat", command=self._add_rule).pack(side="left", padx=4)

        self._rules_list_frame = tk.LabelFrame(frame, text="Активні правила",
                                               font=("Courier New", 9))
        self._rules_list_frame.pack(fill="x", padx=12, pady=4)

        self._rule_log_frame = tk.LabelFrame(frame, text="Журнал спрацювань",
                                             font=("Courier New", 9))
        self._rule_log_frame.pack(fill="both", expand=True, padx=12, pady=4)

        # treeview for rule log
        cols = ("Час","Правило","Значення","Дія")
        self._rule_tree = ttk.Treeview(self._rule_log_frame, columns=cols,
                                       show="headings", height=8)
        for c in cols:
            self._rule_tree.heading(c, text=c)
            self._rule_tree.column(c, width=120)
        self._rule_tree.pack(fill="both", expand=True)
        self._render_rules()

    def _add_rule(self):
        sensor = self._rule_sensor_var.get()
        op     = self._rule_op_var.get()
        msg    = self._rule_msg_entry.get()
        action = self._rule_action_var.get()
        try:
            val = float(self._rule_val_entry.get())
        except ValueError:
            messagebox.showerror("Помилка", "Введіть числове значення")
            return
        self.rules.append(dict(sensor=sensor, op=op, val=val,
                               action=action, msg=msg, enabled=True))
        self._render_rules()

    def _render_rules(self):
        t = self.theme
        for w in self._rules_list_frame.winfo_children(): w.destroy()
        for i, r in enumerate(self.rules):
            row = tk.Frame(self._rules_list_frame, bg=t["ok_bg"], bd=1, relief="flat")
            row.pack(fill="x", pady=2, padx=4)
            txt = f"{r['sensor']} {r['op']} {r['val']} → {r['action']}: {r['msg']}"
            tk.Label(row, text=txt, font=("Courier New", 9),
                     fg=t["ok"], bg=t["ok_bg"]).pack(side="left", padx=6, pady=3)
            tk.Button(row, text="Видалити", font=("Courier New", 8),
                      relief="flat", fg=t["err"],
                      command=lambda idx=i: self._del_rule(idx)).pack(side="right", padx=4)

    def _del_rule(self, idx):
        self.rules.pop(idx)
        self._render_rules()

    def _check_rules(self):
        for r in self.rules:
            s = next((x for x in self.sensors if x["name"] == r["sensor"]), None)
            if not s: continue
            v = s["current"]
            triggered = (r["op"] == ">" and v > r["val"]) or \
                        (r["op"] == "<" and v < r["val"]) or \
                        (r["op"] == "=" and abs(v - r["val"]) < 0.5)
            if triggered:
                self.rule_log.insert(0, dict(
                    time=datetime.now().strftime("%H:%M:%S"),
                    rule=f"{r['sensor']} {r['op']} {r['val']}",
                    value=f"{v} {s['unit']}",
                    action=r["action"]
                ))
                if len(self.rule_log) > 100:
                    self.rule_log = self.rule_log[:100]
                if r["action"] == "alarm":
                    self.root.bell()

        for row in self._rule_tree.get_children():
            self._rule_tree.delete(row)
        for row in self.rule_log[:50]:
            self._rule_tree.insert("", "end",
                values=(row["time"], row["rule"], row["value"], row["action"]))

    def _build_thresholds(self):
        frame = tk.Frame(self.content)
        self.pages["thresholds"] = frame

        top = tk.Frame(frame)
        top.pack(fill="x", padx=12, pady=8)
        tk.Label(top, text="Налаштування порогів",
                 font=("Courier New", 13)).pack(side="left")
        tk.Button(top, text="💾 Зберегти", font=("Courier New", 9),
                  relief="flat", padx=8,
                  command=self._save_thresholds).pack(side="right")

        canvas = tk.Canvas(frame)
        scrollbar = ttk.Scrollbar(frame, orient="vertical", command=canvas.yview)
        self._thresh_inner = tk.Frame(canvas)
        self._thresh_inner.bind("<Configure>",
            lambda e: canvas.configure(scrollregion=canvas.bbox("all")))
        canvas.create_window((0,0), window=self._thresh_inner, anchor="nw")
        canvas.configure(yscrollcommand=scrollbar.set)
        canvas.pack(side="left", fill="both", expand=True, padx=12)
        scrollbar.pack(side="right", fill="y")

        self._thresh_entries = {}
        cols = 3
        for i, s in enumerate(self.sensors):
            card = tk.LabelFrame(self._thresh_inner, text=s["name"],
                                 font=("Courier New", 9))
            card.grid(row=i//cols, column=i%cols, padx=8, pady=6, sticky="nw")
            entries = {}
            for field, label in [("warn","Попередження"),("err","Критично")]:
                r = tk.Frame(card)
                r.pack(fill="x", pady=2)
                tk.Label(r, text=f"{label}:", width=14, anchor="w",
                         font=("Courier New", 9)).pack(side="left")
                e = tk.Entry(r, width=8, font=("Georgia", 11))
                e.insert(0, str(s[field]))
                e.pack(side="left")
                tk.Label(r, text=s["unit"], font=("Courier New", 8)).pack(side="left", padx=2)
                entries[field] = e
            self._thresh_entries[s["name"]] = entries

    def _save_thresholds(self):
        for s in self.sensors:
            ent = self._thresh_entries.get(s["name"], {})
            for field in ("warn","err"):
                try:
                    s[field] = float(ent[field].get())
                except (ValueError, KeyError):
                    pass
        messagebox.showinfo("Пороги", "Пороги збережено!")
        self._add_access_log("Пороги", "Оновлено пороги датчиків")

    def _build_export(self):
        frame = tk.Frame(self.content)
        self.pages["export"] = frame

        top = tk.Frame(frame)
        top.pack(fill="x", padx=12, pady=8)
        tk.Label(top, text="Експорт та журнал даних",
                 font=("Courier New", 13)).pack(side="left")
        tk.Button(top, text="⬇ CSV", font=("Courier New", 9),
                  relief="flat", padx=8, command=self._export_csv).pack(side="right", padx=4)
        tk.Button(top, text="⬇ JSON", font=("Courier New", 9),
                  relief="flat", padx=8, command=self._export_json).pack(side="right", padx=4)
        tk.Button(top, text="🗑 Очистити", font=("Courier New", 9),
                  relief="flat", padx=8, command=self._clear_log).pack(side="right", padx=4)

        log_frame = tk.LabelFrame(frame, text="Журнал (останні 500 записів)",
                                  font=("Courier New", 9))
        log_frame.pack(fill="both", expand=True, padx=12, pady=4)

        cols = ("Час","Датчик","Місце","Значення","Одиниця","Статус")
        self._log_tree = ttk.Treeview(log_frame, columns=cols, show="headings", height=18)
        for c in cols:
            self._log_tree.heading(c, text=c)
            self._log_tree.column(c, width=100)
        vsb = ttk.Scrollbar(log_frame, orient="vertical", command=self._log_tree.yview)
        self._log_tree.configure(yscrollcommand=vsb.set)
        self._log_tree.pack(side="left", fill="both", expand=True)
        vsb.pack(side="right", fill="y")

    def _render_export(self):
        for row in self._log_tree.get_children():
            self._log_tree.delete(row)
        for r in self.data_log[:200]:
            self._log_tree.insert("", "end",
                values=(r["time"], r["name"], r["location"],
                        r["value"], r["unit"], r["status"]))

    def _export_csv(self):
        path = filedialog.asksaveasfilename(
            defaultextension=".csv",
            filetypes=[("CSV", "*.csv"), ("All", "*.*")],
            initialfile=f"iot_{datetime.now().strftime('%Y-%m-%d')}.csv")
        if not path: return
        with open(path, "w", newline="", encoding="utf-8-sig") as f:
            w = csv.writer(f)
            w.writerow(["Час","Датчик","Місце","Значення","Одиниця","Статус"])
            for r in self.data_log:
                w.writerow([r["time"],r["name"],r["location"],
                             r["value"],r["unit"],r["status"]])
        messagebox.showinfo("Експорт", f"CSV збережено:\n{path}")

    def _export_json(self):
        path = filedialog.asksaveasfilename(
            defaultextension=".json",
            filetypes=[("JSON","*.json"),("All","*.*")],
            initialfile=f"iot_{datetime.now().strftime('%Y-%m-%d')}.json")
        if not path: return
        with open(path, "w", encoding="utf-8") as f:
            json.dump(self.data_log, f, ensure_ascii=False, indent=2)
        messagebox.showinfo("Експорт", f"JSON збережено:\n{path}")

    def _clear_log(self):
        self.data_log = []
        self._render_export()

    def _build_access_log(self):
        frame = tk.Frame(self.content)
        self.pages["access_log"] = frame

        top = tk.Frame(frame)
        top.pack(fill="x", padx=12, pady=8)
        tk.Label(top, text="Журнал доступу",
                 font=("Courier New", 13)).pack(side="left")
        tk.Button(top, text="🗑 Очистити", font=("Courier New", 9),
                  relief="flat", padx=8,
                  command=lambda: (self.access_log.clear(), self._render_access_log())
                  ).pack(side="right")

        self._session_frame = tk.LabelFrame(frame, text="Активна сесія",
                                            font=("Courier New", 9))
        self._session_frame.pack(fill="x", padx=12, pady=4)

        log_frame = tk.LabelFrame(frame, text="Журнал подій",
                                  font=("Courier New", 9))
        log_frame.pack(fill="both", expand=True, padx=12, pady=4)

        cols = ("Час","Користувач","Роль","Подія","Деталі")
        self._alog_tree = ttk.Treeview(log_frame, columns=cols,
                                       show="headings", height=18)
        for c in cols:
            self._alog_tree.heading(c, text=c)
            self._alog_tree.column(c, width=130)
        vsb = ttk.Scrollbar(log_frame, orient="vertical", command=self._alog_tree.yview)
        self._alog_tree.configure(yscrollcommand=vsb.set)
        self._alog_tree.pack(side="left", fill="both", expand=True)
        vsb.pack(side="right", fill="y")

    def _render_access_log(self):
        t = self.theme
        for w in self._session_frame.winfo_children(): w.destroy()
        if self.current_user and self.session_start:
            dur = int((datetime.now() - self.session_start).total_seconds() // 60)
            info = (f"👤 {self.current_user['username']} "
                    f"({self.current_user['label']}) · "
                    f"Сесія: {dur} хв · "
                    f"з {self.session_start.strftime('%H:%M:%S')}")
            tk.Label(self._session_frame, text=info,
                     font=("Courier New", 9), fg=t["info"],
                     bg=t["info_bg"]).pack(fill="x", padx=6, pady=4)

        for row in self._alog_tree.get_children():
            self._alog_tree.delete(row)
        for r in self.access_log:
            self._alog_tree.insert("", "end",
                values=(r["time"], r["user"], r["role"], r["event"], r["detail"]))

    def _add_access_log(self, event, detail="", failed=False):
        self.access_log.insert(0, dict(
            time=datetime.now().strftime("%H:%M:%S"),
            user=self.current_user["username"] if self.current_user else "?",
            role=self.current_user["label"]    if self.current_user else "?",
            event=event, detail=detail, failed=failed
        ))
        if len(self.access_log) > 200:
            self.access_log = self.access_log[:200]

    def _refresh(self):
        t = self.theme
        now = datetime.now().strftime("%H:%M:%S")
        for s in self.sensors:
            s["prev"] = s["current"]
            if s["unit"] == "":
                s["current"] = 1.0 if random.random() < 0.05 else 0.0
            else:
                delta = (random.random() - 0.48) * (s["max_v"] - s["min_v"]) * 0.03
                s["current"] = round(
                    max(s["min_v"], min(s["max_v"], s["current"] + delta)), 2)
            s["history"].append(s["current"])
            self.data_log.insert(0, dict(
                time=now, name=s["name"], location=s["location"],
                value=s["current"], unit=s["unit"],
                status=sensor_status(s["current"], s)
            ))
        if len(self.data_log) > MAX_LOG:
            self.data_log = self.data_log[:MAX_LOG]

        self._check_rules()
        if hasattr(self, "current_page"):
            if self.current_page == "dashboard": self._render_dashboard()
            if self.current_page == "export":    self._render_export()

    def _start_live_update(self):
        self._update_clock()
        self._tick()

    def _tick(self):
        self._refresh()
        self._live_job = self.root.after(5000, self._tick)

    def _update_clock(self):
        now = datetime.now().strftime("%H:%M:%S")
        if hasattr(self, "_clock_lbl"):
            self._clock_lbl.config(text=now)
        self.root.after(1000, self._update_clock)

    def _toggle_theme(self):
        self.dark_mode = not self.dark_mode
        self.theme = DARK if self.dark_mode else LIGHT
        self._apply_theme_to_frame(self.main_frame)
        if hasattr(self, "current_page"):
            self._show_page(self.current_page)

    def _style_bg(self, widget):
        try:
            widget.config(bg=self.theme["bg"])
        except tk.TclError:
            pass

    def _apply_theme_to_frame(self, frame):
        t = self.theme
        def recurse(w):
            cls = w.winfo_class()
            try:
                if cls in ("Frame","Toplevel","Label","Button","Checkbutton","Radiobutton"):
                    w.config(bg=t["surface"] if cls != "Frame" else t["bg"],
                             fg=t["text"])
                if cls == "Button":
                    w.config(bg=t["surface2"], activebackground=t["surface3"])
                if cls == "LabelFrame":
                    w.config(bg=t["surface"], fg=t["text2"])
            except tk.TclError:
                pass
            for child in w.winfo_children():
                recurse(child)
        recurse(frame)
        # style ttk
        style = ttk.Style()
        style.theme_use("clam")
        style.configure("TCombobox",
                        fieldbackground=t["surface"],
                        background=t["surface"],
                        foreground=t["text"],
                        selectbackground=t["surface2"])
        style.configure("Treeview",
                        background=t["surface"],
                        fieldbackground=t["surface"],
                        foreground=t["text"],
                        rowheight=22)
        style.configure("Treeview.Heading",
                        background=t["surface2"],
                        foreground=t["text2"])
        style.map("Treeview", background=[("selected", t["info_bg"])])

if __name__ == "__main__":
    root = tk.Tk()
    root.title("IoT Monitor Ultimate")
    try:
        root.iconbitmap("")
    except Exception:
        pass
    app = IoTApp(root)
    root.mainloop()
