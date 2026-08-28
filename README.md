# Qualcommax PPE Builder

### OpenWrt image builder for the Xiaomi AX3600 — PPE (Packet Processing Engine) hardware offload on the upstream EDMA drivers

[![Build](https://img.shields.io/github/actions/workflow/status/Payti/Qualcommax_PPE_Builder_Xiaomi_AX3600_eko.one.pl_packages/build-ppe.yml?branch=main&style=flat-square&logo=github&label=Build)](https://github.com/Payti/Qualcommax_PPE_Builder_Xiaomi_AX3600_eko.one.pl_packages/actions/workflows/build-ppe.yml)
[![Lint](https://img.shields.io/github/actions/workflow/status/Payti/Qualcommax_PPE_Builder_Xiaomi_AX3600_eko.one.pl_packages/lint.yml?branch=main&style=flat-square&logo=github&label=Lint)](https://github.com/Payti/Qualcommax_PPE_Builder_Xiaomi_AX3600_eko.one.pl_packages/actions/workflows/lint.yml)
[![License](https://img.shields.io/github/license/Payti/Qualcommax_PPE_Builder_Xiaomi_AX3600_eko.one.pl_packages?style=flat-square&label=License)](LICENSE)
[![Last Commit](https://img.shields.io/github/last-commit/Payti/Qualcommax_PPE_Builder_Xiaomi_AX3600_eko.one.pl_packages?style=flat-square&label=Last%20Commit)](https://github.com/Payti/Qualcommax_PPE_Builder_Xiaomi_AX3600_eko.one.pl_packages/commits/main)

A GitHub Actions pipeline that builds one OpenWrt image for the **Xiaomi
AX3600**: Qualcomm PPE hardware offload running on OpenWrt main's **upstream
`qca_edma` / `qca_ppe` ethernet drivers**
([PR #22381](https://github.com/openwrt/openwrt/pull/22381)) — not the vendor
`qca-nss-dp` / `qca-ssdk` stack every other NSS build uses. Built from
[openwrt-nss-edma](https://github.com/JuliusBairaktaris/openwrt-nss-edma)

---

## Use it

Grab the `*-sysupgrade.bin` from the newest `ppe-*`
[release](https://github.com/Payti/Qualcommax_PPE_Builder_Xiaomi_AX3600_eko.one.pl_packages/releases)
and flash it:

```sh
sysupgrade -n /tmp/ppe-openwrt-qualcommax-ipq807x-xiaomi_ax3600-squashfs-sysupgrade.bin
```

Releases from this repository target the Xiaomi AX3600 only. The image is
intended for an existing OpenWrt installation; install OpenWrt first when
the router is still running the stock Xiaomi firmware. Release assets are
named with the `ppe-` prefix and include the upstream commit used for the
build in their notes.

Or via LuCI: **System → Backup / Flash Firmware**, upload, uncheck "Keep
settings" for a first-time flash. Coming from stock Xiaomi firmware? Install
OpenWrt first via the [official guide](https://openwrt.org/toh/xiaomi/ax3600).

Wi-Fi is **enabled out of the box**: SSID `OpenWrt`, WPA2/WPA3 (`sae-mixed`),
password `openwrt-ppe`. The password is public in this repo — **change it on
first login** (LuCI → Network → Wireless).

Remember turn on 'Hardware Flow Offloading' for maximum Gigabit speed and lower CPU usage.

```sh
uci set firewall.@defaults[0].flow_offloading='1
uci set firewall.@defaults[0].flow_offloading_hw='1'
uci commit firewall
/etc/init.d/firewall restart
```

---

## What ships by default

The `ppe` image enables the full offload stack plus a hardened, batteries-
included desktop-router config:

| Area | What's on |
|---|---|
| **Wi-Fi** | ath11k NSS offload (wifili) on both radios (`CONFIG_ATH11K_NSS_SUPPORT`); enabled by default (SSID `OpenWrt`, WPA2/WPA3, password `openwrt-ppe` — change it) |
| **Security** | OpenSSH only (post-quantum KEX, AEAD/ETM, RSA ≥ 3072), `PKG_*` hardening (ASLR/PIE, stack protector, FORTIFY_3, RELRO, seccomp), WAN DROP + BCP38, HTTPS redirect, OQS provider in OpenSSL |
| **Toolchain** | GCC 15 + Graphite, Binutils 2.46, Mold linker, LTO, `-mcpu=cortex-a53+crc+crypto`; ccache off |
| **Userland** | LuCI (SSL), `htop`, `iperf3`, `curl`, BBR |

Toolchain and package pins live in
[`devices/xiaomi_ax3600/config`](devices/xiaomi_ax3600/config).

---

## Build it yourself

Everything is parameterized in the `env:` block of
[`.github/workflows/build-ppe.yml`](.github/workflows/build-ppe.yml) — fork the repo,
edit, and the pipeline builds on push. Or build locally:

```sh
git clone https://github.com/JuliusBairaktaris/openwrt-nss-edma openwrt
git clone https://github.com/Payti/Qualcommax_PPE_Builder_Xiaomi_AX3600_eko.one.pl_packages builder
cd openwrt
OPENWRT_DIR="$PWD" BUILDER_REPO="../builder" \
  DEVICE=xiaomi_ax3600 VARIANT=ppe \
  bash ../builder/scripts/prepare-build.sh
make -j"$(nproc)"
```

The few remaining overlay files (the disabled SQM template, wireless defaults, SSH config)
are under `devices/xiaomi_ax3600/files*/` — copy them into the image with a
`files/` directory or the builder pipeline. See [`docs/CUSTOMIZE.md`](docs/CUSTOMIZE.md)
for the full customization guide and [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md)
for how the pipeline works.

### Repo layout

```
devices/xiaomi_ax3600/
  config                 # the .config (target, toolchain, hardening, packages)
  files/                 # base rootfs overlay (sshd_config, QoL uci-defaults)
patches/                 # directory to load custom firmware patches
screenshots/             # screenshots this working build
scripts/                 # check-updates, prepare-build, prune-releases (tested, linted)
docs/                    # CUSTOMIZE.md, ARCHITECTURE.md
.github/workflows/       # build-ppe.yml (check → build → prune), lint.yml
```

The pipeline runs `check → build → prune`: `check` resolves the upstream PPE
ref to a SHA and skips a scheduled build when nothing changed; `build` applies
the config + overlays, compiles, and publishes a release; `prune` keeps the
newest `KEEP` releases. Builds are uncached (fresh runner, reproducible
`SOURCE_DATE_EPOCH`) and the pipeline is linted (`actionlint`, `shellcheck`,
`yamllint`) on every PR.

---

## Contributing

Issues and PRs welcome — see [`CONTRIBUTING.md`](CONTRIBUTING.md).

## Acknowledgements

- **[JuliusBairaktaris](https://github.com/JuliusBairaktaris)** — the
  [openwrt-nss-edma](https://github.com/JuliusBairaktaris/openwrt-nss-edma) this stack builds on
- **[Ansuel (Christian Marangi)](https://github.com/Ansuel)** — the
  [EDMA rework](https://github.com/openwrt/openwrt/pull/22381) this stack builds on
- **[qosmio](https://github.com/qosmio)** — NSS development, the
  [openwrt-ipq](https://github.com/qosmio/openwrt-ipq) tree, and the Wi-Fi
  offload patch lineage
- **[rodriguezst](https://github.com/rodriguezst)** — original
  [ipq807x-openwrt-builder](https://github.com/rodriguezst/ipq807x-openwrt-builder)
- **OpenWrt community** — the
  [IPQ807x NSS Build thread](https://forum.openwrt.org/t/ipq807x-nss-build/148529)

## License

[GPL-2.0](LICENSE), consistent with OpenWrt.
