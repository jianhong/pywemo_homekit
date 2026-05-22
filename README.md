# pywemo_homekit

Changing a Wemo device's Wi-Fi network and extracting its HomeKit setup code using PyWemo involves a two-phase architecture.

PyWemo can be used as a powerful alternative to manage Wemo devices within HomeKit, especially since Belkin officially discontinued cloud support for many Wemo products on January 31, 2026. 

Because a Wemo device cannot change its Wi-Fi credentials while operating on your main home network, you must force it to broadcast its own configuration hotspot (usually named "WeMo.XXXX.XXX"), connect your computer directly to that hotspot, and provision it using a single script. 

## Phase 1: Disconnect and Broadcast (Manual Steps)
You must break the old Wi-Fi loop so the device broadcasts its temporary configuration network:

   1. Unplug the Wemo device.
   2. Press and hold its physical power button while plugging it back into the wall outlet.
   3. Keep holding the button for roughly 5 seconds until the light starts flashing orange/white rapidly, then release it.
   4. Go to your computer's Wi-Fi settings and connect to the broadcasted "WeMo" network (e.g., WeMo.Mini.123).
   5. Uncheck the auto-rejoin from your wifi.

------------------------------
## Phase 2: The Full Provisioning and Code Retrieval Script
Once your computer is connected directly to the Wemo hotspot, using pyWemo programmatically detect the Wemo device on the hotspot, extract its HomeKit Code, flush old HomeKit pairing errors, and push the new Wi-Fi credentials.

### Prerequisites

* Your Wemo device must be turned on and connected to the same Wi-Fi network subnet as your computer (What you did in Phase 1).
* Python 3 and PyWemo must be installed on your system.

### Step-by-Step Process

   1. Install PyWemo: Open your terminal or command prompt and install the library if you haven't already:

```{sh}
pip install pywemo
```

   2. Open Python: Launch an interactive Python shell:

```{sh}
   python
```

   3. Discover and Query the Device: Copy and paste the following Python code line-by-line. Replace '10.22.22.1' with the actual local IP address of your specific Wemo device:

```{python}
import time
import pywemo

# ==========================================
# CONFIGURATION: ENTER YOUR IP OF WEMO AP AND NEW WI-FI HERE
# ==========================================
ip_address = "10.22.22.1"
NEW_SSID = "Your_New_WiFi_Name"
NEW_PASSWORD = "Your_WiFi_Password"
# ==========================================

print("Scanning for the Wemo provisioning access point...")
# Wemo devices in AP mode default to the base IP 10.22.22.1
try:
    url = pywemo.setup_url_for_address(ip_address)
    # Loop with a 1-second pause to prevent crashing
    while url is None:
        print(f"Waiting for device at {ip_address}...")
        time.sleep(1) 
        url = pywemo.setup_url_for_address(ip_address)
    device = pywemo.discovery.device_from_description(url)
except Exception:
    print("Trying alternative discovery method...")
    devices = pywemo.discover_devices()
    if not devices:
        raise RuntimeError("Could not find the Wemo device. Check your Wi-Fi network connection.")
    device = devices[0]

print(f"Connected to Device: {device.name}")
print("-" * 50)

# STEP 1: EXTRACT HOMEKIT SETUP CODE
try:
    print("Attempting to query HomeKit setup metadata...")
    hk_info = device.basicevent.GetHKSetupInfo()
    
    if hk_info and 'HKSetupCode' in hk_info:
        print(f"\n[SUCCESS] HomeKit Setup Code: {hk_info['HKSetupCode']}")
        print(f"[SUCCESS] HomeKit Setup Key:  {hk_info['HKSetupKey']}\n")
    else:
        print("\n[WARNING] Device responded but HomeKit setup code wasn't found in metadata.")
except Exception as e:
    print(f"\n[ERROR] HomeKit metadata query failed: {e}")
    print("Your specific model might require manual pairing or a firmware fallback.\n")

# STEP 2: STABILIZE AND PREPARE HOMEKIT PAIRING ENGINE
try:
    print("Purging cached metadata configurations and resetting HomeKit broadcast flags...")
    device.basicevent.removeHomekitData()
    device.basicevent.setHKSetupState(HKSetupDone='0')
    print("[SUCCESS] HomeKit broadcast parameters initialized successfully.")
except Exception as e:
    print(f"[INFO] Bypassing HomeKit pairing engine reset: {e}")

print("-" * 50)

# STEP 3: RENEW AND PROVISION NEW WI-FI CREDENTIALS
try:
    print(f"Injecting new network credentials for network: '{NEW_SSID}'...")
    print("The device will immediately drop its hotspot connection and join your home network.")
    
    # Send Wi-Fi credentials to the device locally
    device.setup(ssid=NEW_SSID, password=NEW_PASSWORD)
    print("\n[COMPLETE] Credentials pushed. Monitor the physical LED status indicator.")
    print("Wait 60-90 seconds for it to associate with your primary access point.")
except Exception as e:
    print(f"\n[FATAL] Failed to register Wi-Fi configuration profile: {e}")
```

Note: If the port 49153 fails to connect, try omitting it or using pywemo's auto-discovery via devices = pywemo.discover_devices() to find the correct local address.

```{python}
## step a.
import pywemo
url = pywemo.setup_url_for_address("10.22.22.1")
print(url) # if NULL here, retry it multiple times

## step b.
device = pywemo.discovery.device_from_description(url)
print(device)
## it returns something like <WeMo Switch "Wemo Mini">

## step c.
## Query the internal UPnP service for HomeKit details
hk_info = device.basicevent.GetHKSetupInfo()
# Print the setup data
print(hk_info)

## step d.
## update the 
device.setup(ssid=NEW_SSID, password=NEW_PASSWORD)
## it should return ('1', 'success') or flush the errors

## and then you can repeat the step b for next hardware.
```

* HKSetupCode: This is your secret 8-digit pairing code. Enter this exact number (e.g., 888-88-888) manually into the Apple Home app under "Add Accessory" > "More Options".
* HKSetupKey: This is the raw URL string that a physical QR setup sticker would encode. You can ignore this unless you plan to generate a brand new scannable HomeKit QR code payload.


------------------------------
## Step 3: Map Into Apple Home

* Change your computer's Wi-Fi network connection back to your home Wi-Fi network.
* Open the Apple Home app on your iPhone, iPad, or Mac.
* Select Add Accessory -> More Options -> My Accessory Isn't Shown Here / Enter Code.
* Type in the 8-digit HKSetupCode returned in the terminal window above.

