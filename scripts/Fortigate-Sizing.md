def fortigate_bandwidth_sizing():
    print("=== FortiGate Bandwidth Sizing Calculator ===\n")

    # Input values
    users = int(input("Enter number of concurrent users: "))
    per_user_bw = float(input("Enter average bandwidth per user (Mbps) [e.g. 8.71]: "))
    safety_margin_percent = float(input("Enter safety margin percentage (e.g. 10 for 10%): "))
    existing_internet_bw = float(input("Enter existing Internet bandwidth (Mbps): "))

    # Convert safety margin to multiplier
    safety_margin = 1 + (safety_margin_percent / 100)

    # Calculation
    user_usage_with_margin = users * per_user_bw * safety_margin
    total_required_bw = user_usage_with_margin + existing_internet_bw

    # Output
    print("\n=== Calculation Result ===")
    print(f"Users bandwidth (with safety margin): {user_usage_with_margin:.2f} Mbps")
    print(f"Existing Internet bandwidth: {existing_internet_bw:.2f} Mbps")
    print("-------------------------------------------")
    print(f"Total required bandwidth: {total_required_bw:.2f} Mbps")

    # Optional FortiGate sizing hint
    print("\n=== Sizing Hint ===")
    if total_required_bw < 500:
        print("Suggested FortiGate class: Entry / Small Branch (e.g. FG-60F)")
    elif total_required_bw < 1000:
        print("Suggested FortiGate class: Mid-range (e.g. FG-100F / 200F)")
    else:
        print("Suggested FortiGate class: Enterprise (e.g. FG-400F and above)")


if __name__ == "__main__":
    fortigate_bandwidth_sizing()
