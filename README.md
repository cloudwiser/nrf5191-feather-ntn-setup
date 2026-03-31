# nRF9151 Feather GEO-NB NTN Setup Guide

- This guide covers the complete steps required to go from a new nRF9151 Feather board to a working GEO-NB NTN (Non-Terrestrial Network) environment over the Skylo-Viasat-Inmarsat network using VS Code and the CircuitDojo ncs-serial-modem application.
- Massive kudos to Jared [jaredwolff](https://github.com/jaredwolff) for designing, developing and supporting what is a perfect, small-form factor, nRF5191-based module for product prototyping

---

## Prerequisites

- nRF9151 Feather board (A1A silicon)
- NTN-enabled SIM (I use Monogoto but there are alternate NTN-enabled SIMs from other MVNOs)
- Wideband antenna supporting GNSS L1 plus NTN L/S-band (and any requisite LTE TN bands if required) - see next section
- Rust toolchain installed (`rustup`)
- VS Code with nRF Connect extension installed
- probe-rs v0.31.0 or later installed (`cargo install probe-rs-tools`)

---

## GNSS/GEO-NTN Antenna Selection

- The nRF9151 Feather combines the GNSS and LTE/NTN antenna feeds so selection of a suitable wideband antenna that covers the required bands is required. This invariably needs to include the GNSS L-band, the NTN L and/or S-bands plus any required terrestrial LTE bands
- This selection is arguably the most critical elememt in your system : the 23dBm transmit power from the NB-IoT UE can severely challenge the link budget for a reliable GEO-NB NTN attach given a range of factors including obscured LoS (line-of-sight) to the GEO spacecraft, the UE latitude and hence the look angle to the GEO spacecraft and also whether the UE is operating in motion or not
- Assuming you are using a Skylo-supported SIM, you will need an antenna that supports the L-band (Global) aka n255 so:
  - DL: 1525–1559 MHz
  - UL: 1626.5–1660.5 MHz
- ...and/or S-band (Europe) aka n256 so:
  - DL: 2170–2200 MHz
  - UL: 1980–2010 MHz
- Skylo operate their own RAN and core network but the spectrum, space and ground segment access are provided to Skylo by Viasat-Inmarsat, Ligado and TerreStar for n255 and Echostar Mobile for n256
- GPS L1 sits at 1575.42 MHz so close to n255 however an antenna that covers 1559–1610 MHz would bring in the other GNSS constellations i.e. GLONASS, Beidou and Galilleo as well. However this assumes Nordic will add support for these GNSS constellations in future versions of the nRF5191 firmware and hardware
- [TODO]

---

## 1. Install NRF Connect Toolchain and SDK

- Install [nRF Connect for Desktop](https://www.nordicsemi.com/Products/Development-tools/nRF-Connect-for-Desktop)
- Open **Toolchain Manager**
- Install **NCS v3.2.4** toolchain and SDK
- Toolchain installs to: `/<path_to>/ncs/toolchains/<hash>`
- SDK installs to: `/<path_to>/ncs/v3.2.4`

---

## 2. Build the Modem Updater

```bash
cd /<path_to>/GitHub
cargo install --git https://github.com/circuitdojo/modem_updater.git
```

---

## 3. Flash NTN Firmware

Download `mfw_nrf9151-ntn_1.0.0-1.alpha.zip` from [Nordic's website](https://www.nordicsemi.com/Products/Development-hardware/nRF9151-SMA-DK/Download#infotabs) under **nRF9151 SiP NTN firmware**.

> **Note:** Run `verify` only *after* a successful `program` — not before. Running `verify` against a different firmware variant than what is currently flashed will return a NACK error.

```bash
updater program /<path_to>/Downloads/mfw_nrf9151-ntn_1.0.0-1.alpha.zip
updater verify /<path_to>/Downloads/mfw_nrf9151-ntn_1.0.0-1.alpha.zip
```

---

## 4. Clone and initialise the ncs-serial-modem Workspace

```bash
cd /<path_to>/ncs
git clone https://github.com/circuitdojo/ncs-serial-modem.git
cd ncs-serial-modem
unset ZEPHYR_BASE
west init -l /<path_to>/ncs/ncs-serial-modem
west update
```

> **Note:** `unset ZEPHYR_BASE` is required if VS Code has set it in the environment, otherwise `west init` will fail with "already initialized" error.

---

## 5. Configure VS Code Board Roots

```bash
mkdir -p /<path_to>/ncs/ncs-serial-modem/.vscode
cat > /<path_to>/ncs/ncs-serial-modem/.vscode/settings.json << 'EOF'
{
    "nrf-connect.boardRoots": [
        "/<path_to>/ncs/nfed"
    ]
}
EOF
```

> After saving, reload VS Code: `Ctrl+Shift+P` → **Developer: Reload Window**.

---

## 6. Build the App in VS Code

- Select `Open an exisitng application` in the VSC nRF Connect Welcome panel
- Select`/<path_to>/ncs/ncs-serial-modem/app` in the folder dialog
- In the nRF Connect Applications panel in VSC select `Add build configuration`
- Set the following:
  - **Board:** `circuitdojo_feather_nrf9151@3/nrf9151/ns`
  - **SDK:** v3.2.4
  - **Toolchain:** v3.2.4
  - **Application:** `app`
  - **Build system:** sysbuild
- Click **Gemerate and Build**

---

## 7. Flash the App

```bash
probe-rs download --chip nRF9151_xxAA --binary-format hex \
  /<path_to>/ncs/ncs-serial-modem/app/build/merged.hex --allow-erase-all
probe-rs reset --chip nRF9151_xxAA
```

> **Note:** The `--allow-erase-all` flag is required if the core is locked. This is normal after a fresh flash or failed flash attempt.

---

## 8. Connect a Serial Terminal

> I tend to switch between `tio` and the Serial Terminal tool in the `nRF Connect Desktop` for AT comms. If you are using `tio` or similar, then you may need to add a local echo to see your typed commands. On `tio` this is with the `--local-echo` parameter:

```bash
tio --local-echo /dev/ttyACM0
```

> However the `nRF Connect Desktop` Serial Terminal has other nice options such as command history that can be recalled as well as the ability to save command history to a file

---

## 9. Verify NTN Firmware

```text
AT+CGMR
```

> Expected response: `mfw_nrf9151-ntn_1.0.0-1.alpha`

---

## 10. Acquire GNSS Fix

- **Important:** GNSS must be acquired *before* switching to NTN system mode. Run this sequence first if you want a live GPS fix rather than manually entering coordinates.
- **Note:** you'll need an external antenna but that will be required for the NTN modem anyway as per above

```text
AT+CFUN=4
AT%XSYSTEMMODE=0,0,1,0,0
AT+CFUN=31
AT#XGNSS=1,0,0,0
```

> The wait for a fix response can take several minutes on cold start but should eventually return:

```text
#XGNSS: <lat>,<lon>,<alt>,<accuracy>,<speed>,<heading>,"<datetime>"
```

> Note the coordinates, then stop GNSS:

```text
AT#XGNSS=0
```

---

## 11. NTN Connection Sequence

- **Note:** Band n23 = Canada, band n255 = L-band Global, band n256 = S-band Europe (Skylo).
- For best results with a single patch antenna, consider restricting to band n255 only (`AT%XBANDLOCK=2,,"255"`) given n255 is closest to the L1 and L5 bands used by GNSS
- Replace `<lat>`, `<lon>`, `<alt>` with coordinates from the GNSS fix above, or enter known static coordinates.
- Each value must be a separate quoted string.

```text
AT+CFUN=4
AT%XSYSTEMMODE=0,0,0,0,1
AT%XBANDLOCK=2,,"255"
AT%LOCATION=2,"<lat>","<lon>","<alt>",0,0
AT+CEREG=5
AT+CNEC=24
AT+CSCON=3
AT%MDMEV=2
AT+CFUN=1
```

> Expected connection indications:

```text
%MDMEV: SEARCH STATUS 1
+CEREG: 2,"<tac>","<ci>",14
%MDMEV: PRACH CE-LEVEL 0
+CSCON: 1,7,4
%MDMEV: SEARCH STATUS 2
+CEREG: 5,"<tac>","<ci>",14,,,"11100000","00111000"
```

- The duration for the attach and registration following the ```AT+CFUN=1``` command can be highly variable given the nature of the signalling path from the UE to the RAN via GEO to the (cloud-hosted) EPC/5G Core...and back.
- FWIW I have observed typical durations of between 50 and 60 secs for this sequence using monogoto aka Skylo aka Viasat-Inmarsat in Northern Europe via I4F4 (Alphasat)

---

## 12. Monitor Connection Status

```text
AT+CESQ
AT%XMONITOR
```

> `AT+CESQ` returns 6 values — the last two are RSRQ and RSRP. Higher values indicate better signal quality.

---

## 13. Send UDP Test Payload

```text
AT#XSOCKET=1,2,0
AT#XCONNECT="<your_UDP_server_IP_addr>",<port_num>
AT#XSEND="Hello Feather"
AT#XSOCKET=0
```

- Expected responses: `#XSOCKET:0,2,17`, `#XCONNECT:1`, `#XSEND:13`, `#XSOCKET:0,"closed"`
- If you are using Monogoto, you can sign up and use the ubidots service to send UDP test messages although note that the message payload needs to be correctly formatted and converted to hex prior to sending - see [Monogoto : Integrate IoT data with Ubidots](https://docs.monogoto.io/developer/cloud-integrations/ubidots)

---

## 14. Reset Device

```text
AT#XRESET
```

> **Note:** `AT+CFUN=15` does not appear to be supported in the NTN alpha firmware and/or the serial terminal app. Use `AT#XRESET` instead.

---

## 15. Switch Back to Standard LTE (When Needed)

```text
AT+CFUN=0
AT%XSYSTEMMODE=1,1,0,0,0
AT+CFUN=1
```

---

## 16. AT Command Observations

- For some reason `AT+CPIN?` returns ERROR — possibly due to lack of correct configuration
- As mentiioned `AT+CFUN=15` returns ERROR — use `AT#XRESET` instead
- `AT#XGPS` (referenced in the Monogoto docs for the nRF9151 SMA DK) is not available — use `AT#XGNSS` (ncs-serial-modem app command) instead
- `AT%LOCATION=1` can be used for automatic modem-requested location updates — the modem sends `%LOCATION: <accuracy>[,<time>]` notifications whenever a new fix is required for the TA pre-compensation i.e. the modem expects a new GNSS location position to `<accuracy>` within the next `<time>` seconds. Note this is not needed if your `validity=0` i.e. your device is stationary but it is required if the device is mobile (or you are powering the feather down between sessions)

---

## Reference Links

- [CircuitDojo ncs-serial-modem](https://github.com/circuitdojo/ncs-serial-modem)
- [CircuitDojo modem updater](https://github.com/circuitdojo/modem_updater)
- [Monogoto NTN NRF9151 Guide](https://docs.monogoto.io/ntn-satellite-networks/ntn-certified-devices/ntn-certified-modules/nordic-nrf9151-satellite-ntn-network)
- [CircuitDojo nRF9151 Feather docs](https://docs.circuitdojo.com/nrf9151-feather/nrf9151-feather/updating-modem-firmware.html)
- [Nordic nRF9151 NTN AT Commands v0.4](https://files.gitbook.com/v0/b/gitbook-x-prod.appspot.com/o/spaces%2F-M0mPxGpottOEfcucXOR%2Fuploads%2F5bzFMG17YlF3WpnTCW2r%2Fmfw_nrf9151-ntn_at_commands_v0.4.pdf)

---

> Issues and pull requests welcome!

Nick Hall

satsoft.space

21st March 2026
