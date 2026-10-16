#!/usr/bin/env bash
###
# rescue-kexec.sh (3.00)
#
# Boots a running Ubuntu host into an in-RAM ephemeral rescue environment
# via kexec, so its disks are completely unmounted and can be imaged
# remotely. No PXE/BIOS support needed, and as of 3.00 no MAAS instance
# needed either.
#
# What 3.00 dropped vs the 2.x line:
#   - No rack controller. Boot artifacts come from the public Ubuntu
#     image stream (images.maas.io), resolved with sstream-query from
#     the standard `simplestreams` package rather than by scraping a
#     per-node grub config.
#   - No node registration, rescue-mode arming, MAC lookup, or MAAS
#     preseed/OAuth. SSH access comes from our own injected cloud-init.
#   - Networking is taken from this host's live config and handed to the
#     ephemeral kernel; MAAS is not involved in it.
#
# Known limitation when NOT stacking: the stock initrd cannot fetch the
# squashfs over https, so root=squash: is emitted as plain http. Kernel
# and initrd themselves are still downloaded over https by this script.
#
# --stack-squashfs avoids that entirely: the squashfs is downloaded here
# over https and appended to the initrd as one extra cpio segment, along
# with a small local-top script that loop-mounts it. The cmdline then
# points root= straight at the loop device, so the stock initramfs-tools
# mountroot path does the rest. Nothing is fetched at boot.
#
# That extra segment also replaces /scripts/local-top/ORDER with one that
# runs only our script. For a rescue boot that is what we want: it stops
# lvm, mdadm, cryptroot and rooturl from running, so nothing goes looking
# at the local disks we are about to image.
#
# Network note: cloud-init gets its network config from /run/net-*.conf,
# written by the initrd's init-bottom/cloud-initramfs-dyn-netconf, which
# fires on `ip=` being present rather than on the root type. That holds
# whether or not anything fetches over the network, so stacking does not
# break it. If you ever supply a hand-built --initrd, make sure
# cloud-initramfs-dyn-netconf is installed in the build environment, or
# that handoff silently disappears and the rescue env falls back to DHCP.
#
# Usage: sudo ./rescue-kexec.sh [--ssh-key <authorized_keys_file>] \
#                               [--user-data <file>] \
#                               [--release <name>] [--stream <name>] \
#                               [--image-source <url>] \
#                               [--kernel-flavour <ga|hwe>] \
#                               [--arch <amd64|arm64|...>] \
#                               [--dns <ip>] \
#                               [--cmdline <extra args>]... \
#                               [--network-config auto|<file>]
# At least one of --ssh-key or --user-data is required.
###
set -euo pipefail

###
# --- Defaults: all overridable via CLI flags ---
###
RELEASE_DEFAULT="noble"
STREAM_DEFAULT="stable"
KERNEL_FLAVOUR_DEFAULT="ga"
DNS_DEFAULT="1.1.1.1"
IMAGE_SOURCE_BASE="https://images.maas.io/ephemeral-v3"
CLOUD_IMAGE_SOURCE="https://cloud-images.ubuntu.com"

# --remount-local defaults: reboot into the running kernel and the local
# root, with a tmpfs overlay on top so the disk goes read-only.
LOCAL_KERNEL_DEFAULT="/boot/vmlinuz"
LOCAL_INITRD_DEFAULT="/boot/initrd.img"
OVERLAY_SIZE_DEFAULT="4G"

# The kernel keeps at most COMMAND_LINE_SIZE bytes of cmdline, NUL
# included: 2048 on x86 and arm64. Anything longer is silently cut, which
# loses the tail -- BOOTIF=, console=, and any cc:/end_cc pair split by
# the cut, so cloud-init would drop all of its config.
CMDLINE_MAX=2047

# Console login for the rescue environment's default user. Hashed before
# it goes anywhere near the kernel cmdline.
DEFAULT_USER="ubuntu"
PASSWORD_DEFAULT="ubuntu"

###
# --- Optional configuration ---
###
WORKDIR="/tmp/rescue-kexec"
CONSOLE_ARGS="console=ttyS0,115200n8 console=ttyS1,115200n8 console=tty0"

# Where the squashfs lands inside the stacked initramfs, and therefore
# what rescue.squashfs= points at.
SQUASHFS_INITRAMFS_NAME="squashfs"

# Name of the local-top script we inject to loop-mount it.
RESCUE_SCRIPT_NAME="rescue-squashfs"

# Fixed loop device: root= on the cmdline has to name it up front, so
# picking one deterministically is simpler than losetup -f.
RESCUE_LOOP_DEV="/dev/loop0"

# Used to verify the signed stream index when present. If missing, the
# unsigned index.json is used instead and a warning is printed.
KEYRING_FILE="/usr/share/keyrings/ubuntu-cloudimage-keyring.gpg"

main() {
  parse_args "$@"
  check_prerequisites

  if [[ "${REMOUNT_LOCAL}" == true ]]; then
    do_remount_local
    return
  fi

  detect_arch
  detect_network_config

  NETWORK_CONFIG_PARAM=""
  if [[ "${NETWORK_MODE}" != ip ]]; then
    build_network_config_param
  fi

  CC_CONTENT=""
  if [[ -n "${USER_DATA_FILE}" ]]; then
    build_user_data_cc_block
    CC_CONTENT="${USER_DATA_CC_BLOCK}"
  fi
  if [[ "${#SSH_KEY_ARGS[@]}" -gt 0 ]]; then
    parse_authorized_keys "${SSH_KEY_ARGS[@]}"
    if [[ -n "${USER_DATA_FILE}" ]]; then
      build_ssh_only_cc_block
    else
      build_full_cc_block
    fi
    CC_CONTENT="${CC_CONTENT} ${SSH_CC_BLOCK}"
  fi
  if wants_password_block; then
    build_password_cc_block
    CC_CONTENT="${CC_CONTENT} ${PASSWORD_CC_BLOCK}"
  fi
  CC_CONTENT="$(tr -s ' ' <<<"${CC_CONTENT}" | sed -E 's/^ +| +$//g')"
  readonly CC_CONTENT

  mkdir -p "${WORKDIR}"
  resolve_image_urls
  download_kernel_initrd
  if [[ "${STACK_SQUASHFS}" == true ]]; then
    download_squashfs
  fi
  if [[ "${IMAGE_PROVIDER}" == cloud-image ]]; then
    verify_cloud_image_downloads
  fi
  if [[ "${STACK_SQUASHFS}" == true ]]; then
    build_stacked_initrd
  fi
  build_cmdline
  load_kexec
  confirm_and_execute
}

usage() {
  cat >&2 << 'EOF'
Usage: sudo ./rescue-kexec.sh [OPTIONS]

Two modes.

--remount-local: kexec back into the RUNNING kernel and the local root,
adding a tmpfs overlay so the disk is no longer written to. Fast, needs
no network and no image, but the disk is still mounted (read-only under
the overlay), so it is not equivalent to the ephemeral mode below.

  --remount-local         use this mode
  --overlay-size <size>   tmpfs overlay size    (default: 4G)
  --kernel <path>         kernel to use         (default: /boot/vmlinuz)
  --initrd <path>         initrd to use         (default: /boot/initrd.img)
  --cmdline <args>        appended to the current cmdline; repeatable

Otherwise: boot an ephemeral Ubuntu image, leaving the local disks
completely untouched and unmounted so they can be imaged.

  --ssh-key <file|key>    authorized_keys file or a literal public key;
                          injected via cloud-init so you can log in to the
                          rescue env. Repeatable.
  --password <pw>         password for the default user, console login
                          only; SSH stays key-only    (default: ubuntu)
  --user-data <file>      full cloud-init user-data, used as-is
  --release <name>        Ubuntu release        (default: noble)
  --stream <name>         stable | candidate | cloud-image  (default: stable)
                          cloud-image uses cloud-images.ubuntu.com: VM-oriented,
                          ga kernel only, always stacked, checksum-verified
  --image-source <url>    full stream base URL; overrides --stream
  --kernel-flavour <f>    ga | hwe              (default: ga)
  --hwe                   shorthand for --kernel-flavour hwe
  --arch <name>           override auto-detected arch
  --kernel <url|path>     use this kernel instead of resolving one
  --initrd <url|path>     use this initrd instead of resolving one
  --squashfs <url>        use this squashfs URL instead of resolving one
  --dns <ip>              DNS for the rescue env (default: 1.1.1.1)
  --stack-squashfs        download the squashfs here and append it to the
                          initrd, booting it from RAM with no fetch at boot
  --cmdline <args>        appended to the kernel cmdline as-is; repeatable
  --network-config <v>    network config for the rescue env (see NETWORK)
                          auto | ip | <netplan file>
                          (default: auto when stacked, ip otherwise)

At least one of --ssh-key or --user-data is required (ephemeral mode only).
Images are resolved with sstream-query (package: simplestreams) when it
is installed, otherwise by reading the published directory index.
--kernel and --initrd accept a local path as well as a URL.

NETWORK
  auto (default when stacked)
    Stacked: the RUNNING network is rendered into a netplan: every
    interface with a global address (ethernet, bond, bridge, vlan) and
    everything below it, with static addresses, main-table routes and
    the DNS servers in use; NICs matched by permanent MAC. It is a
    snapshot of what works now, not of config files, so it also covers
    Debian/Proxmox (no netplan) and live changes. ip=/BOOTIF= are left
    out. Needs jq. Shown before the confirm prompt, kept in
    <workdir>/runtime-net/, and schema-checked if netplan is installed.
      - tap/veth bridge ports and types netplan cannot express (tun,
        wireguard, ...) are left out with a warning; if the default
        route needs one, it refuses.
      - Interfaces off the default-route path, or currently down, are
        optional: the boot does not wait for them.
      - Policy routing (ip rule, other tables) is not rendered (warned).
      - --dns, if given, replaces the discovered DNS servers.
    Fetch mode, given explicitly: ip= plus a static netplan for the
    default-route NIC only, since its initramfs must reach the network.

  ip
    ip=/BOOTIF= only; the initramfs configures the default-route NIC
    and cloud-init derives the rest. The proven path of 3.70 and
    earlier; the default in fetch mode. Single untagged NIC only.

  <file>
    Your netplan (version 1 or 2; top-level network: optional) for
    anything auto cannot do. Implies --stack-squashfs; ip=/BOOTIF= are
    left out. Name it ./auto or ./ip if it really is called that.

  auto and <file> travel gzipped on the cmdline, limit 2047 bytes
  (checked).

HOW THE STACKED BOOT WORKS (--stack-squashfs)
  1. The kernel unpacks the initrd, including the appended squashfs,
     into RAM. Nothing is fetched over the network.
  2. Initramfs udev loads drivers from the initrd's modules.
  3. An injected local-top script attaches /squashfs to /dev/loop0;
     initramfs-tools mounts it as the root (root=/dev/loop0).
  4. overlayroot puts a tmpfs overlay on it, so the root is writable.
  5. switch_root into systemd inside the squashfs.
  6. udev loads the NIC driver; the interface appears.
  7. cloud-init reads the network config (network-config=, or the
     initramfs state from ip=) and writes /etc/netplan/50-cloud-init.yaml.
  8. netplan / systemd-networkd bring the network up.
  9. cloud-init applies hostname, SSH keys and password; sshd starts.
  Nothing in this flow mounts the local disks. Still check with lsblk,
  pvs and /proc/mdstat before imaging: the image's own udev rules could
  activate LVM or md arrays found on them.
EOF
  exit 1
}

parse_args() {
  RELEASE="${RELEASE_DEFAULT}"
  STREAM="${STREAM_DEFAULT}"
  KERNEL_FLAVOUR="${KERNEL_FLAVOUR_DEFAULT}"
  DNS_SERVER="${DNS_DEFAULT}"
  DNS_EXPLICIT=false
  IMAGE_SOURCE=""
  ARCH_OVERRIDE=""
  KERNEL_OVERRIDE=""
  INITRD_OVERRIDE=""
  SQUASHFS_OVERRIDE=""
  SSH_KEY_ARGS=()
  USER_DATA_FILE=""
  EXTRA_CMDLINE_ARGS=()
  NETWORK_CONFIG_SOURCE=""
  STACK_SQUASHFS=false
  REMOUNT_LOCAL=false
  OVERLAY_SIZE="${OVERLAY_SIZE_DEFAULT}"
  PASSWORD="${PASSWORD_DEFAULT}"
  PASSWORD_EXPLICIT=false

  while [[ $# -gt 0 ]]; do
    case "$1" in
      --ssh-key)        [[ $# -ge 2 ]] || usage; SSH_KEY_ARGS+=("$2"); shift 2 ;;
      --ssh-key=*)      SSH_KEY_ARGS+=("${1#*=}");                    shift   ;;
      --user-data)      [[ $# -ge 2 ]] || usage; USER_DATA_FILE="$2"; shift 2 ;;
      --user-data=*)    USER_DATA_FILE="${1#*=}";                     shift   ;;
      --release)        [[ $# -ge 2 ]] || usage; RELEASE="$2";        shift 2 ;;
      --release=*)      RELEASE="${1#*=}";                            shift   ;;
      --stream)         [[ $# -ge 2 ]] || usage; STREAM="$2";         shift 2 ;;
      --stream=*)       STREAM="${1#*=}";                             shift   ;;
      --image-source)   [[ $# -ge 2 ]] || usage; IMAGE_SOURCE="$2";   shift 2 ;;
      --image-source=*) IMAGE_SOURCE="${1#*=}";                       shift   ;;
      --kernel-flavour) [[ $# -ge 2 ]] || usage; KERNEL_FLAVOUR="$2"; shift 2 ;;
      --kernel-flavour=*) KERNEL_FLAVOUR="${1#*=}";                   shift   ;;
      --hwe)            KERNEL_FLAVOUR="hwe";                         shift   ;;
      --arch)           [[ $# -ge 2 ]] || usage; ARCH_OVERRIDE="$2";  shift 2 ;;
      --arch=*)         ARCH_OVERRIDE="${1#*=}";                      shift   ;;
      --kernel)         [[ $# -ge 2 ]] || usage; KERNEL_OVERRIDE="$2";   shift 2 ;;
      --kernel=*)       KERNEL_OVERRIDE="${1#*=}";                       shift   ;;
      --initrd)         [[ $# -ge 2 ]] || usage; INITRD_OVERRIDE="$2";   shift 2 ;;
      --initrd=*)       INITRD_OVERRIDE="${1#*=}";                       shift   ;;
      --squashfs)       [[ $# -ge 2 ]] || usage; SQUASHFS_OVERRIDE="$2"; shift 2 ;;
      --squashfs=*)     SQUASHFS_OVERRIDE="${1#*=}";                     shift   ;;
      --dns)            [[ $# -ge 2 ]] || usage; DNS_SERVER="$2"; DNS_EXPLICIT=true; shift 2 ;;
      --dns=*)          DNS_SERVER="${1#*=}"; DNS_EXPLICIT=true;                     shift   ;;
      --stack-squashfs) STACK_SQUASHFS=true;                          shift   ;;
      --remount-local)  REMOUNT_LOCAL=true;                           shift   ;;
      --password)       [[ $# -ge 2 ]] || usage; PASSWORD="$2"; PASSWORD_EXPLICIT=true; shift 2 ;;
      --password=*)     PASSWORD="${1#*=}"; PASSWORD_EXPLICIT=true;   shift   ;;
      --overlay-size)   [[ $# -ge 2 ]] || usage; OVERLAY_SIZE="$2";   shift 2 ;;
      --overlay-size=*) OVERLAY_SIZE="${1#*=}";                       shift   ;;
      --cmdline)        [[ $# -ge 2 ]] || usage; EXTRA_CMDLINE_ARGS+=("$2"); shift 2 ;;
      --cmdline=*)      EXTRA_CMDLINE_ARGS+=("${1#*=}");              shift   ;;
      --network-config)   [[ $# -ge 2 ]] || usage; NETWORK_CONFIG_SOURCE="$2"; shift 2 ;;
      --network-config=*) NETWORK_CONFIG_SOURCE="${1#*=}";            shift   ;;
      -h|--help)        usage ;;
      *) echo "ERROR: unknown argument: $1" >&2; usage ;;
    esac
  done

  if [[ "${REMOUNT_LOCAL}" == true ]]; then
    if [[ "${STACK_SQUASHFS}" == true ]]; then
      echo "ERROR: --remount-local and --stack-squashfs are different modes." >&2
      usage
    fi
    if [[ "${#SSH_KEY_ARGS[@]}" -gt 0 || -n "${USER_DATA_FILE}" \
          || "${PASSWORD_EXPLICIT}" == true || -n "${NETWORK_CONFIG_SOURCE}" ]]; then
      echo "ERROR: --ssh-key / --user-data / --password / --network-config do not" >&2
      echo "       apply to --remount-local:" >&2
      echo "       it reboots into your existing root, with your existing users." >&2
      usage
    fi
  elif [[ "${#SSH_KEY_ARGS[@]}" -eq 0 && -z "${USER_DATA_FILE}" ]]; then
    echo "ERROR: at least one of --ssh-key or --user-data is required." >&2
    usage
  fi

  if [[ -z "${PASSWORD}" ]]; then
    echo "ERROR: --password must not be empty." >&2
    usage
  fi

  readonly REMOUNT_LOCAL OVERLAY_SIZE PASSWORD PASSWORD_EXPLICIT

  # --image-source wins over the --stream shorthand.
  # Two providers with different layouts: the MAAS ephemeral tree
  # (stable/candidate), and the public cloud images.
  if [[ "${STREAM}" == cloud-image ]]; then
    IMAGE_PROVIDER="cloud-image"
    if [[ "${KERNEL_FLAVOUR}" != ga ]]; then
      echo "ERROR: --stream cloud-image only publishes the generic (ga) kernel." >&2
      usage
    fi
    # Its initrd has no rooturl, so fetching the squashfs at boot is not
    # an option: it has to be stacked.
    if [[ "${REMOUNT_LOCAL}" != true && "${STACK_SQUASHFS}" != true ]]; then
      echo "==> --stream cloud-image implies --stack-squashfs"
      STACK_SQUASHFS=true
    fi
    [[ -n "${IMAGE_SOURCE}" ]] || IMAGE_SOURCE="${CLOUD_IMAGE_SOURCE}"
  else
    IMAGE_PROVIDER="maas"
    [[ -n "${IMAGE_SOURCE}" ]] || IMAGE_SOURCE="${IMAGE_SOURCE_BASE}/${STREAM}"
  fi
  IMAGE_SOURCE="${IMAGE_SOURCE%/}"
  readonly IMAGE_PROVIDER

  readonly RELEASE STREAM KERNEL_FLAVOUR DNS_SERVER IMAGE_SOURCE
  readonly ARCH_OVERRIDE SSH_KEY_ARGS USER_DATA_FILE
  readonly KERNEL_OVERRIDE INITRD_OVERRIDE SQUASHFS_OVERRIDE
  if uses_network_config_file && [[ ! -f "${NETWORK_CONFIG_SOURCE}" ]]; then
    echo "ERROR: --network-config takes auto, ip or an existing file: ${NETWORK_CONFIG_SOURCE}" >&2
    usage
  fi
  # A network-config file exists because ip= cannot express the network
  # (VLAN, bond, bridge, ...). In fetch mode the initramfs would still
  # need ip= to download the squashfs, which then cannot work, so the
  # squashfs is stacked and the initramfs needs no network at all.
  if uses_network_config_file && [[ "${STACK_SQUASHFS}" != true ]]; then
    echo "==> --network-config <file> implies --stack-squashfs"
    STACK_SQUASHFS=true
  fi
  readonly EXTRA_CMDLINE_ARGS NETWORK_CONFIG_SOURCE
  readonly STACK_SQUASHFS DNS_EXPLICIT
  NETWORK_MODE="$(resolve_network_mode)"
  readonly NETWORK_MODE

  if [[ "${REMOUNT_LOCAL}" == true ]]; then
    echo "==> Mode: remount-local (running kernel + local root, overlay ${OVERLAY_SIZE})"
  elif needs_stream_query; then
    echo "==> Image source: ${IMAGE_SOURCE} (release=${RELEASE} flavour=${KERNEL_FLAVOUR})"
  else
    echo "==> All artifacts supplied manually; image stream not queried."
  fi
  if [[ "${STACK_SQUASHFS}" == true ]]; then
    echo "==> Stacking mode: squashfs will be embedded in the initrd (no fetch at boot)"
  fi
}

# True when --network-config names a file (not auto, ip or unset).
uses_network_config_file() {
  [[ -n "${NETWORK_CONFIG_SOURCE:-}" && "${NETWORK_CONFIG_SOURCE}" != auto \
     && "${NETWORK_CONFIG_SOURCE}" != ip ]]
}

# How the rescue env gets its network:
#   ip      ip=/BOOTIF= on the cmdline only (initramfs + dyn-netconf)
#   simple  ip= plus a one-NIC static netplan  (fetch mode, auto)
#   render  netplan rendered from the running system, no ip=  (stacked)
#   file    the user's netplan, no ip=  (forces stacking)
# Fetch mode keeps ip=: its initramfs must reach the network itself.
resolve_network_mode() {
  case "${NETWORK_CONFIG_SOURCE:-}" in
    ip) echo ip ;;
    auto | "")
      if [[ "${STACK_SQUASHFS:-false}" == true ]]; then
        echo render
      elif [[ "${NETWORK_CONFIG_SOURCE:-}" == auto ]]; then
        echo simple
      else
        echo ip
      fi
      ;;
    *) echo file ;;
  esac
}

# NETWORK_MODE once parse_args has fixed it; derived otherwise (tests).
current_network_mode() {
  if [[ -n "${NETWORK_MODE:-}" ]]; then
    echo "${NETWORK_MODE}"
  else
    resolve_network_mode
  fi
}

# True when at least one artifact still has to come from the stream.
# --remount-local uses the running system's own kernel, so never.
needs_stream_query() {
  [[ "${REMOUNT_LOCAL:-false}" == true ]] && return 1
  [[ -z "${KERNEL_OVERRIDE}" || -z "${INITRD_OVERRIDE}" || -z "${SQUASHFS_OVERRIDE}" ]]
}

check_prerequisites() {
  local cmd
  for cmd in curl kexec ip awk base64; do
    if ! command -v "${cmd}" >/dev/null 2>&1; then
      echo "ERROR: required command '${cmd}' not found." >&2
      exit 1
    fi
  done
  if [[ "${STACK_SQUASHFS}" == true ]]; then
    if ! command -v cpio >/dev/null 2>&1; then
      echo "ERROR: cpio is required for --stack-squashfs." >&2
      exit 1
    fi
  fi
  if [[ "${REMOUNT_LOCAL}" != true && "${NETWORK_MODE}" == render ]] \
     && ! command -v jq >/dev/null 2>&1; then
    echo "ERROR: jq is required to render the running network (apt install jq)," >&2
    echo "       or pass --network-config ip|<file>." >&2
    exit 1
  fi

  if [[ "${EUID}" -ne 0 ]]; then
    echo "ERROR: must run as root." >&2
    exit 1
  fi
}

# Maps `uname -m` to the arch names used in the Ubuntu image streams.
detect_arch() {
  local machine

  if [[ -n "${ARCH_OVERRIDE}" ]]; then
    ARCH="${ARCH_OVERRIDE}"
    readonly ARCH
    echo "==> Architecture: ${ARCH} (overridden)"
    return
  fi

  machine="$(uname -m)"
  case "${machine}" in
    x86_64)  ARCH="amd64"   ;;
    aarch64) ARCH="arm64"   ;;
    ppc64le) ARCH="ppc64el" ;;
    s390x)   ARCH="s390x"   ;;
    *)
      echo "ERROR: unrecognized architecture '${machine}'; use --arch." >&2
      exit 1
      ;;
  esac
  readonly ARCH
  echo "==> Architecture: ${machine} -> ${ARCH}"
}

# Converts a CIDR prefix length (e.g. 24) to a dotted netmask.
prefix_to_netmask() {
  local prefix=$1 mask=() i
  for ((i = 0; i < 4; i++)); do
    if (( prefix >= 8 )); then
      mask+=(255)
      prefix=$((prefix - 8))
    else
      mask+=( $(( 256 - 2**(8 - prefix) )) )
      prefix=0
    fi
  done
  echo "${mask[0]}.${mask[1]}.${mask[2]}.${mask[3]}"
}

# Derives networking from the interface holding the default route: that's
# the NIC that can actually reach the outside world, and the one the
# rescue environment should come up on.
detect_network_config() {
  local route_line addr_line cidr

  route_line="$(ip -4 route show default | head -n1)"
  if [[ -z "${route_line}" ]]; then
    echo "ERROR: no IPv4 default route found; cannot auto-detect network config." >&2
    exit 1
  fi

  GATEWAY="$(awk '{for(i=1;i<=NF;i++) if ($i=="via") print $(i+1)}' <<<"${route_line}")"
  LOCAL_IFACE="$(awk '{for(i=1;i<=NF;i++) if ($i=="dev") print $(i+1)}' <<<"${route_line}")"
  if [[ -z "${GATEWAY}" || -z "${LOCAL_IFACE}" ]]; then
    echo "ERROR: could not parse gateway/interface from: ${route_line}" >&2
    exit 1
  fi

  addr_line="$(ip -4 -o addr show dev "${LOCAL_IFACE}" | head -n1)"
  if [[ -z "${addr_line}" ]]; then
    echo "ERROR: no IPv4 address found on ${LOCAL_IFACE}." >&2
    exit 1
  fi
  cidr="$(awk '{print $4}' <<<"${addr_line}")"
  STATIC_IP="${cidr%%/*}"
  IP_PREFIX="${cidr##*/}"
  NETMASK="$(prefix_to_netmask "${IP_PREFIX}")"

  LOCAL_MAC="$(cat "/sys/class/net/${LOCAL_IFACE}/address")"
  HOSTNAME="$(hostname)"

  readonly GATEWAY LOCAL_IFACE STATIC_IP IP_PREFIX NETMASK LOCAL_MAC HOSTNAME

  echo "==> Auto-detected network config:"
  echo "    iface=${LOCAL_IFACE} mac=${LOCAL_MAC} ip=${STATIC_IP}/${IP_PREFIX} netmask=${NETMASK}"
  echo "    gateway=${GATEWAY} hostname=${HOSTNAME} dns=${DNS_SERVER}"
}

### Image resolution (simplestreams) ###

# Wraps sstream-query so the signed/unsigned decision lives in one place.
run_sstream_query() {
  if [[ -f "${KEYRING_FILE}" ]]; then
    sstream-query --keyring="${KEYRING_FILE}" \
      "${IMAGE_SOURCE}/streams/v1/index.sjson" \
      "release=${RELEASE}" "arch=${ARCH}" \
      --output-format="%(version_name)s %(ftype)s %(path)s"
  else
    echo "WARN: ${KEYRING_FILE} not found; using unsigned stream index." >&2
    sstream-query \
      "${IMAGE_SOURCE}/streams/v1/index.json" \
      "release=${RELEASE}" "arch=${ARCH}" \
      --output-format="%(version_name)s %(ftype)s %(path)s"
  fi
}

# Overridable so tests can pretend simplestreams is (not) installed.
have_sstream_query() {
  command -v sstream-query >/dev/null 2>&1
}

# cloud-images.ubuntu.com layout:
#   <source>/<release>/<version>/<release>-server-cloudimg-<arch>.squashfs
#   <source>/<release>/<version>/unpacked/<release>-server-cloudimg-<arch>-{vmlinuz,initrd}-generic
# Walks dated version directories newest first (skipping the "current"
# alias, so the version actually used is visible and reproducible) and
# stops at the first with all three artifacts. Emits the same
# "version ftype path" lines as the other backends.
scrape_cloud_image_index() {
  local base="${IMAGE_SOURCE}/${RELEASE}"
  local stem="${RELEASE}-server-cloudimg-${ARCH}"
  local listing versions ver entries unpacked

  if ! listing="$(http_list_dir "${base}")"; then
    echo "ERROR: could not read ${base}/" >&2
    return 1
  fi
  versions="$(grep -E '^[0-9]{8}(\.[0-9]+)?/$' <<<"${listing}" | tr -d / | sort -Vr || true)"
  if [[ -z "${versions}" ]]; then
    echo "ERROR: no version directories under ${base}/" >&2
    return 1
  fi

  for ver in ${versions}; do
    entries="$(http_list_dir "${base}/${ver}")" || continue
    if ! grep -qx "${stem}.squashfs" <<<"${entries}"; then
      continue
    fi
    unpacked="$(http_list_dir "${base}/${ver}/unpacked")" || continue
    if ! grep -qx "${stem}-vmlinuz-generic" <<<"${unpacked}" \
        || ! grep -qx "${stem}-initrd-generic" <<<"${unpacked}"; then
      continue
    fi

    echo "${ver} squashfs ${RELEASE}/${ver}/${stem}.squashfs"
    echo "${ver} boot-kernel ${RELEASE}/${ver}/unpacked/${stem}-vmlinuz-generic"
    echo "${ver} boot-initrd ${RELEASE}/${ver}/unpacked/${stem}-initrd-generic"
    return 0
  done

  echo "ERROR: no complete cloud-image build for ${ARCH} under ${base}/" >&2
  return 1
}

# Checks one downloaded artifact against the SHA256SUMS published next to
# it, after verifying that SHA256SUMS is signed by the Ubuntu cloud-image
# key. Without the keyring or gpgv the hash is still checked, but only
# proves the download matches what the server says -- a warning says so.
verify_artifact_checksum() {
  local url="$1" file="$2"
  local dir="${url%/*}" name="${url##*/}"
  local sums="${WORKDIR}/SHA256SUMS.$$" sig="${WORKDIR}/SHA256SUMS.$$.gpg"
  local expected actual

  if ! curl -fsSL "${dir}/SHA256SUMS" -o "${sums}"; then
    echo "ERROR: could not fetch ${dir}/SHA256SUMS" >&2
    exit 1
  fi

  if [[ -f "${KEYRING_FILE}" ]] && command -v gpgv >/dev/null 2>&1; then
    if ! curl -fsSL "${dir}/SHA256SUMS.gpg" -o "${sig}" \
        || ! gpgv --keyring "${KEYRING_FILE}" "${sig}" "${sums}" 2>/dev/null; then
      rm -f "${sums}" "${sig}"
      echo "ERROR: signature check of ${dir}/SHA256SUMS failed." >&2
      exit 1
    fi
  else
    echo "WARN: no ${KEYRING_FILE} or gpgv; SHA256SUMS signature not verified." >&2
  fi

  # Lines are "<hash> *<name>" (binary mode) or "<hash>  <name>".
  expected="$(awk -v n="${name}" '$2 == n || $2 == "*" n { print $1; exit }' "${sums}")"
  rm -f "${sums}" "${sig}"
  if [[ -z "${expected}" ]]; then
    echo "ERROR: ${name} not listed in ${dir}/SHA256SUMS" >&2
    exit 1
  fi

  actual="$(sha256sum "${file}" | awk '{ print $1 }')"
  if [[ "${actual}" != "${expected}" ]]; then
    echo "ERROR: checksum mismatch for ${name}" >&2
    echo "       expected ${expected}" >&2
    echo "       got      ${actual}" >&2
    exit 1
  fi
  echo "==> ${name}: sha256 OK"
}

# Only artifacts that came from the index are checked; a manually
# supplied --kernel/--initrd/--squashfs has no published checksum.
verify_cloud_image_downloads() {
  if [[ -z "${KERNEL_OVERRIDE}" ]]; then
    verify_artifact_checksum "${KERNEL_URL}" "${KERNEL_FILE}"
  fi
  if [[ -z "${INITRD_OVERRIDE}" ]]; then
    verify_artifact_checksum "${INITRD_URL}" "${INITRD_FILE}"
  fi
  if [[ -z "${SQUASHFS_OVERRIDE}" ]]; then
    verify_artifact_checksum "${SQUASHFS_URL}" "${SQUASHFS_FILE}"
  fi
}

# Single entry point for listing artifacts. Both backends emit the same
# "version ftype path" lines, so resolve_image_urls picks versions the
# same way regardless of where the listing came from.
list_image_artifacts() {
  if [[ "${IMAGE_PROVIDER:-maas}" == cloud-image ]]; then
    scrape_cloud_image_index
  elif have_sstream_query; then
    run_sstream_query
  else
    echo "WARN: sstream-query not found; reading the directory index instead." >&2
    echo "      Stream signatures are not verified on this path (TLS only);" >&2
    echo "      sudo apt install simplestreams for signature-verified resolution." >&2
    scrape_image_index
  fi
}

# True if a stream path belongs to the requested kernel flavour. Matches
# the directory component exactly as <flavour>-<version>, so "hwe" picks
# hwe-24.04 but not hwe-24.04-edge sitting right next to it. A loose
# */hwe-*/* glob matches both, and whichever came last would silently win.
is_wanted_flavour() {
  # Cloud images carry one kernel per build; nothing to choose between.
  [[ "${IMAGE_PROVIDER:-maas}" == cloud-image ]] && return 0
  [[ "$1" =~ /${KERNEL_FLAVOUR}-[0-9]+(\.[0-9]+)*/ ]]
}

# Lists entry names from an Apache autoindex page, one per line, with a
# trailing / on directories. Skips the parent link (absolute href) and
# the column-sort links (?C=...).
http_list_dir() {
  local url="$1" page
  page="$(curl -fsSL "${url%/}/")" || return 1
  grep -oE 'href="[^"?/][^"]*"' <<<"${page}" \
    | sed -E 's/^href="//; s/"$//' || true
}

# Fallback for hosts without simplestreams: walks the published tree
#   <source>/<release>/<arch>/<version>/<flavour>-<ubuntu>/generic/boot-*
#   <source>/<release>/<arch>/<version>/squashfs
# newest version first, and stops at the first one that has all three
# artifacts for the requested flavour. Emits the same lines as
# run_sstream_query so the caller does not care which backend ran.
# Usually costs three requests; walks further only if the newest build
# is incomplete.
scrape_image_index() {
  local base="${IMAGE_SOURCE}/${RELEASE}/${ARCH}"
  local listing versions ver entries flavdir files prefix

  if ! listing="$(http_list_dir "${base}")"; then
    echo "ERROR: could not read ${base}/" >&2
    return 1
  fi
  versions="$(grep -E '^[0-9]+(\.[0-9]+)*/$' <<<"${listing}" | tr -d / | sort -Vr || true)"
  if [[ -z "${versions}" ]]; then
    echo "ERROR: no version directories under ${base}/" >&2
    return 1
  fi

  for ver in ${versions}; do
    entries="$(http_list_dir "${base}/${ver}")" || continue
    if ! grep -qx 'squashfs' <<<"${entries}"; then
      continue
    fi
    flavdir="$(grep -E "^${KERNEL_FLAVOUR}-[0-9]+(\.[0-9]+)*/\$" <<<"${entries}" \
      | tr -d / | sort -Vr | head -n1 || true)"
    if [[ -z "${flavdir}" ]]; then
      continue
    fi
    files="$(http_list_dir "${base}/${ver}/${flavdir}/generic")" || continue
    if ! grep -qx 'boot-kernel' <<<"${files}" || ! grep -qx 'boot-initrd' <<<"${files}"; then
      continue
    fi

    prefix="${RELEASE}/${ARCH}/${ver}"
    echo "${ver} squashfs ${prefix}/squashfs"
    echo "${ver} boot-kernel ${prefix}/${flavdir}/generic/boot-kernel"
    echo "${ver} boot-initrd ${prefix}/${flavdir}/generic/boot-initrd"
    return 0
  done

  echo "ERROR: no complete build for flavour ${KERNEL_FLAVOUR} under ${base}/" >&2
  return 1
}

# Applies any --kernel/--initrd/--squashfs overrides, then queries the
# stream only for whatever is still missing. When the stream is used, the
# newest version that has all three artifacts wins, so a resolved kernel,
# initrd and squashfs always come from one matched build rather than
# being mixed across versions.
resolve_image_urls() {
  local query_out ver ftype path
  local -A kernel_by_ver=() initrd_by_ver=() squash_by_ver=()

  KERNEL_URL="${KERNEL_OVERRIDE}"
  INITRD_URL="${INITRD_OVERRIDE}"
  SQUASHFS_URL="${SQUASHFS_OVERRIDE}"
  IMAGE_VERSION="manual"

  if needs_stream_query; then
    echo "==> Querying image stream..."
    if ! query_out="$(list_image_artifacts)"; then
      echo "ERROR: could not list images from ${IMAGE_SOURCE}." >&2
      exit 1
    fi
    if [[ -z "${query_out}" ]]; then
      echo "ERROR: no images found for release=${RELEASE} arch=${ARCH}." >&2
      exit 1
    fi

    while read -r ver ftype path; do
      [[ -n "${ver}" ]] || continue
      case "${ftype}" in
        boot-kernel)
          is_wanted_flavour "${path}" && kernel_by_ver["${ver}"]="${path}"
          ;;
        boot-initrd)
          is_wanted_flavour "${path}" && initrd_by_ver["${ver}"]="${path}"
          ;;
        squashfs)
          squash_by_ver["${ver}"]="${path}"
          ;;
      esac
    done <<< "${query_out}"

    IMAGE_VERSION=""
    for ver in $(printf '%s\n' "${!squash_by_ver[@]}" | sort -Vr); do
      if [[ -n "${kernel_by_ver[${ver}]:-}" && -n "${initrd_by_ver[${ver}]:-}" ]]; then
        IMAGE_VERSION="${ver}"
        break
      fi
    done

    if [[ -z "${IMAGE_VERSION}" ]]; then
      echo "ERROR: no stream version has boot-kernel + boot-initrd + squashfs" >&2
      echo "for release=${RELEASE} arch=${ARCH} flavour=${KERNEL_FLAVOUR}." >&2
      echo "Raw listing output follows so you can check field names:" >&2
      echo "${query_out}" | sed 's/^/    /' >&2
      exit 1
    fi

    # Overrides win; only fill what wasn't supplied.
    [[ -n "${KERNEL_URL}" ]]   || KERNEL_URL="${IMAGE_SOURCE}/${kernel_by_ver[${IMAGE_VERSION}]}"
    [[ -n "${INITRD_URL}" ]]   || INITRD_URL="${IMAGE_SOURCE}/${initrd_by_ver[${IMAGE_VERSION}]}"
    [[ -n "${SQUASHFS_URL}" ]] || SQUASHFS_URL="${IMAGE_SOURCE}/${squash_by_ver[${IMAGE_VERSION}]}"
  fi

  # When stacking, the squashfs is embedded and nothing is fetched at
  # boot, so no URL ends up on the cmdline and the http downgrade is
  # irrelevant. Otherwise: a stream-resolved squashfs is downgraded to
  # http because the stock initrd has no https client; a manually
  # supplied one is passed through exactly as given -- an explicit URL
  # should not be silently rewritten -- but https is worth warning about.
  if [[ "${STACK_SQUASHFS}" == true ]]; then
    SQUASHFS_BOOT_URL=""
  elif [[ -n "${SQUASHFS_OVERRIDE}" ]]; then
    SQUASHFS_BOOT_URL="${SQUASHFS_URL}"
    if [[ "${SQUASHFS_BOOT_URL}" == https://* ]]; then
      echo "WARN: --squashfs is https; the stock initrd cannot fetch https." >&2
      echo "      Passing it through unchanged anyway (or use --stack-squashfs)." >&2
    fi
  else
    SQUASHFS_BOOT_URL="$(to_http_url "${SQUASHFS_URL}")"
  fi

  readonly IMAGE_VERSION KERNEL_URL INITRD_URL SQUASHFS_URL SQUASHFS_BOOT_URL

  echo "==> Image version: ${IMAGE_VERSION}"
  echo "    kernel:   ${KERNEL_URL}${KERNEL_OVERRIDE:+  (manual)}"
  echo "    initrd:   ${INITRD_URL}${INITRD_OVERRIDE:+  (manual)}"
  if [[ "${STACK_SQUASHFS}" == true ]]; then
    echo "    squashfs: ${SQUASHFS_URL}${SQUASHFS_OVERRIDE:+  (manual)}  -> embedded in initrd"
  else
    echo "    squashfs: ${SQUASHFS_BOOT_URL}${SQUASHFS_OVERRIDE:+  (manual)}"
  fi
}

to_http_url() {
  local url="$1"
  if [[ "${url}" == https://* ]]; then
    echo "http://${url#https://}"
  else
    echo "${url}"
  fi
}

# Fetches one artifact into the work dir. A local path is copied rather
# than fetched, so --kernel/--initrd can point at files already on disk
# (handy when hand-building an initrd).
fetch_artifact() {
  local src="$1" dest="$2" label="$3"

  if [[ "${src}" == http://* || "${src}" == https://* ]]; then
    echo "==> Downloading ${label}"
    curl -fsSL "${src}" -o "${dest}"
  elif [[ -f "${src}" ]]; then
    echo "==> Copying ${label} from ${src}"
    cp -- "${src}" "${dest}"
  else
    echo "ERROR: ${label} is neither a URL nor an existing file: ${src}" >&2
    exit 1
  fi
}

download_kernel_initrd() {
  KERNEL_FILE="${WORKDIR}/boot-kernel"
  INITRD_FILE="${WORKDIR}/boot-initrd"
  fetch_artifact "${KERNEL_URL}" "${KERNEL_FILE}" "kernel"
  fetch_artifact "${INITRD_URL}" "${INITRD_FILE}" "initrd"
  validate_kernel_image "${KERNEL_FILE}"
  warn_if_kernel_initrd_mismatch
}

# On x86, an uncompressed `vmlinux` is an ELF linked to load at a fixed
# physical address, which kexec rejects ("Invalid memory segment
# 0x1000000 - ..."); the bootable image is the relocatable bzImage from
# the same build. Caught here rather than as a cryptic kexec error.
validate_kernel_image() {
  local file="$1" magic
  [[ "$(uname -m)" == x86_64 ]] || return 0
  magic="$(head -c 4 "${file}" | od -An -tx1 | tr -d ' \n')"
  if [[ "${magic}" == 7f454c46 ]]; then
    echo "ERROR: the kernel is an uncompressed ELF vmlinux, which kexec cannot" >&2
    echo "       load on x86. Use the bzImage from the same build, e.g." >&2
    echo "       <build-dir>/arch/x86/boot/bzImage" >&2
    exit 1
  fi
}

# A kernel and its initrd must come from the same build: the initrd's
# modules sit under /lib/modules/<that kernel's uname -r>, so with any
# other kernel every modprobe in early boot finds nothing. A custom
# --kernel with a stream-resolved initrd only boots if everything it
# needs is built in (=y): NIC driver, CONFIG_PACKET, BLK_DEV_LOOP,
# SQUASHFS and its compressor, OVERLAY_FS -- and anything needed after
# boot, since the squashfs's modules will not match either.
warn_if_kernel_initrd_mismatch() {
  if [[ -n "${KERNEL_OVERRIDE}" && -z "${INITRD_OVERRIDE}" ]]; then
    echo "WARN: custom --kernel with the stream's initrd. Its modules are built" >&2
    echo "      for the stream kernel and will not load into yours, so your kernel" >&2
    echo "      must have built in (=y): your NIC driver, CONFIG_PACKET," >&2
    echo "      BLK_DEV_LOOP, SQUASHFS (+ its compressor), OVERLAY_FS. Otherwise" >&2
    echo "      pass a matching --initrd built for this kernel." >&2
  fi
}

### Stacking (--stack-squashfs) ###

download_squashfs() {
  # Downloaded straight into the cpio staging directory: the file then
  # has nlink=1 and needs no copy or hardlink before archiving.
  SQUASHFS_STAGE="${WORKDIR}/squashfs-cpio"
  rm -rf "${SQUASHFS_STAGE}"
  mkdir -p "${SQUASHFS_STAGE}"
  SQUASHFS_FILE="${SQUASHFS_STAGE}/${SQUASHFS_INITRAMFS_NAME}"
  fetch_artifact "${SQUASHFS_URL}" "${SQUASHFS_FILE}" "squashfs"
  readonly SQUASHFS_STAGE SQUASHFS_FILE
  warn_if_squashfs_large
}

# The squashfs lives in initramfs tmpfs for the whole session, so it is
# permanently resident RAM. Worth a heads-up before kexec rather than an
# OOM after it.
warn_if_squashfs_large() {
  local squashfs_kb mem_kb pct
  squashfs_kb=$(( $(stat -c %s "${SQUASHFS_FILE}") / 1024 ))
  mem_kb="$(awk '/^MemTotal:/{print $2}' /proc/meminfo)"
  [[ -n "${mem_kb}" && "${mem_kb}" -gt 0 ]] || return 0

  pct=$(( squashfs_kb * 100 / mem_kb ))
  echo "==> squashfs is $(( squashfs_kb / 1024 )) MiB (${pct}% of $(( mem_kb / 1024 )) MiB RAM)"
  if [[ "${pct}" -ge 50 ]]; then
    echo "WARN: the squashfs stays resident in initramfs tmpfs for the whole" >&2
    echo "      session, on top of overlayroot=tmpfs for writes. At ${pct}% of" >&2
    echo "      total RAM this host may not have room to boot." >&2
  fi
}

# The local-top script we inject. It does one thing: attach the squashfs
# that our cpio segment placed in the initramfs to a loop device. The
# cmdline already names that device as root=, so the stock mountroot
# mounts it normally -- no /conf/param.conf rewriting, no -o move, and
# /root ends up a real mountpoint so overlayroot=tmpfs can move it.
write_rescue_script() {
  local dest="$1"

  cat > "${dest}" << RESCUE_EOF
#!/bin/sh
# Injected by rescue-kexec.sh. Attaches the in-initramfs squashfs to a
# loop device so the stock initramfs-tools root mount can use it.
PREREQ=""
prereqs() { echo "\${PREREQ}"; }
case "\${1:-}" in
  prereqs) prereqs; exit 0 ;;
esac

. /scripts/functions

squashfs_path=""
for param in \$(cat /proc/cmdline); do
  case "\${param}" in
    rescue.squashfs=*) squashfs_path="\${param#rescue.squashfs=}" ;;
  esac
done

[ -n "\${squashfs_path}" ] || exit 0

if [ ! -f "\${squashfs_path}" ]; then
  panic "rescue-squashfs: \${squashfs_path} not present in initramfs"
fi

# Usually built in; harmless if modprobe is absent or they are missing.
modprobe -q loop 2>/dev/null || true
modprobe -q squashfs 2>/dev/null || true

[ -e "${RESCUE_LOOP_DEV}" ] || mknod "${RESCUE_LOOP_DEV}" b 7 0 || true

if ! losetup "${RESCUE_LOOP_DEV}" "\${squashfs_path}"; then
  panic "rescue-squashfs: losetup ${RESCUE_LOOP_DEV} \${squashfs_path} failed"
fi

log_success_msg "rescue-squashfs: \${squashfs_path} attached to ${RESCUE_LOOP_DEV}"
RESCUE_EOF
  chmod 755 "${dest}"
}

# Builds the combined initrd as the base image plus one appended cpio
# segment carrying the squashfs, our local-top script, and an ORDER that
# runs only that script.
#
# Appending rather than unpack/repack preserves any microcode early-cpio
# segment at the front of the base image, and avoids decompressing 80 MB
# of zstd.
#
# The padding matters: the kernel's initramfs unpacker only recognises a
# cpio segment when its start is 4-byte aligned, and skips NUL bytes
# between segments. The vendor initrd does not end on a boundary, so
# without this the segment lands misaligned and is silently never
# unpacked. 512 is used rather than 4 to match what cpio does for its
# own archives.
#
# The segment is left uncompressed: the squashfs dominates it and is
# already compressed, so gzipping would burn boot CPU for nothing.
build_stacked_initrd() {
  local stacked="${WORKDIR}/boot-initrd.stacked"
  local order="${SQUASHFS_STAGE}/scripts/local-top/ORDER"

  echo "==> Injecting ${RESCUE_SCRIPT_NAME} local-top script"
  mkdir -p "${SQUASHFS_STAGE}/scripts/local-top"
  write_rescue_script "${SQUASHFS_STAGE}/scripts/local-top/${RESCUE_SCRIPT_NAME}"

  # Replaces the base ORDER wholesale, so only our script runs: nothing
  # else in local-top gets a chance to touch the local disks.
  cat > "${order}" << ORDER_EOF
/scripts/local-top/${RESCUE_SCRIPT_NAME} "\$@"
[ -e /conf/param.conf ] && . /conf/param.conf
ORDER_EOF

  cp "${INITRD_FILE}" "${stacked}"
  pad_to_boundary "${stacked}" 512

  echo "==> Appending squashfs and scripts as one cpio segment"
  ( cd "${SQUASHFS_STAGE}" && find . -mindepth 1 -print | sort \
      | cpio -o -H newc --quiet ) >> "${stacked}"

  INITRD_FILE="${stacked}"
  verify_stacked_initrd
  rm -rf "${SQUASHFS_STAGE}"
  echo "==> Combined initrd: $(( $(stat -c %s "${INITRD_FILE}") / 1024 / 1024 )) MiB"
}

# Appends NUL bytes until the file length is a multiple of $2. The kernel
# skips NULs between initramfs segments, so this is safe filler.
pad_to_boundary() {
  local file="$1" boundary="$2" size pad
  size="$(stat -c %s "${file}")"
  pad=$(( (boundary - (size % boundary)) % boundary ))
  if [[ "${pad}" -gt 0 ]]; then
    head -c "${pad}" /dev/zero >> "${file}"
  fi
}

# Confirms the squashfs cpio header actually landed where the kernel will
# find it. This class of bug is invisible until boot otherwise: the data
# sits in the file, looks complete, and is silently skipped.
verify_stacked_initrd() {
  local result
  result="$(SQ_NAME="${SQUASHFS_INITRAMFS_NAME}" \
    RESCUE_NAME="${RESCUE_SCRIPT_NAME}" \
    EXPECTED_SIZE="$(stat -c %s "${SQUASHFS_FILE}")" \
    python3 - "${INITRD_FILE}" << 'PYEOF'
import os, sys

sq = os.environ["SQ_NAME"].encode()
rescue = os.environ["RESCUE_NAME"].encode()
expected = int(os.environ["EXPECTED_SIZE"])
data = open(sys.argv[1], "rb").read()


def find_header(path):
    """Locate the newc header whose filename field is exactly `path`.

    The filename starts 110 bytes into the header, so anchoring there
    avoids matching a longer path that merely ends with the same text
    (squashfs vs scripts/local-top/rescue-squashfs). cpio strips a
    leading "./", but accept either form.
    """
    for want in (path + b"\x00", b"./" + path + b"\x00"):
        start = 0
        while True:
            i = data.find(want, start)
            if i < 0:
                break
            off = i - 110
            if off >= 0 and data[off:off + 6] == b"070701":
                return off
            start = i + 1
    return None


problems = []

sq_off = find_header(sq)
if sq_off is None:
    problems.append("no cpio header found naming the squashfs")
else:
    size = int(data[sq_off + 54:sq_off + 62], 16)
    if sq_off % 4:
        problems.append(f"squashfs header at {sq_off} is {sq_off % 4} mod 4; "
                        "the kernel only accepts 4-byte-aligned segments")
    if size != expected:
        problems.append(f"squashfs header filesize {size} != {expected}")

if find_header(b"scripts/local-top/" + rescue) is None:
    problems.append("injected local-top script not found")
if find_header(b"scripts/local-top/ORDER") is None:
    problems.append("injected ORDER not found")

print("FAIL " + "; ".join(problems) if problems
      else f"OK squashfs at {sq_off}, script and ORDER present")
PYEOF
  )"

  echo "==> Verifying stacked initrd: ${result}"
  if [[ "${result}" != OK* ]]; then
    echo "ERROR: stacked initrd would not unpack correctly; refusing to kexec." >&2
    exit 1
  fi
}

### cloud-init injection ###

# How cloud-init reads cc: blocks (cloudinit.util.read_cc_from_cmdline):
# it extracts every "cc:...end_cc" on the cmdline and joins them with
# newlines into ONE YAML text. They are concatenated, not merged. So each
# block must be block-style top-level "key: value" lines -- never a
# {...} flow mapping, because two of those back to back are invalid YAML
# and cloud-init then discards ALL cmdline config (hostname, keys,
# password) at once. Multi-key blocks separate their keys with CC_NL, a
# literal backslash-n that cloud-init turns into a newline.
CC_NL='\n'

escape_for_yaml_dq() {
  sed -e 's/\\/\\\\/g' -e 's/"/\\"/g' <<<"$1"
}

# Reads an authorized_keys file, stripping any leading options
# (command=, from=, no-port-forwarding, ...) from each line. A forced
# command or restricted key would likely block the interactive shell we
# need in the rescue environment, so only "keytype base64 [comment]"
# is kept.
# Prints "keytype base64 [comment]" for a public-key line, or nothing.
# Leading authorized_keys options (command=, from=, no-port-forwarding,
# ...) are stripped: a forced command or restricted key would likely
# block the interactive shell we need in the rescue environment.
extract_public_key() {
  local keytypes='ssh-rsa|ssh-dss|ssh-ed25519'
  keytypes+='|ecdsa-sha2-nistp256|ecdsa-sha2-nistp384|ecdsa-sha2-nistp521'
  keytypes+='|sk-ssh-ed25519@openssh[.]com|sk-ecdsa-sha2-nistp256@openssh[.]com'
  keytypes+='|ssh-rsa-cert-v01@openssh[.]com|ssh-dss-cert-v01@openssh[.]com'
  keytypes+='|ssh-ed25519-cert-v01@openssh[.]com'
  keytypes+='|ecdsa-sha2-nistp256-cert-v01@openssh[.]com'
  keytypes+='|ecdsa-sha2-nistp384-cert-v01@openssh[.]com'
  keytypes+='|ecdsa-sha2-nistp521-cert-v01@openssh[.]com'

  awk -v types="${keytypes}" '
    {
      n = split($0, words, " ")
      for (i = 1; i <= n; i++) {
        if (words[i] ~ "^(" types ")$" && i < n) {
          out = words[i]
          for (j = i + 1; j <= n; j++) out = out " " words[j]
          print out
          exit
        }
      }
    }' <<<"$1"
}

# Collects keys from each --ssh-key value: an existing file is read as
# authorized_keys (one key per line), anything else must itself be a
# public key. Populates AUTH_KEY_ENTRIES.
parse_authorized_keys() {
  local src line cleaned before

  AUTH_KEY_ENTRIES=()
  for src in "$@"; do
    if [[ -f "${src}" ]]; then
      # A private key here would mean someone is about to paste secret
      # material onto a kernel cmdline readable by every local user.
      if grep -q 'PRIVATE KEY' "${src}"; then
        echo "ERROR: ${src} is a PRIVATE key. Pass the .pub file instead." >&2
        exit 1
      fi
      before="${#AUTH_KEY_ENTRIES[@]}"
      while IFS= read -r line; do
        [[ -z "${line}" || "${line}" =~ ^[[:space:]]*# ]] && continue
        cleaned="$(extract_public_key "${line}")"
        if [[ -n "${cleaned}" ]]; then
          AUTH_KEY_ENTRIES+=("${cleaned}")
        else
          echo "WARN: could not parse a key type from line, skipping: ${line}" >&2
        fi
      done < "${src}"
      echo "==> $(( ${#AUTH_KEY_ENTRIES[@]} - before )) key(s) from ${src} (leading options, if any, stripped)"
    else
      cleaned="$(extract_public_key "${src}")"
      if [[ -z "${cleaned}" ]]; then
        echo "ERROR: --ssh-key value is neither an existing file nor a public key:" >&2
        echo "       ${src:0:60}" >&2
        exit 1
      fi
      AUTH_KEY_ENTRIES+=("${cleaned}")
      echo "==> 1 key from the command line (${cleaned%% *})"
    fi
  done

  if [[ "${#AUTH_KEY_ENTRIES[@]}" -eq 0 ]]; then
    echo "ERROR: no valid keys found in the --ssh-key value(s)." >&2
    exit 1
  fi
  readonly AUTH_KEY_ENTRIES
}

join_ssh_keys() {
  local entries=() entry escaped
  for entry in "${AUTH_KEY_ENTRIES[@]}"; do
    escaped="$(escape_for_yaml_dq "${entry}")"
    entries+=("\"${escaped}\"")
  done
  local IFS=,
  echo "${entries[*]}"
}

# URL-encodes --user-data for embedding in a cc:...end_cc block. URL
# encoding is one of the two encodings cloud-init's kernel-cmdline
# reader supports, and avoids escaping ambiguity. Caveat: quote() never
# escapes letters/digits/"_.-~", so a literal "end_cc" anywhere in the
# file terminates parsing early. Avoid that string.
build_user_data_cc_block() {
  local encoded byte_count

  if [[ ! -f "${USER_DATA_FILE}" ]]; then
    echo "ERROR: user-data file not found: ${USER_DATA_FILE}" >&2
    exit 1
  fi
  if ! command -v python3 >/dev/null 2>&1; then
    echo "ERROR: python3 is required to URL-encode --user-data content." >&2
    exit 1
  fi

  encoded="$(python3 -c \
    'import sys,urllib.parse; sys.stdout.write(urllib.parse.quote(sys.stdin.read()))' \
    < "${USER_DATA_FILE}")"
  USER_DATA_CC_BLOCK="cc:${encoded}end_cc"
  readonly USER_DATA_CC_BLOCK

  byte_count="$(wc -c < "${USER_DATA_FILE}")"
  echo "==> Embedded --user-data from ${USER_DATA_FILE} (URL-encoded, ${byte_count} bytes)"
}

# Used when --user-data is NOT given: we own the whole cloud-config, so
# disable datasource probing, set the hostname (cloud-init would default
# to "ubuntu" regardless of ip='s hostname field), inject the keys, and
# make sure sshd is up.
build_full_cc_block() {
  local joined hostname_escaped
  joined="$(join_ssh_keys)"
  hostname_escaped="$(escape_for_yaml_dq "${HOSTNAME}")"

  SSH_CC_BLOCK="cc:datasource_list: ['None']${CC_NL}"
  SSH_CC_BLOCK+="hostname: \"${hostname_escaped}\"${CC_NL}"
  SSH_CC_BLOCK+="ssh_authorized_keys: [${joined}]${CC_NL}"
  SSH_CC_BLOCK+="runcmd: [['systemctl','enable','--now','ssh']]end_cc"
  readonly SSH_CC_BLOCK
}

# --user-data is a take-full-control choice, so the default password is
# only layered on top of it when --password was given explicitly.
wants_password_block() {
  [[ -z "${USER_DATA_FILE}" || "${PASSWORD_EXPLICIT}" == true ]]
}

# Sets the default user's password via cloud-init chpasswd. The block
# lands on the kernel cmdline, which anyone can read from /proc/cmdline,
# so only a SHA-512 crypt hash goes in, never the plaintext. The
# plaintext reaches openssl on stdin, keeping it out of `ps` as well.
# Separate cc: block; cloud-init merges it with the others.
build_password_cc_block() {
  local hash user_escaped

  if ! command -v openssl >/dev/null 2>&1; then
    echo "ERROR: openssl is required to hash --password." >&2
    exit 1
  fi
  hash="$(printf '%s' "${PASSWORD}" | openssl passwd -6 -stdin)"
  if [[ "${hash}" != '$6$'* ]]; then
    echo "ERROR: could not hash --password." >&2
    exit 1
  fi
  user_escaped="$(escape_for_yaml_dq "${DEFAULT_USER}")"

  PASSWORD_CC_BLOCK="cc:chpasswd: {'expire': False, 'users': "
  PASSWORD_CC_BLOCK+="[{'name': \"${user_escaped}\", 'password': \"${hash}\", 'type': 'hash'}]}end_cc"
  readonly PASSWORD_CC_BLOCK

  if [[ "${PASSWORD_EXPLICIT}" == true ]]; then
    echo "==> Console password set for user ${DEFAULT_USER} (hashed)"
  else
    echo "==> Console password for user ${DEFAULT_USER} is the default 'ubuntu' (hashed)"
  fi
}

# Used alongside --user-data: only the keys go in, as a separate cc:
# block. Supplying user-data is a take-full-control choice, so everything
# else is left to it. cloud-init concatenates this after the user-data,
# so if the user-data also sets ssh_authorized_keys, the later key wins:
# these REPLACE those rather than add to them.
build_ssh_only_cc_block() {
  local joined
  joined="$(join_ssh_keys)"
  SSH_CC_BLOCK="cc:ssh_authorized_keys: [${joined}]end_cc"
  readonly SSH_CC_BLOCK
}

# Opt-in (--network-config auto|<file>). cloud-init normally derives a
# correct MAC-matched netplan from the initramfs config on its own; this
# overrides it. Uses cloud-init's dedicated network-config= kernel
# parameter, NOT a network: key inside cc: -- user-data is not permitted
# to set network config. cloudinit/stages.py consults the cmdline source
# before the initramfs one, so this wins over what ip= produces.
build_network_config_param() {
  local yaml mode
  mode="$(current_network_mode)"

  if [[ "${mode}" == render ]]; then
    render_runtime_network
    yaml="${RENDERED_NETPLAN}"
  elif [[ "${mode}" == simple ]]; then
    yaml="network:
  version: 2
  ethernets:
    rescue0:
      match:
        macaddress: \"${LOCAL_MAC}\"
      set-name: rescue0
      addresses: [\"${STATIC_IP}/${IP_PREFIX}\"]
      routes:
        - to: default
          via: \"${GATEWAY}\"
      nameservers:
        addresses: [\"${DNS_SERVER}\"]
"
    echo "==> Network config: auto (MAC ${LOCAL_MAC}, ${STATIC_IP}/${IP_PREFIX})"
  else
    validate_network_config_file "${NETWORK_CONFIG_SOURCE}"
    yaml="$(< "${NETWORK_CONFIG_SOURCE}")"
    echo "==> Network config: ${NETWORK_CONFIG_SOURCE}"
  fi

  NETWORK_CONFIG_PARAM="network-config=$(encode_network_config "${yaml}")"
  readonly NETWORK_CONFIG_PARAM
}

# cloud-init base64-decodes network-config= and gunzips it if gzipped
# (cloudinit/net/cmdline.py _b64dgz), so gzip first whenever that comes
# out shorter: the cmdline is size-limited. -n keeps names and
# timestamps out of the output, so it is reproducible.
encode_network_config() {
  local plain gz
  plain="$(printf '%s' "$1" | base64 -w0)"
  gz="$(printf '%s' "$1" | gzip -9n | base64 -w0)"
  if [[ "${#gz}" -lt "${#plain}" ]]; then
    printf '%s' "${gz}"
  else
    printf '%s' "${plain}"
  fi
}

# Checks the file is what cloud-init will accept, mirroring its own
# handling: a YAML mapping, top-level network: key optional
# (stages.py _get_network_key_contents), version 1 or 2. Catching it
# here beats a rescue env that silently comes up without networking.
validate_network_config_file() {
  local file="$1" result

  if ! python3 -c 'import yaml' 2>/dev/null; then
    echo "WARN: python3-yaml not available; ${file} not validated." >&2
    return 0
  fi
  result="$(python3 - "${file}" << 'VALIDATE_EOF'
import sys, yaml
try:
    cfg = yaml.safe_load(open(sys.argv[1]))
except yaml.YAMLError as e:
    print("not valid YAML: %s" % str(e).splitlines()[0])
    sys.exit()
if isinstance(cfg, dict) and "network" in cfg:
    cfg = cfg["network"]
if not isinstance(cfg, dict):
    print("not a mapping")
    sys.exit()
if cfg.get("version") not in (1, 2):
    print("missing or unsupported version (need 1 or 2)")
    sys.exit()
print("OK")
VALIDATE_EOF
  )"
  if [[ "${result}" != OK ]]; then
    echo "ERROR: --network-config ${file}: ${result}" >&2
    exit 1
  fi
}

### Network rendering from the running system ###

# Turns captured `ip -j` state into a netplan v2 document (as JSON,
# which is valid YAML for cloud-init). Inputs are files so this stays a
# pure transform: $1 = ip -j -d addr show, $2/$3 = ip -j -4/-6 route
# show table main, $4 = DNS as {"addresses":[...],"search":[...]}.
# Prints {"netplan":..., "warnings":[...], "errors":[...]}.
#
# Scope: every interface with a global address whose type netplan can
# express (ethernet, bond, bridge, vlan), plus everything below it.
# Addresses are rendered static: this is a snapshot of what works now,
# not of what the config files say. Physical NICs are matched by their
# permanent MAC (bond members report the bond's MAC as their current
# one), so kernel renaming in the rescue env does not matter. Anything
# not on a default-route path, or currently down, is optional so
# systemd-networkd-wait-online does not stall the boot on it.
render_netplan_json() {
  jq -n \
    --slurpfile links "$1" --slurpfile r4 "$2" --slurpfile r6 "$3" \
    --slurpfile dns "$4" '
  def kind:
    if . == null then "missing"
    elif .link_type == "loopback" then "loopback"
    elif (.linkinfo.info_kind // null) == null then
      (if .link_type == "ether" then "ethernet" else "other" end)
    else .linkinfo.info_kind end;
  def supported: kind | IN("ethernet", "bond", "bridge", "vlan");
  def global_addrs:
    [ .addr_info[]?
      | select(.scope == "global")
      | select(((.temporary // false) or (.deprecated // false)
                or (.dadfailed // false)) | not)
      | "\(.local)/\(.prefixlen)" ];
  def perm_mac: .linkinfo.info_slave_data.perm_hwaddr // .permaddr // .address;
  def opt($k; $v): if $v == null or $v == 0 or $v == "" then {} else {($k): $v} end;
  def proto: .protocol // "boot";

  $links[0] as $links
  | ($links | INDEX(.ifname)) as $by
  | def ports($n): [ $links[] | select(.master == $n) ];
    def lower:
      if kind == "vlan" then [ .link | select(. != null) ]
      elif kind | IN("bond", "bridge") then
        [ ports(.ifname)[] | select(supported) | .ifname ]
      else [] end;
    def grow: . as $s | ([ $s[], ($s[] | $by[.] | lower[]) ] | unique);
    def closure: until(. == grow; grow);

  [ $links[] | select(kind != "loopback") | select((global_addrs | length) > 0) ]
    as $addressed
  | ([ $addressed[] | select(supported) | .ifname ] | closure) as $set

  | def route_ok:
      ((.type // "unicast") == "unicast") and (proto != "kernel")
      and (.gateway != null or (proto | IN("static", "boot")));
    def to_route($fam):
      { to: (if .dst == "default" then "default"
             elif (.dst | contains("/")) then .dst
             else .dst + (if $fam == 6 then "/128" else "/32" end) end) }
      + opt("via"; .gateway) + opt("metric"; .metric) + opt("from"; .prefsrc)
      + (if (.flags // []) | index("onlink") then {"on-link": true} else {} end)
      + (if .gateway == null then {scope: "link"} else {} end);

  ([ ($r4[0][] | select(route_ok) | {dev, dst, r: to_route(4)}),
     ($r6[0][] | select(route_ok) | {dev, dst, r: to_route(6)}) ]) as $routes
  | [ $routes[] | select(.dev as $d | $d != null and ($set | index($d)) != null) ] as $kept
  | ([ $kept[] | select(.dst == "default") | .dev ] | unique) as $default_devs
  | ($default_devs | closure) as $default_set
  | (if ($default_devs | length) > 0 then $default_devs
     else [ $set[] | select(($by[.] | global_addrs | length) > 0) ][0:1] end)
    as $dns_devs

  | def node($n):
      $by[$n] as $l | ($l | kind) as $k | $l.linkinfo.info_data as $d
      | (if $k == "ethernet" then
           { match: { macaddress: ($l | perm_mac) }, "set-name": $n }
           + (if $l.master == null and $l.address != ($l | perm_mac)
              then { macaddress: $l.address } else {} end)
         elif $k == "bond" then
           { parameters: ({ mode: $d.mode }
               + opt("mii-monitor-interval"; $d.miimon)
               + opt("up-delay"; $d.updelay) + opt("down-delay"; $d.downdelay)
               + (if $d.mode == "802.3ad" then
                    opt("lacp-rate"; $d.ad_lacp_rate) + opt("ad-select"; $d.ad_select)
                  else {} end)
               + (if $d.mode | IN("802.3ad", "balance-xor", "balance-tlb") then
                    opt("transmit-hash-policy"; $d.xmit_hash_policy) else {} end)
               + (if $d.mode == "active-backup" then opt("primary"; $d.primary)
                  else {} end)),
             macaddress: $l.address }
         elif $k == "bridge" then
           { parameters: { stp: (($d.stp_state // 0) != 0) }, macaddress: $l.address }
         else { id: $d.id, link: $l.link } end)
      + ([ ports($n)[] | select(supported) | .ifname ] | sort
         | if length > 0 and ($k | IN("bond", "bridge"))
           then { interfaces: . } else {} end)
      + (if ($l.mtu // 1500) != 1500 then { mtu: $l.mtu } else {} end)
      + (($l | global_addrs) as $a
         | if ($a | length) > 0
           then { addresses: $a, dhcp4: false, dhcp6: false } else {} end)
      + (if $l.master == null then { "accept-ra": false } else {} end)
      + ([ $kept[] | select(.dev == $n) | .r ]
         | if length > 0 then { routes: . } else {} end)
      + (if ($dns_devs | index($n)) != null then
           { nameservers: ({ addresses: $dns[0].addresses }
               + (if ($dns[0].search | length) > 0
                  then { search: $dns[0].search } else {} end)) }
         else {} end)
      + (if ($default_set | index($n)) == null
            or (($l.operstate // "UNKNOWN") | IN("UP", "UNKNOWN") | not)
         then { optional: true } else {} end);

    def section($k): [ $set[] | select(($by[.] | kind) == $k) ]
      | if length > 0 then { (if $k == "ethernet" then "ethernets" else "\($k)s" end):
                             (map({ (.): node(.) }) | add) }
        else {} end;

  { netplan: { network: ({ version: 2 } + section("ethernet") + section("bond")
                         + section("bridge") + section("vlan")) },
    rendered: $set,
    warnings: (
      [ $addressed[] | select(supported | not)
        | "left out \(.ifname) (\(kind), \(global_addrs | join(" "))): type not supported; it will not be reachable through it" ]
      + [ $set[] | $by[.] | select(kind | IN("bond", "bridge")) | .ifname as $n
          | [ ports($n)[] | select(supported | not) | "\(.ifname) (\(kind))" ]
          | select(length > 0)
          | "\($n): left out port(s) \(join(", ")); they will not exist in the rescue env" ]
      + [ $routes[] | select(.dev == null) | "left out multipath route to \(.dst)" ]
      + [ $routes[] | select(.dev as $d | $d != null and ($set | index($d)) == null)
          | "left out route to \(.dst) via \(.dev): interface not rendered" ]),
    errors: (
      [ $set[] | $by[.] | select(kind == "vlan") | . as $v | $by[$v.link]
        | select(supported | not)
        | "vlan \($v.ifname): parent \($v.link // "?") is \(kind), not supported" ]
      + (if ($set | length) == 0 then
           [ "no interface with a global address of a supported type" ] else [] end)
      + (if ([ $routes[] | select(.dst == "default") ] | length) > 0
            and ($default_devs | length) == 0 then
           [ "the default route uses an interface that cannot be rendered" ]
         else [] end)) }'
}

# DNS for the rendered config, from the resolv.conf actually in use.
# systemd-resolved's own file lists the real upstream servers; the
# stub 127.0.0.53 in /etc/resolv.conf would be useless in the rescue
# env. Loopback servers are dropped; if nothing is left, $2 is used.
runtime_dns_json() {
  local file="$1" fallback="$2"
  if [[ ! -r "${file}" ]]; then
    file=/dev/null
  fi
  jq -Rn --arg fallback "${fallback}" '
    [ inputs | sub("#.*$"; "") | [ splits("[ \t]+") | select(length > 0) ] ]
      as $lines
    | ([ $lines[] | select(.[0] == "nameserver") | .[1]
         | select(test("^127[.]|^::1$") | not) ]) as $ns
    | { addresses: (if ($ns | length) > 0 then $ns else [ $fallback ] end),
        search: [ $lines[] | select(.[0] == "search") | .[1:][] ] }' < "${file}"
}

# Rules outside local/main/default (VPNs, multi-uplink setups) route
# via other tables. Only the main table is rendered.
warn_if_policy_routing() {
  local fam extra
  for fam in -4 -6; do
    extra="$(ip -j "${fam}" rule show 2>/dev/null \
      | jq -r '[ .[]? | select(.table | IN("local", "main", "default") | not)
                 | "\(.priority): table \(.table)" ] | join("; ")' 2>/dev/null || true)"
    if [[ -n "${extra}" ]]; then
      echo "WARN: policy routing (ip ${fam} rule) is not rendered: ${extra}" >&2
    fi
  done
}

# Runs netplan's generator over the file in a scratch root, when it
# exists here (Ubuntu), to catch schema errors before the jump. The bare
# libexec binary, not `netplan generate`: the latter also runs
# `systemctl daemon-reload` on the live host even with --root-dir.
readonly NETPLAN_GENERATOR="/usr/libexec/netplan/generate"
validate_with_netplan_generate() {
  local file="$1" root out
  if [[ ! -x "${NETPLAN_GENERATOR}" ]]; then
    echo "    (netplan not installed here; schema not pre-validated)"
    return 0
  fi
  root="$(mktemp -d)"
  install -D -m 600 "${file}" "${root}/etc/netplan/90-rescue.yaml"
  if ! out="$("${NETPLAN_GENERATOR}" --root-dir "${root}" 2>&1)"; then
    rm -rf "${root}"
    echo "ERROR: rendered network config rejected by netplan:" >&2
    echo "${out}" >&2
    exit 1
  fi
  rm -rf "${root}"
  echo "    netplan schema check: OK"
}

# Captures the running network, renders it and reports. Sets
# RENDERED_NETPLAN to the compact document for the cmdline. The pretty
# copy stays in ${WORKDIR}/runtime-net/ for inspection.
render_runtime_network() {
  local dir="${WORKDIR}/runtime-net" dns_file=/etc/resolv.conf
  local dns_fallback="${DNS_SERVER}" warning error f
  local -a errors=()

  mkdir -p "${dir}"
  ip -j -d addr show > "${dir}/addr.json"
  ip -j -4 route show table main > "${dir}/route4.json"
  ip -j -6 route show table main > "${dir}/route6.json" 2>/dev/null || true
  for f in addr route4 route6; do
    [[ -s "${dir}/${f}.json" ]] || echo '[]' > "${dir}/${f}.json"
  done
  if [[ -r /run/systemd/resolve/resolv.conf ]]; then
    dns_file=/run/systemd/resolve/resolv.conf
  fi
  runtime_dns_json "${dns_file}" "${dns_fallback}" > "${dir}/dns.json"
  if [[ "${DNS_EXPLICIT:-false}" == true ]]; then
    jq --arg d "${DNS_SERVER}" '.addresses = [ $d ]' "${dir}/dns.json" > "${dir}/dns.tmp"
    mv "${dir}/dns.tmp" "${dir}/dns.json"
  fi

  render_netplan_json "${dir}/addr.json" "${dir}/route4.json" \
    "${dir}/route6.json" "${dir}/dns.json" > "${dir}/result.json"

  while IFS= read -r warning; do
    echo "WARN: ${warning}" >&2
  done < <(jq -r '.warnings[]' "${dir}/result.json")
  warn_if_policy_routing
  mapfile -t errors < <(jq -r '.errors[]' "${dir}/result.json")
  if [[ "${#errors[@]}" -gt 0 ]]; then
    for error in "${errors[@]}"; do
      echo "ERROR: ${error}" >&2
    done
    echo "       Describe the network yourself with --network-config <file>." >&2
    exit 1
  fi

  jq '.netplan' "${dir}/result.json" > "${dir}/netplan.yaml"
  echo "==> Network config: rendered from the running system ($(jq -r '.rendered | join(", ")' "${dir}/result.json"))"
  sed 's/^/    /' "${dir}/netplan.yaml"
  validate_with_netplan_generate "${dir}/netplan.yaml"
  RENDERED_NETPLAN="$(jq -c '.netplan' "${dir}/result.json")"
}

# Refuses a cmdline the kernel would truncate, rather than booting into
# an environment with no SSH, no hostname and maybe no network.
check_cmdline_length() {
  local len
  len="$(printf '%s' "${FINAL_CMDLINE}" | wc -c)"
  if [[ "${len}" -le "${CMDLINE_MAX}" ]]; then
    echo "==> Cmdline is ${len}/${CMDLINE_MAX} bytes"
    return 0
  fi
  echo "ERROR: kernel cmdline is ${len} bytes; the kernel keeps only ${CMDLINE_MAX}." >&2
  echo "       Breakdown:" >&2
  echo "         cloud-init (cc:) blocks: $(printf '%s' "${CC_CONTENT:-}" | wc -c)" >&2
  echo "         network-config=:         $(printf '%s' "${NETWORK_CONFIG_PARAM:-}" | wc -c)" >&2
  echo "         --cmdline extras:        $(printf '%s' "${EXTRA_CMDLINE_ARGS[*]:-}" | wc -c)" >&2
  echo "       To shrink: prefer ed25519 keys (RSA-4096 is ~750 bytes each)," >&2
  echo "       pass fewer keys, or trim --user-data / --network-config." >&2
  exit 1
}

### --remount-local ###

# Overridable so tests can feed a known cmdline.
read_proc_cmdline() {
  cat /proc/cmdline
}

# Reuses the current cmdline verbatim, minus any existing overlayroot
# settings, plus our tmpfs overlay. Everything else -- root=, LUKS,
# LVM names, console, vendor quirks -- is whatever already boots this
# machine, which is the point: we are not trying to re-derive it.
build_remount_cmdline() {
  local current

  current="$(read_proc_cmdline)"
  current="$(sed -E 's/(^| )overlayroot(_cfgdisk)?=[^[:space:]]*//g' <<<"${current}")"

  FINAL_CMDLINE="${current} overlayroot=tmpfs:size=${OVERLAY_SIZE}"
  FINAL_CMDLINE+=" overlayroot_cfgdisk=disabled"
  if [[ "${#EXTRA_CMDLINE_ARGS[@]}" -gt 0 ]]; then
    FINAL_CMDLINE+=" ${EXTRA_CMDLINE_ARGS[*]}"
  fi
  FINAL_CMDLINE="$(tr -s ' ' <<<"${FINAL_CMDLINE}" | sed -E 's/^ +| +$//g')"
  readonly FINAL_CMDLINE

  check_cmdline_length
  echo "==> Final cmdline:"
  echo "    ${FINAL_CMDLINE}"
}

do_remount_local() {
  KERNEL_FILE="${KERNEL_OVERRIDE:-${LOCAL_KERNEL_DEFAULT}}"
  INITRD_FILE="${INITRD_OVERRIDE:-${LOCAL_INITRD_DEFAULT}}"

  local f
  for f in "${KERNEL_FILE}" "${INITRD_FILE}"; do
    if [[ ! -f "${f}" ]]; then
      echo "ERROR: ${f} not found. Pass --kernel / --initrd explicitly." >&2
      exit 1
    fi
  done
  echo "==> kernel: ${KERNEL_FILE}"
  echo "==> initrd: ${INITRD_FILE}"

  validate_kernel_image "${KERNEL_FILE}"
  build_remount_cmdline

  # -s uses the kexec_file_load syscall, which is what works under
  # kernel lockdown / secure boot. Falls back to the older interface.
  echo "==> Loading kexec image (not yet executing)"
  if ! kexec -s -l "${KERNEL_FILE}" --initrd="${INITRD_FILE}" \
      --command-line="${FINAL_CMDLINE}" 2>/dev/null; then
    echo "WARN: kexec -s failed; retrying without it." >&2
    kexec -l "${KERNEL_FILE}" --initrd="${INITRD_FILE}" \
      --command-line="${FINAL_CMDLINE}"
  fi
  echo "==> kexec image loaded successfully."

  confirm_remount_local
}

confirm_remount_local() {
  echo
  echo "About to reboot into the running kernel with a ${OVERLAY_SIZE} tmpfs overlay."
  echo "Writes after this point land in RAM and are lost on the next boot."
  echo "The root disk stays mounted, read-only beneath the overlay."
  read -r -p "Type 'yes' to reboot now, anything else to abort: " answer
  if [[ "${answer}" != "yes" ]]; then
    echo "Aborted. Image is loaded; run 'systemctl kexec' when ready."
    exit 0
  fi
  execute_kexec
}

# systemctl kexec rather than kexec -e: it runs the normal shutdown
# first, so filesystems are flushed and remounted read-only before the
# jump. kexec -e yanks the machine with the disk still dirty, which in
# the ephemeral mode means imaging a filesystem with an unreplayed
# journal -- recoverable, but not the clean point-in-time copy the whole
# exercise is for.
execute_kexec() {
  if command -v systemctl >/dev/null 2>&1; then
    systemctl kexec
  else
    echo "WARN: no systemctl; using kexec -e, which does not unmount cleanly." >&2
    echo "      Disks may be imaged with a dirty filesystem." >&2
    kexec -e
  fi
}

# The ip= device field is "BOOTIF", not the interface name: casper's
# initrd resolves that placeholder against BOOTIF=01-<mac> by matching
# /sys/class/net/*/address, so it doesn't matter what the kexec'd kernel
# ends up calling the NIC. DNS matters more in 3.00 than it did in 2.x,
# because the squashfs URL is now a hostname rather than a rack
# controller IP: the initrd has to resolve it.
#
# With a rendered or file network config both ip= and BOOTIF= are left
# out. The squashfs is stacked, so the initramfs needs no network, and
# without ip= cloud-initramfs-dyn-netconf stays idle instead of putting
# the host's address on the raw NIC -- which could clash with the
# VLAN/bond the file defines. The file is then the only network config.
build_cmdline() {
  local ip_field="ip=${STATIC_IP}::${GATEWAY}:${NETMASK}:${HOSTNAME}:BOOTIF:off:${DNS_SERVER}"
  local bootif_field="BOOTIF=01-${LOCAL_MAC}"
  local root_field

  if [[ "$(current_network_mode)" == render || "$(current_network_mode)" == file ]]; then
    ip_field=""
    bootif_field=""
  fi

  if [[ "${STACK_SQUASHFS}" == true ]]; then
    # Our injected local-top script attaches /<name> to the loop device
    # before mountroot runs, so the stock root mount path handles it.
    root_field="root=${RESCUE_LOOP_DEV} rootfstype=squashfs"
    root_field+=" rescue.squashfs=/${SQUASHFS_INITRAMFS_NAME}"
    if [[ "${IMAGE_PROVIDER:-maas}" == cloud-image ]]; then
      # The cloud image's fstab mounts LABEL=UEFI (and names
      # LABEL=cloudimg-rootfs). On a host that was itself deployed from
      # a cloud image those labels exist on the LOCAL disk, and systemd
      # would mount its ESP. fstab=no makes the generator ignore fstab.
      root_field+=" fstab=no"
    fi
  else
    root_field="root=squash:${SQUASHFS_BOOT_URL}"
  fi

  FINAL_CMDLINE="nomodeset ${root_field} ${ip_field} ip6=off"
  FINAL_CMDLINE+=" ${NETWORK_CONFIG_PARAM}"
  FINAL_CMDLINE+=" overlayroot=tmpfs overlayroot_cfgdisk=disabled ro"
  FINAL_CMDLINE+=" ${CC_CONTENT} ${bootif_field} ${CONSOLE_ARGS} rootwait"
  if [[ "${#EXTRA_CMDLINE_ARGS[@]}" -gt 0 ]]; then
    FINAL_CMDLINE+=" ${EXTRA_CMDLINE_ARGS[*]}"
  fi
  FINAL_CMDLINE="$(tr -s ' ' <<<"${FINAL_CMDLINE}" | sed -E 's/^ +| +$//g')"
  readonly FINAL_CMDLINE

  check_cmdline_length
  echo "==> Final cmdline:"
  echo "    ${FINAL_CMDLINE}"
}

load_kexec() {
  echo "==> Loading kexec image (not yet executing)"
  kexec -l "${KERNEL_FILE}" --initrd="${INITRD_FILE}" \
    --command-line="${FINAL_CMDLINE}"
  echo "==> kexec image loaded successfully."
}

confirm_and_execute() {
  echo
  echo "About to jump into the rescue kernel on ${LOCAL_IFACE}."
  echo "This ends the current session immediately and is not reversible"
  echo "without a full power-cycle back through this OS's own boot."
  read -r -p "Type 'yes' to execute kexec now, anything else to abort: " answer
  if [[ "${answer}" != "yes" ]]; then
    echo "Aborted. Image is loaded; run 'systemctl kexec' when ready."
    exit 0
  fi
  execute_kexec
}

# Only run main when executed directly, not when sourced (e.g. by tests).
if [[ "${BASH_SOURCE[0]}" == "${0}" ]]; then
  main "$@"
fi
