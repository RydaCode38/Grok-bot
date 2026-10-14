# Grok-bot
import customtkinter as ctk
import requests
import threading
import time
from datetime import datetime
import webbrowser
import sys
import os

ctk.set_appearance_mode("dark")
ctk.set_default_color_theme("dark-blue")

class GrokBotLive(ctk.CTk):
    def __init__(self):
        super().__init__()
        self.title("Grok Bot – Live GPU Night Shift")
        self.geometry("1280x900")
        self.minsize(1100, 750)
        self.configure(fg_color="#0d0d0f")

        self.budget = 1000.0
        self.profit = 0.0
        self.revenue = 0.0
        self.rent_cost = 0.0
        self.jobs_done = 0
        self.nodes_live = 0
        self.running = True
        self.offers = []

        self.build_ui()
        self.start_live_loop()

    def build_ui(self):
        # Header
        header = ctk.CTkFrame(self, fg_color="#16161a", height=90, corner_radius=12)
        header.pack(fill="x", padx=12, pady=(12, 6))
        header.pack_propagate(False)

        title_frame = ctk.CTkFrame(header, fg_color="transparent")
        title_frame.pack(side="left", padx=20, pady=10)
        ctk.CTkLabel(title_frame, text="Grok Bot", font=ctk.CTkFont(size=28, weight="bold")).pack(anchor="w")
        self.status_label = ctk.CTkLabel(title_frame, text="● LIVE · scanning markets", text_color="#00ff9d", font=ctk.CTkFont(size=13))
        self.status_label.pack(anchor="w")

        self.profit_label = ctk.CTkLabel(header, text="+$0.00", font=ctk.CTkFont(size=28, weight="bold"), text_color="#00ff9d")
        self.profit_label.pack(side="right", padx=25)

        # Agents
        agents = ctk.CTkFrame(self, fg_color="transparent")
        agents.pack(fill="x", padx=12, pady=4)
        for name, color in [("SCOUT", "#3b82f6"), ("QUANT", "#a855f7"), ("RISK", "#ef4444"), ("EXEC", "#ec4899"), ("YOU", "#eab308")]:
            f = ctk.CTkFrame(agents, fg_color="#1c1c22", width=130, height=55, corner_radius=10)
            f.pack(side="left", padx=6, pady=4)
            f.pack_propagate(False)
            ctk.CTkLabel(f, text=name, font=ctk.CTkFont(size=15, weight="bold"), text_color=color).pack(expand=True)

        # Main area
        main = ctk.CTkFrame(self, fg_color="transparent")
        main.pack(fill="both", expand=True, padx=12, pady=6)

        # Left – Live Offers
        left = ctk.CTkFrame(main, fg_color="#16161a", width=460, corner_radius=12)
        left.pack(side="left", fill="y", padx=(0, 10))
        left.pack_propagate(False)

        ctk.CTkLabel(left, text="LIVE OFFERS (echte Marktdaten)", font=ctk.CTkFont(size=15, weight="bold")).pack(pady=(12, 6))
        self.offer_box = ctk.CTkScrollableFrame(left, fg_color="#0d0d0f", corner_radius=8)
        self.offer_box.pack(fill="both", expand=True, padx=10, pady=(0, 12))

        # Right – Log
        right = ctk.CTkFrame(main, fg_color="#16161a", corner_radius=12)
        right.pack(side="left", fill="both", expand=True)

        ctk.CTkLabel(right, text="LOG CONSOLE", font=ctk.CTkFont(size=15, weight="bold")).pack(pady=(12, 6))
        self.log_box = ctk.CTkTextbox(right, fg_color="#0d0d0f", text_color="#00ff9d",
                                       font=ctk.CTkFont(family="Consolas", size=13), corner_radius=8)
        self.log_box.pack(fill="both", expand=True, padx=10, pady=(0, 12))

        # Bottom bar
        controls = ctk.CTkFrame(self, fg_color="#16161a", height=75, corner_radius=12)
        controls.pack(fill="x", padx=12, pady=(0, 12))
        controls.pack_propagate(False)

        ctk.CTkButton(controls, text="Refresh Now", command=self.manual_refresh, width=140, height=36).pack(side="left", padx=15, pady=18)
        ctk.CTkButton(controls, text="Vast.ai öffnen", command=lambda: webbrowser.open("https://cloud.vast.ai"), width=130, height=36).pack(side="left", padx=6)
        ctk.CTkButton(controls, text="RunPod öffnen", command=lambda: webbrowser.open("https://www.runpod.io/console/gpu-cloud"), width=130, height=36).pack(side="left", padx=6)

        self.stats_label = ctk.CTkLabel(controls, text="Budget: $1000 | Jobs: 0 | Nodes: 0", font=ctk.CTkFont(size=14))
        self.stats_label.pack(side="right", padx=20)

    def log(self, msg):
        ts = datetime.now().strftime("%H:%M:%S")
        line = f"[{ts}] {msg}\n"
        self.log_box.insert("end", line)
        self.log_box.see("end")

    def fetch_live_prices(self):
        offers = []
        try:
            for sku in ["h100-sxm", "h100-pcie", "rtx-4090", "a100-sxm-80gb", "h200-sxm"]:
                r = requests.get(f"https://priceofcompute.com/api/v1/prices/{sku}", timeout=10)
                if r.status_code == 200:
                    data = r.json()
                    price = data.get("prices", {}).get("on_demand", {}).get("usd_per_gpu_hr")
                    if price:
                        offers.append({
                            "gpu": sku.upper().replace("-", " "),
                            "price": float(price),
                            "provider": "Market Median",
                            "source": "priceofcompute"
                        })
            # GPU Finder
            r2 = requests.get("https://gpufinder.dev/api/v1/snapshot?gpu=h100&count=8", timeout=10)
            if r2.status_code == 200:
                data = r2.json()
                items = data if isinstance(data, list) else data.get("data", data.get("offers", []))
                for item in items[:10]:
                    price = (item.get("perGpuHourlyPrice") or item.get("pricePerGpuHourUsd") or
                             item.get("price") or item.get("wholeNodeHourlyPrice"))
                    if price:
                        try:
                            p = float(price)
                            if p > 0.1:
                                offers.append({
                                    "gpu": "H100",
                                    "price": p,
                                    "provider": str(item.get("provider", {}).get("name", item.get("provider", "GPU Finder")))[:20],
                                    "source": "gpufinder"
                                })
                        except:
                            pass
        except Exception as e:
            self.log(f"RISK · Netzwerkfehler: {str(e)[:60]}")
        return offers

    def update_offers_ui(self, offers):
        for w in self.offer_box.winfo_children():
            w.destroy()

        if not offers:
            ctk.CTkLabel(self.offer_box, text="Keine Live-Angebote gerade\n(oder API kurz down)", text_color="#888").pack(pady=30)
            return

        offers = sorted(offers, key=lambda x: x["price"])[:15]
        for o in offers:
            row = ctk.CTkFrame(self.offer_box, fg_color="#1c1c22", height=42, corner_radius=8)
            row.pack(fill="x", pady=3, padx=4)
            row.pack_propagate(False)

            ctk.CTkLabel(row, text=o["gpu"][:18], width=140, anchor="w", font=ctk.CTkFont(size=13)).pack(side="left", padx=10)
            ctk.CTkLabel(row, text=f"${o['price']:.2f}/h", text_color="#00ff9d", width=85, font=ctk.CTkFont(size=13, weight="bold")).pack(side="left")
            ctk.CTkLabel(row, text=o["provider"][:16], text_color="#999", width=120, anchor="w").pack(side="left")

            def make_cmd(offer=o):
                return lambda: self.approve_offer(offer)
            ctk.CTkButton(row, text="Approve", width=85, height=28, fg_color="#166534", hover_color="#15803d",
                          command=make_cmd()).pack(side="right", padx=8)

    def approve_offer(self, offer):
        # Simuliere 4–8 Stunden 8x Node
        hours = 5
        gpus = 8
        cost = offer["price"] * gpus * hours * 0.4   # konservativ
        if cost > (self.budget - self.rent_cost):
            self.log("RISK · DENIED – Budget würde überschritten")
            return

        self.rent_cost += cost
        self.nodes_live += 1
        self.jobs_done += 4
        earned = cost * 1.75
        self.revenue += earned
        self.profit = self.revenue - self.rent_cost

        self.log(f"YOU · APPROVED {offer['gpu']} @ ${offer['price']:.2f}/h ({offer['provider']})")
        self.log(f"EXEC · Node gerentet · erwartet +${earned:.2f} Revenue")
        self.update_stats()

    def update_stats(self):
        txt = f"+${self.profit:.2f}" if self.profit >= 0 else f"-${abs(self.profit):.2f}"
        self.profit_label.configure(text=txt)
        self.profit_label.configure(text_color="#00ff9d" if self.profit >= 0 else "#ef4444")
        remaining = self.budget - self.rent_cost
        self.stats_label.configure(text=f"Budget left: ${remaining:.0f}  |  Jobs: {self.jobs_done}  |  Nodes: {self.nodes_live}")

    def live_loop(self):
        while self.running:
            self.log("SCOUT · scanning markets…")
            offers = self.fetch_live_prices()
            self.after(0, lambda o=offers: self.update_offers_ui(o))
            if offers:
                cheapest = min(offers, key=lambda x: x["price"])
                self.log(f"QUANT · best offer: {cheapest['gpu']} @ ${cheapest['price']:.2f}/h")
                self.log("RISK · stopline ON – warte auf YOUR Approve")
            else:
                self.log("RISK · aktuell keine sauberen Offers")
            time.sleep(75)

    def start_live_loop(self):
        self.log("Grok Bot gestartet · stopline ON · Human Approval required")
        self.log("Live-Daten: priceofcompute.com + gpufinder.dev")
        self.log("────────────────────────────────────────")
        t = threading.Thread(target=self.live_loop, daemon=True)
        t.start()

    def manual_refresh(self):
        self.log("Manual refresh…")
        offers = self.fetch_live_prices()
        self.update_offers_ui(offers)

    def on_closing(self):
        self.running = False
        self.destroy()

if __name__ == "__main__":
    app = GrokBotLive()
    app.protocol("WM_DELETE_WINDOW", app.on_closing)
    app.mainloop()
