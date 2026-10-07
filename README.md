# PVE NVIDIA GPU Dashboard Mod

Adds **NVIDIA GPU** monitoring to the Proxmox VE node summary page, next to the CPU, RAM and swap graphs.

Script: `pve-nvidia-dashboard.sh` (originally `pve-gpu-dashboard-mod_v3.0.sh`, v3.0)

Based on `pve-mod-gui-nvidia.sh` by Meliox. License: MIT.

## What you get

**Node Summary → Status panel**, for each GPU:

- VRAM usage bar, titled with the GPU name, driver version and CUDA version
- One metrics line: `GPU: 35% | Temp: 52°C | Power: 70/250W | Fan: 40%`
  - Temperature turns yellow at 70 °C and red at 85 °C (158 °F and 185 °F when using Fahrenheit)
  - Fan speed is hidden for passively cooled cards
  - Power shows `draw/limit` when the card reports a limit

**Node Summary → graphs (below Network traffic)**, for each GPU:

- Temperature
- VRAM usage
- Power draw

Graph history is collected in the browser and stored in `localStorage`. It survives page refreshes but is specific to each browser.

## Requirements

- Proxmox VE 8.x or 9.x
- NVIDIA driver installed on the **PVE host**, with `nvidia-smi` working
- Root access and `perl`

Check first:

```bash
nvidia-smi
```

## Install

```bash
chmod +x pve-nvidia-dashboard.sh
./pve-nvidia-dashboard.sh install
```

The installer asks two questions:

1. **Temperature unit**: Celsius (default) or Fahrenheit.
2. **History duration** kept in the browser:

| Choice | Duration | Interval | Points | Approx. size per GPU |
| --- | --- | --- | --- | --- |
| 1 (default) | 24 hours | 1 minute | 1,440 | ~144 KB |
| 2 | 1 week | 1 minute | 10,080 | ~1 MB |
| 3 | 30 days | 5 minutes | 8,640 | ~864 KB |

When it finishes, hard-refresh the Proxmox page (**Ctrl+Shift+R**).

If the mod is already installed, `install` stops and asks you to uninstall first.

## Uninstall

```bash
./pve-nvidia-dashboard.sh uninstall
```

This restores the latest backups of the two patched files and restarts `pveproxy`. Hard-refresh the browser afterwards.

If other PVE mods are detected, such as the sensors or the original NVIDIA mod, the script warns you. Restoring the backup also removes any mod installed after this one, and asks you to confirm before continuing.

## How it works

- **`/usr/share/perl5/PVE/API2/Nodes.pm`** is patched so that every call to the node status API (`/nodes/<node>/status`) also runs `nvidia-smi --query-gpu=...`. It adds the driver and CUDA versions, and per GPU `gpu<N>_name`, `gpu<N>_temp`, `gpu<N>_util`, `gpu<N>_mem_util`, `gpu<N>_mem_used`, `gpu<N>_mem_total`, `gpu<N>_power_draw`, `gpu<N>_power_limit`, `gpu<N>_fan` and `gpu<N>_metrics`.
- **`/usr/share/pve-manager/js/pvemanagerlib.js`** is patched to add the status items after the SWAP item. It also adds a history store that polls the same API from the browser, plus the three graphs per GPU after the Network traffic graph.

GPU count, names, driver and CUDA versions are read at install time. If you add or remove a GPU, uninstall and install again.

### Files touched or created

| Path | Purpose |
| --- | --- |
| `/usr/share/perl5/PVE/API2/Nodes.pm` | Patched: GPU values added to node status |
| `/usr/share/pve-manager/js/pvemanagerlib.js` | Patched: status items and graphs |
| `~/PVE-GPU-DASHBOARD/` | Timestamped backups of the two patched files |

## Configuration

Edit the top of the script before installing:

```bash
TEMP_WARNING=70     # warning threshold (°C)
TEMP_CRITICAL=85    # critical threshold (°C)
BACKUP_DIR=""       # empty = ~/PVE-GPU-DASHBOARD; a custom directory must already exist
```

## Troubleshooting

**"nvidia-smi is not installed or not in PATH".** Install the NVIDIA driver on the host first.

**"GPU dashboard mod is already installed".** Run `uninstall`, then `install`.

**Nothing changed in the web UI.** Hard-refresh with Ctrl+Shift+R. Check `systemctl status pveproxy`.

**CUDA version shows `N/A`.** It is read from the `nvidia-smi` header, with `nvcc` as a fallback. `N/A` is cosmetic and does not affect the graphs.

**After a Proxmox update, the GPU items disappear.** Updating `pve-manager` replaces the patched files. Run `install` again.

## Safety

This mod edits Proxmox system files. The script backs them up first and verifies each backup. You can also restore the originals at any time with `apt install --reinstall pve-manager`.

## See also

`pve-intel-flex-dashboard.sh` provides the same dashboard for Intel Flex GPUs. See `README-intel-flex.md`.
