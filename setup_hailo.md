To install **TAPPAS v3.26.2** with **HailoRT v4.15.0** on a Raspberry Pi 5, you need to perform a manual installation. The default `sudo apt install hailo-all` command usually installs much newer versions (e.g., v4.20+), so you must manually handle the driver and middleware to lock them to your specific requirements.

### Phase 1: Enable PCIe Gen 3.0

The Raspberry Pi 5 disables PCIe Gen 3.0 by default. To get the best performance for face recognition, you must enable it.

1. Open the configuration tool:
```bash
sudo raspi-config

```


2. Navigate to **Advanced Options** > **PCIe Speed**.
3. Select **Yes** to enable Gen 3.0 mode.
4. Finish and reboot:
```bash
sudo reboot

```



---

### Phase 2: Install HailoRT v4.15.0 and PCIe Driver

Since you need a specific version, you cannot use the standard repository. You must download the `.deb` files from the [Hailo Developer Zone](https://hailo.ai/developer-zone/) (requires a free account).

1. **Download the following files** to your Pi:
* `hailort-pcie-driver_4.15.0_all.deb`
* `hailort_4.15.0_arm64.deb`


2. **Install dependencies:**
```bash
sudo apt update
sudo apt install -y dkms build-essential linux-headers-$(uname -r)

```


3. **Install the Driver and Middleware:**
```bash
sudo dpkg -i hailort-pcie-driver_4.15.0_all.deb
sudo dpkg -i hailort_4.15.0_arm64.deb

```


4. **Verify the installation:**
```bash
hailortcli fw-control identify

```


> You should see "Firmware Version: 4.15.0" in the output.



---

### Phase 3: Install TAPPAS v3.26.2

TAPPAS v3.26.2 is an older release (December 2023). It is best installed within its own directory to manage its specific GStreamer dependencies.

1. **Clone the TAPPAS repository** and checkout the specific version:
```bash
git clone https://github.com/hailo-ai/tappas.git
cd tappas
git checkout v3.26.2

```


2. **Install TAPPAS dependencies:**
```bash
sudo apt install -y python3-gi python3-gi-cairo gir1.2-gtk-3.0 python3-virtualenv

```


3. **Run the installation script:**
```bash
./install.sh --skip-hailort

```


*(Note: We use `--skip-hailort` because we already installed the specific v4.15.0 version in Phase 2).*

---

### Phase 4: Run the Face Recognition Example

Once installed, TAPPAS provides a pre-built pipeline for face recognition.

1. **Initialize the environment:**
```bash
source $(find . -name tappas_env.sh)

```


2. **Navigate to the Face Recognition app:**
```bash
cd apps/h8/gstreamer/face_recognition

```


3. **Run the demo (using a camera input):**
```bash
./face_recognition.sh --input /dev/video0

```



### Troubleshooting Tips

* **Version Mismatch:** If `hailortcli` shows a different version than `4.15.0`, you may need to uninstall existing versions first using `sudo apt purge hailo-all`.
* **Kernel Headers:** If the PCIe driver fails to install, ensure your kernel headers are fully updated with `sudo apt full-upgrade` and another reboot.

**Would you like me to help you configure a custom GStreamer pipeline for your face recognition model?**
