**System Optimization Guide: ZRAM & EarlyOOM Configuration**  
This documentation covers the end-to-end setup, configuration, tuning, and troubleshooting of ZRAM and EarlyOOM on Linux, optimized specifically for mid-spec systems like the Dell Latitude E6540 running Linux Mint (Cinnamon).  
![](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAnEAAAACCAYAAAA3pIp+AAAABmJLR0QA/wD/AP+gvaeTAAAACXBIWXMAAA7EAAAOxAGVKw4bAAAAMUlEQVR4nO3WAQkAIBAEsBPMYs4PZhMDWMAA5njYUmxU1UqyAwBAF2cmeZE4AIBO7gentgXapSWpbgAAAABJRU5ErkJggg==)  
**Table of Contents**  
1. [Architectural Overview](#anchor-1 "#anchor-1")  
2. [Step-by-Step Installation](#anchor-2 "#anchor-2")  
3. [Configuring ZRAM](#anchor-3 "#anchor-3")  
4. [Configuring EarlyOOM](#anchor-4 "#anchor-4")  
5. [Applying Changes & Verifying System Status](#anchor-5 "#anchor-5")  
6. [Safe Crisis Simulation (Testing Your Setup)](#anchor-6 "#anchor-6")  
![](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAnEAAAACCAYAAAA3pIp+AAAABmJLR0QA/wD/AP+gvaeTAAAACXBIWXMAAA7EAAAOxAGVKw4bAAAANElEQVR4nO3OUQmAABBAsSdYxKYXx1gmEBOIFfwTYUuwZWa2ag8AgL841uquzq8nAAC8dj05WgYLQTzjnAAAAABJRU5ErkJggg==)  
**1. Architectural Overview**  
When system memory (RAM) is exhausted, traditional Linux environments rely on disk-based Swap space (SSD/HDD). This setup introduces two operational pain points:  
- **Disk Thrashing:** Physical storage drives are orders of magnitude slower than RAM. Moving memory pages to disk locks up the I/O bus, freezing the user interface (mouse cursor stutters, display freezes).  
- **Delayed OOM Response:** The native Linux kernel Out-Of-Memory (OOM) killer triggers only during absolute resource exhaustion, frequently waiting minutes before terminating the offending application.  
This configuration resolves these issues using a two-tier defense perimeter:  
1. **ZRAM (First Line of Defense):** Creates a virtual, compressed swap block device entirely inside physical memory. Idle or cold background pages are compressed on-the-fly, expanding memory capacity by up to 2.5× to 3× without disk overhead or hardware wear.  
2. **EarlyOOM (Second Line of Defense):** A lightweight user-space daemon that polls /proc/meminfo up to 10 times per second. It interceptively terminates the single largest memory-hogging process before a full kernel panic or desktop lockup occurs.  
![](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAnEAAAACCAYAAAA3pIp+AAAABmJLR0QA/wD/AP+gvaeTAAAACXBIWXMAAA7EAAAOxAGVKw4bAAAANUlEQVR4nO3OQQmAABRAsSd4NIGJjPWxpgGsYQVvImwJtszMXp0BAPAX91pt1fH1BACA164HhZwEOFrXVOsAAAAASUVORK5CYII=)  
**2. Step-by-Step Installation**  
Execute the appropriate command for your Linux distribution to install both background daemons:  
**For Ubuntu, Linux Mint, Pop!_OS, or Debian derivatives:**  
sudo apt update && sudo apt install -y earlyoom zram-tools  
   
**For Fedora or RHEL derivatives:**  
*Note: Fedora includes ZRAM configured out of the box.*  
sudo dnf install earlyoom  
 sudo systemctl enable --now earlyoom  
   
**For Arch Linux:**  
sudo pacman -S earlyoom systemd-zram-generator  
   
![](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAnEAAAACCAYAAAA3pIp+AAAABmJLR0QA/wD/AP+gvaeTAAAACXBIWXMAAA7EAAAOxAGVKw4bAAAANUlEQVR4nO3OMQ2AABAAsSNBCUpfD6ZYGZDAgAU2QtIq6DIzW7UHAMBfHGt1V+fXEwAAXrseHCoGAe/SKtAAAAAASUVORK5CYII=)  
**3. Configuring ZRAM**  
With zram-tools installed, the compression layer deploys automatically upon boot. For a typical 8GB configuration, the utility structures a ~3.8GB virtual block device using fast compression algorithms like lz4 or zstd.  
**Optimizing Kernel Swappiness**  
For physical disk swap, lowering swappiness is standard practice. **For ZRAM setups, you must invert this behavior.** Forcing the kernel to aggressively utilize compressed virtual swap leaves raw physical RAM open for high-priority foreground applications.  
1. Create an explicit kernel configuration override file:  
2. sudo nano /etc/sysctl.d/99-zram.conf  
   
3. Populate the file with the optimized priority flag:  
4. vm.swappiness = 150  
   
5. Save the file (Ctrl+O, Enter) and exit (Ctrl+X). Apply the new kernel rule immediately:  
6. sudo sysctl --system  
   
![](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAnEAAAACCAYAAAA3pIp+AAAABmJLR0QA/wD/AP+gvaeTAAAACXBIWXMAAA7EAAAOxAGVKw4bAAAANklEQVR4nO3OQQmAABRAsScYxpg/h5VMYARvRrCCNxG2BFtmZquOAAD4i3Ot7mr/egIAwGvXA224BcUMk6pDAAAAAElFTkSuQmCC)  
**4. Configuring EarlyOOM**  
EarlyOOM must be tuned carefully when combined with ZRAM. If the swap eviction limit is set too conservatively (e.g., -s 50), EarlyOOM will kill apps prematurely when your ZRAM pool is merely half full with normal compressed data.  
**Modifying the Operational Arguments**  
1. Open the primary configuration file:  
2. sudo nano /etc/default/earlyoom  
   
3. Strip out duplicate active configuration variables and define an optimized, short-form single string at the bottom of the file:  
4. EARLYOOM_ARGS="-r 3600 -m 10 -s 10 --avoid '^(cinnamon|Xorg|Xwayland|systemd|init)$'"  
   
**Operational Flag Breakdown:**  
- -r 3600: Restricts regular logging intervals to once an hour to keep system logs clean.  
- -m 10: Triggers an application SIGTERM when available physical RAM drops to 10% or lower.  
- -s 10: Prevents early termination by allowing the ZRAM swap cache to fill to 90% before acting.  
- --avoid '^(...) $': Explicitly hard-locks critical core infrastructure—like the Linux Mint Cinnamon Desktop environment (cinnamon) and display server managers (Xorg, Xwayland)—preventing them from being picked as termination targets.  
![](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAnEAAAACCAYAAAA3pIp+AAAABmJLR0QA/wD/AP+gvaeTAAAACXBIWXMAAA7EAAAOxAGVKw4bAAAAM0lEQVR4nO3OUQmAQBBAwSdcjsu6HYxoDsEK/okwk2COmdnVGQAAf3GtalX76wkAAK/dDxFWBDkFf6+SAAAAAElFTkSuQmCC)  
**5. Applying Changes & Verifying System Status**  
After saving your file, clear out systemd's default execution limits and reload the runtime parameters to avoid operational errors (status=13 or loop failures).  
# Clear any existing service failure flags  
 sudo systemctl reset-failed earlyoom  
   
 # Restart the daemon with your new configurations  
 sudo systemctl restart earlyoom  
   
**Verifying Active Status**  
Execute the system health checker:  
sudo systemctl status earlyoom  
   
Ensure the engine displays an active flag matching this format:  
● earlyoom.service - Early OOM Daemon  
      Active: active (running)  
   
**Evaluating Live ZRAM Compression Stats**  
Analyze your live memory compaction metrics with:  
zramctl  
   
This prints your live translation table:  
- **DISKSIZE**: Total virtual allocation available to the swap pool.  
- **DATA**: Uncompressed size of memory contents currently housed in ZRAM.  
- **COMPR**: Actual physical memory foot-print consumed after compression.  
*Calculation Example:* If your table displays DATA 1.9G and COMPR 708.2M, your processor is maintaining a **2.68:1 memory compression ratio**.  
![](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAnEAAAACCAYAAAA3pIp+AAAABmJLR0QA/wD/AP+gvaeTAAAACXBIWXMAAA7EAAAOxAGVKw4bAAAALUlEQVR4nO3OQQ0AIAwEsAMlSJ0UrOFkGngRklZBR1WtJDsAAPzizNcDAADuNcKwAyU+nb+5AAAAAElFTkSuQmCC)  
**6. Safe Crisis Simulation (Testing Your Setup~Not Advised)**  
You can verify your configuration safely by generating an artificial memory expansion loop. This ensures EarlyOOM steps in exactly as intended without freezing your interface.  
1. Open a clean terminal window.  
2. Fire the memory-buffer simulator:  
3. tail /dev/zero  
   
4. Watch the terminal output. Within moments of hitting your 10% memory cushion limit, EarlyOOM will isolate the process and kill it instantly, printing a safe closure message:  
5. Killed  
   
Your desktop session remains completely uninterrupted, confirming your system memory shield is working perfectly.  
