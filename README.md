# ==========================================================
# TASK MANAGER – FULL FINAL BUILD (ALL CORE + POWER FEATURES)
# ==========================================================

import tkinter as tk
from tkinter import messagebox, ttk
from tkcalendar import Calendar
from datetime import datetime
import json
import os
import threading
import time
import winsound
from plyer import notification

# -------------------- SETTINGS --------------------
APP_TITLE = "Task Manager"
TASK_FILE = "tasks.json"
CHECK_INTERVAL = 20  # seconds

# -------------------- GLOBALS --------------------
tasks = []
running = True

# ==================================================
# ---------------- NOTIFICATION --------------------
# ==================================================

def send_notification(task):
    notification.notify(
        title="Task Reminder",
        message=f"[{task['priority']}] {task['text']}",
        timeout=5
    )
    winsound.Beep(1000, 500)

# ==================================================
# ---------------- REMINDER ENGINE -----------------
# ==================================================

def reminder_loop():
    while running:
        now = datetime.now()

        for task in tasks:
            if task["done"]:
                continue

            task_time = datetime.strptime(
                task["time"], "%Y-%m-%d %H:%M"
            )

            if now >= task_time:
                send_notification(task)
                task["done"] = True
                save_tasks()

        time.sleep(CHECK_INTERVAL)

# ==================================================
# ---------------- TASK ACTIONS --------------------
# ==================================================

def get_selected_datetime():
    date_str = calendar.get_date()
    hour = hour_spin.get()
    minute = minute_spin.get()
    return datetime.strptime(
        f"{date_str} {hour}:{minute}",
        "%Y-%m-%d %H:%M"
    )

def add_task():
    text = entry.get().strip()
    priority = priority_var.get()

    if not text:
        messagebox.showwarning("Empty Task", "Please enter a task.")
        return

    reminder_time = get_selected_datetime()

    task = {
        "text": text,
        "time": reminder_time.strftime("%Y-%m-%d %H:%M"),
        "priority": priority,
        "done": False
    }

    tasks.append(task)
    listbox.insert(
        tk.END,
        f"[{priority}] {text} @ {task['time']}"
    )

    save_tasks()
    entry.delete(0, tk.END)

def mark_done():
    selected = listbox.curselection()

    if not selected:
        messagebox.showinfo("No Selection", "Select a task to mark as done.")
        return

    index = selected[0]
    completed_task = tasks.pop(index)
    listbox.delete(index)

    save_tasks()

    messagebox.showinfo(
        "Task Completed",
        f"Completed: {completed_task['text']}"
    )

# ==================================================
# ---------------- SAVE / LOAD ---------------------
# ==================================================

def load_tasks():
    if not os.path.exists(TASK_FILE):
        return

    with open(TASK_FILE, "r", encoding="utf-8") as f:
        data = json.load(f)

    for task in data:
        tasks.append(task)
        listbox.insert(
            tk.END,
            f"[{task['priority']}] {task['text']} @ {task['time']}"
        )

def save_tasks():
    with open(TASK_FILE, "w", encoding="utf-8") as f:
        json.dump(tasks, f, indent=4)

def on_close():
    global running
    running = False
    save_tasks()
    root.destroy()

def send_notification(task):
    notification.notify(
        title="Task Reminder",
        message=f"[{task['priority']}] {task['text']}",
        timeout=5
    )
    try:
        winsound.PlaySound(
            "alarm.wav",
            winsound.SND_FILENAME | winsound.SND_ASYNC
        )
    except:
        pass

# ==================================================
# -------------------- UI --------------------------
# ==================================================

root = tk.Tk()
root.title(APP_TITLE)
root.geometry("460x580")
root.protocol("WM_DELETE_WINDOW", on_close)

tk.Label(
    root,
    text="Task Manager",
    font=("Segoe UI", 16, "bold")
).pack(pady=10)

# ---- Task Entry + Priority ----
top_frame = tk.Frame(root)
top_frame.pack(pady=5)

entry = tk.Entry(top_frame, width=26)
entry.grid(row=0, column=0, padx=5)

priority_var = tk.StringVar(value="Medium")
priority_menu = ttk.Combobox(
    top_frame,
    textvariable=priority_var,
    values=["High", "Medium", "Low"],
    width=8,
    state="readonly"
)
priority_menu.grid(row=0, column=1, padx=5)

tk.Button(
    top_frame,
    text="Add Task",
    command=add_task
).grid(row=0, column=2)

# ---- Calendar ----
tk.Label(root, text="Select Date").pack(pady=(10, 0))

calendar = Calendar(
    root,
    selectmode="day",
    date_pattern="yyyy-mm-dd"
)
calendar.pack(pady=5)

# ---- Time Spinner ----
time_frame = tk.Frame(root)
time_frame.pack(pady=5)

tk.Label(time_frame, text="Hour").grid(row=0, column=0)
hour_spin = tk.Spinbox(time_frame, from_=0, to=23, width=5, format="%02.0f")
hour_spin.grid(row=1, column=0, padx=5)

tk.Label(time_frame, text="Minute").grid(row=0, column=1)
minute_spin = tk.Spinbox(time_frame, from_=0, to=59, width=5, format="%02.0f")
minute_spin.grid(row=1, column=1, padx=5)

# ---- Task List ----
listbox = tk.Listbox(
    root,
    width=58,
    height=10,
    font=("Segoe UI", 10)
)
listbox.pack(pady=10)

tk.Button(
    root,
    text="Mark as Done",
    command=mark_done
).pack(pady=5)

# ==================================================
# ---------------- INIT ----------------------------
# ==================================================

load_tasks()

threading.Thread(
    target=reminder_loop,
    daemon=True
).start()

root.mainloop()
