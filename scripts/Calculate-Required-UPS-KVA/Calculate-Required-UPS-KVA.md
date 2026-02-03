

---
layout: page
title: Calculate-Required-UPS-KVA
---

# Calculate-Required-UPS-KVA

```bash

# # interactive_ups_sizing_fixed.py
"""
Interactive UPS sizing calculator.
Prompts for device wattage and quantity, then calculates required kVA and battery energy.
"""

def calculate_ups(pf=0.9, efficiency=0.9, runtime_hr=1.0):
    devices = []
    print("UPS Sizing Calculator – enter each device. Press Enter on an empty name to finish.\n")

    while True:
        name = input("Enter device name (or press Enter to finish): ").strip()
        if name == "":
            break  # Stop input loop
        try:
            watts = float(input(f"  Watts required for one {name}: ").strip())
            qty = int(input(f"  Number of {name} devices: ").strip())
            devices.append((name, watts, qty))
        except ValueError:
            print("⚠ Please enter numeric values for watts and quantity.\n")

    if not devices:
        print("\nNo devices were entered. Exiting.")
        return

    # Totals
    total_watts = sum(w * q for _, w, q in devices)
    total_va = total_watts / pf
    battery_kwh = total_watts * runtime_hr / efficiency
    suggested_kva = round(total_va / 1000 + 0.5, 1)  # add margin

    # Output summary
    print("\n--- UPS Sizing Summary ---")
    for name, watts, qty in devices:
        print(f"{qty} × {name} @ {watts:.1f} W = {watts * qty:.1f} W")
    print(f"\nTotal Real Power (W) : {total_watts:.1f} W")
    print(f"Total Apparent Power : {total_va:.1f} VA")
    print(f"Battery Energy Needed: {battery_kwh:.2f} kWh for {runtime_hr} hour(s)")
    print(f"Suggested UPS Rating : {suggested_kva:.1f} kVA (with margin)")

if __name__ == "__main__":
    calculate_ups()
