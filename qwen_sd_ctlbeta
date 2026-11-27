#!/usr/bin/env bash
set -euo pipefail

# ==============================================================================
# 通用 Linux 服务器本地无限制生图/改图一键管理脚本 (Universal Edition)
# 支持架构: x86_64 (Intel/AMD AVX2/AVX-512) & ARM64/aarch64 (Ampere/RK3588/RPi)
# 支持系统: Debian 11-13 / Ubuntu 20.04-24.04 / Armbian / Rocky / Alma / Fedora
# 支持配置: 6GB/10GB 小内存自建机、Netcup VPS/RS、OVH 独服、大内存工作站
# 引擎模型: stable-diffusion.cpp + FastAPI (Qwen-Image-2.1-Turbo & SD 1.5/SDXL)
# ==============================================================================

CONF_FILE="/etc/qwen-sd-webui.conf"
SYSCTL_FILE="/etc/sysctl.d/99-qwen-sd.conf"
SERVICE_NAME="qwen-sd-webui"
SERVICE_FILE="/etc/systemd/system/${SERVICE_NAME}.service"
SWAP_FILE="/swapfile-qwensd"

# 默认配置（若已安装会从 CONF_FILE 自动覆盖）
APP_DIR="/opt/qwen-sd-webui"
WEB_HOST="0.0.0.0"
WEB_PORT="7860"
BUILD_MODE="native"     # native | portable
CPU_QUOTA_PCT="0"       # 0=不限制，或如 80=限制总CPU使用率80%(适配Netcup普通VPS)

RED='\033[0;31m'; GREEN='\033[0;32m'; YELLOW='\033[1;33m'; CYAN='\033[0;36m'; NC='\033[0m'

log_info()  { echo -e "${CYAN}[INFO]${NC} $*"; }
log_ok()    { echo -e "${GREEN}[OK]${NC} $*"; }
log_warn()  { echo -e "${YELLOW}[WARN]${NC} $*"; }
log_err()   { echo -e "${RED}[ERR]${NC} $*" >&2; }

load_conf() {
    if [[ -f "$CONF_FILE" ]]; then
        # shellcheck disable=SC1090
        source "$CONF_FILE"
    fi
}

save_conf() {
    cat <<EOF > "$CONF_FILE"
APP_DIR="${APP_DIR}"
WEB_HOST="${WEB_HOST}"
WEB_PORT="${WEB_PORT}"
BUILD_MODE="${BUILD_MODE}"
CPU_QUOTA_PCT="${CPU_QUOTA_PCT}"
EOF
    chmod 644 "$CONF_FILE"
}

require_root() {
    if [[ $EUID -ne 0 ]]; then
        log_err "请使用 root 权限运行: sudo bash $0"
        exit 1
    fi
}

get_arch() {
    uname -m
}

get_virt() {
    if command -v systemd-detect-virt >/dev/null 2>&1; then
        systemd-detect-virt 2>/dev/null || echo "none"
    else
        echo "unknown"
    fi
}

get_ram_mb() {
    awk '/MemTotal/ {printf "%d", $2 / 1024}' /proc/meminfo
}

get_ram_gb() {
    awk '/MemTotal/ {printf "%d", ($2 + 524288) / 1048576}' /proc/meminfo
}

get_cpu_features() {
    local arch
    arch=$(get_arch)
    if [[ "$arch" == "x86_64" ]]; then
        local f=""
        grep -qi "avx512f" /proc/cpuinfo && f+="AVX-512 "
        grep -qi "avx2"    /proc/cpuinfo && f+="AVX2 "
        grep -qi "fma"     /proc/cpuinfo && f+="FMA "
        grep -qi "avx"     /proc/cpuinfo && [[ -z "$f" ]] && f+="AVX "
        echo "${f:-SSE4/Generic}"
    elif [[ "$arch" == "aarch64" || "$arch" == "arm64" ]]; then
        local f="NEON "
        grep -qi "asimddp" /proc/cpuinfo && f+="DotProd "
        grep -qi "i8mm"    /proc/cpuinfo && f+="I8MM "
        grep -qi "fphp"    /proc/cpuinfo && f+="FP16 "
        grep -qi "sve"     /proc/cpuinfo && f+="SVE "
        echo "$f"
    else
        echo "Generic"
    fi
}

get_recommended_threads() {
    local arch log_c phys_c
    arch=$(get_arch)
    log_c=$(nproc 2>/dev/null || echo 4)

    # ARM64 或非 x86 架构通常无超线程，直接以逻辑核心数为准
    if [[ "$arch" == "aarch64" || "$arch" == "arm64" ]]; then
        echo "$log_c"
        return
    fi

    # x86_64 尝试获取物理核心数（避免超线程争抢 AVX2 单元）
    phys_c=$(lscpu -p=CORE,SOCKET 2>/dev/null | grep -v '^#' | sort -u | wc -l || true)
    if [[ -n "$phys_c" && "$phys_c" -ge 1 && "$phys_c" -le "$log_c" ]]; then
        # 部分 KVM VPS 会把所有 vCPU 的 CORE/SOCKET 都标成 0，导致 phys_c=1，此时回退到 nproc
        if [[ "$phys_c" -eq 1 && "$log_c" -gt 2 ]]; then
            echo "$log_c"
        else
            echo "$phys_c"
        fi
    else
        echo "$log_c"
    fi
}

gen_rand_pass() {
    if command -v openssl >/dev/null 2>&1; then
        openssl rand -base64 24 | tr -dc 'A-Za-z0-9' | cut -c1-16
    else
        python3 -c 'import secrets, string; print("".join(secrets.choice(string.ascii_letters+string.digits) for _ in range(16)))'
    fi
}

install_system_deps() {
    log_info "检测包管理器并安装编译与运行依赖..."
    if command -v apt-get >/dev/null 2>&1; then
        export DEBIAN_FRONTEND=noninteractive
        apt-get update -qq
        apt-get install -y git cmake build-essential pkg-config curl openssl \
            python3 python3-venv python3-pip libwebp-dev ca-certificates
    elif command -v dnf >/dev/null 2>&1; then
        dnf install -y git cmake gcc gcc-c++ make pkgconfig curl openssl \
            python3 python3-pip libwebp-devel ca-certificates
    elif command -v yum >/dev/null 2>&1; then
        yum install -y git cmake3 gcc gcc-c++ make pkgconfig curl openssl \
            python3 python3-pip libwebp-devel ca-certificates
    else
        log_err "未检测到支持的包管理器 (apt / dnf / yum)，请手动安装 git cmake g++ curl python3。"
        exit 1
    fi
}

choose_install_dir() {
    local root_gb home_gb default_dir="/opt/qwen-sd-webui"
    root_gb=$(df -BG --output=avail / 2>/dev/null | tail -n1 | tr -dc '0-9' || echo "0")
    home_gb=$(df -BG --output=avail /home 2>/dev/null | tail -n1 | tr -dc '0-9' || echo "0")

    echo
    log_info "磁盘可用空间检测: / = ${root_gb}GB, /home = ${home_gb}GB"
    if [[ "${root_gb:-0}" -lt 30 && "${home_gb:-0}" -gt "${root_gb:-0}" ]]; then
        log_warn "根分区 (/) 空间较小，建议安装到空间更大的 /home 分区！"
        default_dir="/home/qwen-sd-webui"
    fi

    read -rp "请输入安装与模型存储目录 [默认: ${default_dir}]: " custom_dir
    APP_DIR="${custom_dir:-$default_dir}"
    APP_DIR="${APP_DIR%/}"
}

setup_swap_smart() {
    local ram_gb swap_mb swap_gb total_gb need_gb avail_gb fs_type
    ram_gb=$(get_ram_gb)
    swap_mb=$(free -m | awk '/^Swap:/ {print $2}')
    swap_gb=$(( (${swap_mb:-0} + 512) / 1024 ))
    total_gb=$(( ram_gb + swap_gb ))

    if [[ "$ram_gb" -ge 24 ]]; then
        log_ok "检测到物理内存 ${ram_gb}GB（充足），跳过 Swap 创建。"
        return
    fi

    # 目标总虚拟内存 >= 22GB（确保 6G/10G 机器能稳跑 Qwen-Image-2.1 且编译不 OOM）
    local target_total=22
    if [[ "$total_gb" -ge "$target_total" ]]; then
        log_ok "当前内存 ${ram_gb}GB + Swap ${swap_gb}GB = ${total_gb}GB，满足运行需求。"
        return
    fi

    need_gb=$(( target_total - ram_gb ))
    [[ "$need_gb" -lt 8 ]] && need_gb=8
    [[ "$need_gb" -gt 16 ]] && need_gb=16

    avail_gb=$(df -BG --output=avail / 2>/dev/null | tail -n1 | tr -dc '0-9' || echo "0")
    if [[ "$avail_gb" -le $(( need_gb + 6 )) ]]; then
        local safe_gb=$(( avail_gb - 6 ))
        if [[ "$safe_gb" -lt 4 ]]; then
            log_warn "根分区剩余仅 ${avail_gb}GB，空间不足以创建 Swap！请确保模型盘或物理内存充足。"
            return
        fi
        log_warn "根分区剩余 ${avail_gb}GB，自动将建议 Swap 调整为 ${safe_gb}GB。"
        need_gb="$safe_gb"
    fi

    echo
    log_warn "检测到物理内存为 ${ram_gb}GB（当前 Swap: ${swap_gb}GB）。"
    log_info "小内存服务器（6G/10G）运行千问生图与编译引擎强烈建议配置 ${need_gb}GB 专用 Swap。"
    read -rp "是否立即创建 ${need_gb}GB Swap (${SWAP_FILE})? [Y/n]: " ans
    if [[ ! "${ans:-Y}" =~ ^[Yy]$ ]]; then
        log_warn "已跳过 Swap 创建，若出现 Killed(-9) 报错请重新配置 Swap。"
        return
    fi

    # 若已有旧脚本 Swap 先卸载重建
    if [[ -f "$SWAP_FILE" ]]; then
        swapoff "$SWAP_FILE" 2>/dev/null || true
        rm -f "$SWAP_FILE"
    fi

    fs_type=$(df -T / 2>/dev/null | awk 'NR==2 {print $2}' || echo "ext4")
    log_info "正在创建 ${need_gb}GB Swap (文件系统: ${fs_type})..."
    if [[ "$fs_type" == "btrfs" ]]; then
        truncate -s 0 "$SWAP_FILE"
        chattr +C "$SWAP_FILE" 2>/dev/null || true
        dd if=/dev/zero of="$SWAP_FILE" bs=1M count=$(( need_gb * 1024 )) status=progress
    else
        fallocate -l "${need_gb}G" "$SWAP_FILE" 2>/dev/null || \
            dd if=/dev/zero of="$SWAP_FILE" bs=1M count=$(( need_gb * 1024 )) status=progress
    fi

    chmod 600 "$SWAP_FILE"
    mkswap "$SWAP_FILE" >/dev/null
    swapon "$SWAP_FILE"
    grep -q "$SWAP_FILE" /etc/fstab || echo "${SWAP_FILE} none swap sw 0 0" >> /etc/fstab

    # 针对 mmap 权重加载与小内存 Swap 优化内核参数
    cat <<EOF > "$SYSCTL_FILE"
vm.swappiness=25
vm.vfs_cache_pressure=50
EOF
    sysctl -p "$SYSCTL_FILE" >/dev/null 2>&1 || true
    log_ok "${need_gb}GB Swap 已启用并完成内存页缓存调优。"
}

setup_auth_and_network() {
    local virt log_c
    virt=$(get_virt)
    log_c=$(nproc 2>/dev/null || echo 4)

    echo
    log_info "配置 WebUI 监听地址与公网安全鉴权："
    echo "  1) 监听 0.0.0.0   (局域网/公网直连，强制开启账密登录) [推荐]"
    echo "  2) 监听 127.0.0.1 (仅本机，配合 SSH 隧道 / Tailscale / Nginx 反代)"
    read -rp "请选择监听模式 [1-2, 默认 1]: " listen_mode
    if [[ "${listen_mode:-1}" == "2" ]]; then
        WEB_HOST="127.0.0.1"
    else
        WEB_HOST="0.0.0.0"
    fi

    read -rp "请输入 WebUI 端口 [默认: ${WEB_PORT}]: " custom_port
    if [[ -n "$custom_port" && "$custom_port" =~ ^[0-9]+$ ]]; then
        WEB_PORT="$custom_port"
    fi

    local auth_enabled="1"
    local auth_user="admin"
    local auth_pass=""

    if [[ "$WEB_HOST" == "127.0.0.1" ]]; then
        read -rp "仅本机监听模式下，是否仍开启网页账密认证? [y/N]: " keep_auth
        [[ ! "${keep_auth:-N}" =~ ^[Yy]$ ]] && auth_enabled="0"
    fi

    if [[ "$auth_enabled" == "1" ]]; then
        read -rp "设置登录用户名 [默认: admin]: " in_user
        auth_user="${in_user:-admin}"
        read -rp "设置登录密码 [直接回车自动生成 16 位强密码]: " in_pass
        if [[ -z "$in_pass" ]]; then
            auth_pass=$(gen_rand_pass)
            log_ok "已自动生成随机强密码，请务必保存："
        else
            auth_pass="$in_pass"
        fi
    fi

    echo
    log_info "配置 CPU 占用上限（防止 Netcup 等普通 VPS 长时间 100% 满载触发公平使用限频）："
    if [[ "$virt" != "none" ]]; then
        echo -e "  检测到当前运行在虚拟化环境 (${YELLOW}${virt}${NC})，当前共 ${log_c} 核。"
        echo "  - 本地自建机 / 独服 / Netcup RS：建议输入 0 (不限流，满速跑)"
        echo "  - Netcup 普通 VPS / 共享型云主机：建议输入 75 或 80 (限制总算力上限)"
    else
        echo "  检测到物理机环境，默认输入 0 (不限流)。"
    fi
    read -rp "请输入单服务总 CPU 占用上限百分比 [0=不限, 或 50-95, 默认 ${CPU_QUOTA_PCT}]: " in_quota
    if [[ -n "$in_quota" && "$in_quota" =~ ^[0-9]+$ ]]; then
        if [[ "$in_quota" -ge 50 && "$in_quota" -le 95 ]]; then
            CPU_QUOTA_PCT="$in_quota"
        else
            CPU_QUOTA_PCT="0"
        fi
    fi

    local ram_gb def_threads def_mmap def_max_edge
    ram_gb=$(get_ram_gb)
    def_threads=$(get_recommended_threads)
    # 20GB 以下内存默认开启 --mmap，10GB 以下默认推理长边上限 768px
    if [[ "$ram_gb" -le 20 ]]; then def_mmap="1"; else def_mmap="1"; fi
    if [[ "$ram_gb" -le 10 ]]; then def_max_edge="768"; else def_max_edge="1024"; fi

    mkdir -p "$APP_DIR"
    cat <<EOF > "${APP_DIR}/.env"
APP_DIR=${APP_DIR}
AUTH_ENABLED=${auth_enabled}
AUTH_USER=${auth_user}
AUTH_PASS=${auth_pass}
DEFAULT_THREADS=${def_threads}
DEFAULT_MMAP=${def_mmap}
DEFAULT_MAX_EDGE=${def_max_edge}
EOF
    chmod 600 "${APP_DIR}/.env"
    save_conf

    if [[ "$auth_enabled" == "1" ]]; then
        echo -e "  ${YELLOW}WebUI 登录账号:${NC} ${GREEN}${auth_user}${NC}"
        echo -e "  ${YELLOW}WebUI 登录密码:${NC} ${GREEN}${auth_pass}${NC}"
    fi
}

write_webui_app() {
    log_info "正在写入全平台适配 FastAPI WebUI (${APP_DIR}/app.py)..."
    cat <<'PY_EOF' > "${APP_DIR}/app.py"
import os
import re
import glob
import uuid
import json
import time
import base64
import platform
import secrets
import threading
import subprocess
from io import BytesIO
from collections import deque
from pathlib import Path
from typing import Optional

from PIL import Image, ImageOps
from fastapi import FastAPI, UploadFile, File, Form, HTTPException, Request
from fastapi.responses import HTMLResponse, JSONResponse, Response
from fastapi.staticfiles import StaticFiles

APP_DIR = Path(os.environ.get("APP_DIR", str(Path(__file__).resolve().parent)))
SD_CLI = APP_DIR / "bin" / "sd-cli"
MODELS_QWEN = APP_DIR / "models" / "qwen21"
MODELS_QWEN_DIT = MODELS_QWEN / "dit"
MODELS_QWEN_LLM = MODELS_QWEN / "llm"
MODELS_QWEN_MMPROJ = MODELS_QWEN / "mmproj"
MODELS_QWEN_VAE = MODELS_QWEN / "vae"
MODELS_LORAS = APP_DIR / "models" / "loras"
MODELS_SD = APP_DIR / "models" / "sd"
if not MODELS_SD.exists() and (APP_DIR / "models" / "sd15").exists():
    MODELS_SD = APP_DIR / "models" / "sd15"
UPLOADS_DIR = APP_DIR / "uploads"
OUTPUTS_DIR = APP_DIR / "outputs"

AUTH_ENABLED = os.environ.get("AUTH_ENABLED", "0") == "1"
AUTH_USER = os.environ.get("AUTH_USER", "admin")
AUTH_PASS = os.environ.get("AUTH_PASS", "")
DEFAULT_THREADS = int(os.environ.get("DEFAULT_THREADS", str(os.cpu_count() or 4)))
DEFAULT_MMAP = os.environ.get("DEFAULT_MMAP", "1") == "1"
DEFAULT_MAX_EDGE = int(os.environ.get("DEFAULT_MAX_EDGE", "768"))

for d in (
    MODELS_QWEN, MODELS_QWEN_DIT, MODELS_QWEN_LLM,
    MODELS_QWEN_MMPROJ, MODELS_QWEN_VAE, MODELS_LORAS,
    MODELS_SD, UPLOADS_DIR, OUTPUTS_DIR,
):
    d.mkdir(parents=True, exist_ok=True)

# Qwen-Image-2.1-Turbo 官方 8 步 sigma 调度序列
QWEN_TURBO_SIGMAS = "1.0,0.978453,0.954180,0.926626,0.895080,0.845148,0.704534,0.414568,0.0"

app = FastAPI(title="Universal Qwen / SD Studio")


@app.middleware("http")
async def basic_auth_middleware(request: Request, call_next):
    if AUTH_ENABLED and AUTH_PASS:
        auth = request.headers.get("Authorization", "")
        valid = False
        if auth.startswith("Basic "):
            try:
                decoded = base64.b64decode(auth[6:]).decode("utf-8")
                user, pwd = decoded.split(":", 1)
                if secrets.compare_digest(user, AUTH_USER) and secrets.compare_digest(pwd, AUTH_PASS):
                    valid = True
            except Exception:
                valid = False
        if not valid:
            return Response(
                content="Authentication Required",
                status_code=401,
                headers={"WWW-Authenticate": 'Basic realm="Universal Qwen-SD Studio"'},
            )
    return await call_next(request)


app.mount("/outputs", StaticFiles(directory=str(OUTPUTS_DIR)), name="outputs")

# --------------------------- 自动探测 sd-cli 支持的参数 ---------------------------
_cli_caps_cache = None


def get_cli_caps() -> set:
    global _cli_caps_cache
    if _cli_caps_cache is not None:
        return _cli_caps_cache
    caps = set()
    if SD_CLI.exists():
        try:
            r = subprocess.run(
                [str(SD_CLI), "--help"],
                stdout=subprocess.PIPE,
                stderr=subprocess.STDOUT,
                text=True,
                timeout=8,
            )
            help_txt = r.stdout or ""
            for flag in ("--mmap", "--diffusion-fa", "--vae-tiling", "--vae-conv-direct", "--lora-model-dir"):
                if flag in help_txt:
                    caps.add(flag)
        except Exception:
            pass
    _cli_caps_cache = caps
    return caps


# --------------------------- 任务队列与后台调度 ---------------------------
queue_cond = threading.Condition()
pending_queue = deque()
current_job = None
last_finished = {
    "job_id": "",
    "status": "idle",
    "output_url": "",
    "error": "",
    "logs": ["就绪。支持 x86_64/ARM64、小内存 mmap 优化、原图 1:1 尺寸还原与多任务队列。"],
    "progress": 0,
    "step_text": "",
    "elapsed_sec": 0,
}


def align32(val: int, min_v: int = 256, max_v: int = 2048) -> int:
    val = max(min_v, min(max_v, val))
    return max(min_v, min(max_v, int(round(val / 32) * 32)))


def get_sys_stats():
    mem_total = mem_avail = swap_total = swap_free = 0
    try:
        with open("/proc/meminfo", "r", encoding="utf-8") as f:
            for line in f:
                if line.startswith("MemTotal:"):
                    mem_total = int(line.split()[1])
                elif line.startswith("MemAvailable:"):
                    mem_avail = int(line.split()[1])
                elif line.startswith("SwapTotal:"):
                    swap_total = int(line.split()[1])
                elif line.startswith("SwapFree:"):
                    swap_free = int(line.split()[1])
    except Exception:
        pass
    load1 = "0.00"
    try:
        load1 = Path("/proc/loadavg").read_text(encoding="utf-8").split()[0]
    except Exception:
        pass
    return {
        "arch": platform.machine(),
        "ram_used_gb": round((mem_total - mem_avail) / 1048576, 1) if mem_total else 0.0,
        "ram_total_gb": round(mem_total / 1048576, 1) if mem_total else 0.0,
        "swap_used_gb": round((swap_total - swap_free) / 1048576, 1) if swap_total else 0.0,
        "swap_total_gb": round(swap_total / 1048576, 1) if swap_total else 0.0,
        "load1": load1,
        "logical_cores": os.cpu_count() or 4,
        "default_threads": DEFAULT_THREADS,
        "default_mmap": DEFAULT_MMAP,
        "default_max_edge": DEFAULT_MAX_EDGE,
        "cli_caps": sorted(list(get_cli_caps())),
    }


def _resolve_qwen_file(rel_or_name: str) -> Path:
    base = str(MODELS_QWEN.resolve())
    p = (MODELS_QWEN / rel_or_name).resolve()
    if str(p).startswith(base + os.sep) and p.exists():
        return p
    raise HTTPException(400, f"模型文件不存在: {rel_or_name}")


def scan_models():
    qwen_dit, qwen_llm, qwen_mmproj, qwen_vae = [], [], [], []

    # 1. 优先读取分类子目录（支持任意命名的无审查/去审查权重）
    for p in sorted(MODELS_QWEN_DIT.glob("*.gguf")):
        qwen_dit.append(f"dit/{p.name}")
    for p in sorted(MODELS_QWEN_LLM.glob("*.gguf")):
        qwen_llm.append(f"llm/{p.name}")
    for p in sorted(MODELS_QWEN_MMPROJ.glob("*.gguf")):
        qwen_mmproj.append(f"mmproj/{p.name}")
    for p in sorted(list(MODELS_QWEN_VAE.glob("*.safetensors")) + list(MODELS_QWEN_VAE.glob("*.gguf"))):
        qwen_vae.append(f"vae/{p.name}")

    # 2. 兼容原平铺在 models/qwen21/ 目录下的权重（大小写不敏感）
    for p in sorted(MODELS_QWEN.iterdir()):
        if not p.is_file():
            continue
        low = p.name.lower()
        if low.endswith(".safetensors") and "vae" in low:
            qwen_vae.append(p.name)
        elif low.endswith(".gguf"):
            if "mmproj" in low:
                qwen_mmproj.append(p.name)
            elif any(k in low for k in ("qwen3-vl", "qwen2.5-vl", "vl-", "abliterat", "uncensor", "heretic", "obliterat", "instruct")):
                if "qwen-image" in low or "dit" in low:
                    qwen_dit.append(p.name)
                else:
                    qwen_llm.append(p.name)
            elif "vae" in low:
                qwen_vae.append(p.name)
            else:
                qwen_dit.append(p.name)

    # 自动将无审查/去审查权重置顶作为默认首选
    def _uncensored_first(name: str):
        low = name.lower()
        is_free = any(k in low for k in ("abliterat", "uncensor", "heretic", "unrestrict", "obliterat", "huihui"))
        is_sub = "/" in name
        return (0 if is_free else 1, 0 if is_sub else 1, low)

    qwen_dit.sort(key=_uncensored_first)
    qwen_llm.sort(key=_uncensored_first)
    qwen_mmproj.sort(key=lambda x: (0 if "/" in x else 1, x.lower()))
    qwen_vae.sort(key=lambda x: (0 if "/" in x else 1, x.lower()))

    sd_ckpts = sorted([
        Path(p).name
        for p in (
            glob.glob(str(MODELS_SD / "*.safetensors"))
            + glob.glob(str(MODELS_SD / "*.ckpt"))
            + glob.glob(str(MODELS_SD / "*.gguf"))
        )
    ])
    loras = sorted([
        p.stem for p in list(MODELS_LORAS.glob("*.safetensors")) + list(MODELS_LORAS.glob("*.gguf"))
    ])
    return {
        "qwen": {
            "dit": qwen_dit,
            "llm": qwen_llm,
            "mmproj": qwen_mmproj,
            "vae": qwen_vae,
            "ready_t2i": bool(qwen_dit and qwen_llm and qwen_vae),
            "ready_edit": bool(qwen_dit and qwen_llm and qwen_vae and qwen_mmproj),
        },
        "sd": {
            "checkpoints": sd_ckpts,
            "ready": bool(sd_ckpts),
        },
        "loras": loras,
        "sys": get_sys_stats(),
    }


def list_gallery():
    items = []
    for p in sorted(OUTPUTS_DIR.glob("*.png"), key=lambda x: x.stat().st_mtime, reverse=True)[:60]:
        meta = {}
        meta_file = p.with_suffix(".json")
        if meta_file.exists():
            try:
                meta = json.loads(meta_file.read_text(encoding="utf-8"))
            except Exception:
                pass
        items.append({
            "filename": p.name,
            "url": f"/outputs/{p.name}",
            "mtime": int(p.stat().st_mtime),
            "size_kb": round(p.stat().st_size / 1024, 1),
            "meta": meta,
        })
    return items


def cleanup_old_uploads():
    cutoff = time.time() - 86400
    for p in UPLOADS_DIR.glob("ref_*"):
        try:
            if p.stat().st_mtime < cutoff:
                p.unlink()
        except Exception:
            pass


def parse_progress(line: str, job: dict) -> bool:
    low = line.lower()
    m = re.search(r"(?:\|\s*|step\s+)(\d+)\s*/\s*(\d+)", line, re.IGNORECASE)
    if m:
        cur, total = int(m.group(1)), int(m.group(2))
        if 0 <= cur <= total and total > 0:
            pct = max(5, min(95, int(cur * 90 / total) + 5))
            sp_m = re.search(r"(\d+(?:\.\d+)?\s*(?:s/it|it/s))", line)
            sp = f" · {sp_m.group(1)}" if sp_m else ""
            job["progress"] = pct
            job["step_text"] = f"采样步数 {cur}/{total}{sp}"
            return True
    if "loading" in low and ("model" in low or "tensors" in low or "llm" in low or "vae" in low):
        if job["progress"] < 10:
            job["progress"] = 8
            job["step_text"] = "加载模型权重中..."
    elif "conditioning" in low or "get_learned_condition" in low or "encode" in low:
        if job["progress"] < 18:
            job["progress"] = 15
            job["step_text"] = "编码提示词与参考图..."
    elif "sampling" in low and job["progress"] < 20:
        job["progress"] = 20
        job["step_text"] = "扩散采样中..."
    elif "decoding latents" in low or ("vae" in low and "decode" in low):
        job["progress"] = 96
        job["step_text"] = "VAE 解码出图中..."
    return False


def worker_loop():
    global current_job
    while True:
        with queue_cond:
            while not pending_queue:
                queue_cond.wait()
            job = pending_queue.popleft()
            current_job = job
            job["status"] = "running"
            job["started_at"] = time.time()
            job["progress"] = 5
            job["step_text"] = "启动推理进程..."

        cleanup_old_uploads()
        out_path: Path = job["output_path"]
        post_resize = job.get("post_resize")
        try:
            proc = subprocess.Popen(
                job["cmd"],
                stdout=subprocess.PIPE,
                stderr=subprocess.STDOUT,
                text=True,
                bufsize=1,
                universal_newlines=True,
            )
            job["proc"] = proc

            assert proc.stdout is not None
            for line in proc.stdout:
                clean = re.sub(r"\x1b\[[0-9;]*[a-zA-Z]", "", line).strip()
                if not clean:
                    continue
                is_bar = parse_progress(clean, job)
                if is_bar and job["logs"] and "|" in job["logs"][-1] and "/" in job["logs"][-1]:
                    job["logs"][-1] = clean
                else:
                    job["logs"].append(clean)
                    if len(job["logs"]) > 250:
                        job["logs"] = job["logs"][-250:]

            ret = proc.wait()
            elapsed = int(time.time() - job["started_at"])
            if ret == 0 and out_path.exists():
                if post_resize:
                    with Image.open(out_path) as out_im:
                        if out_im.size != post_resize:
                            job["logs"].append(
                                f"[POST] 精确还原原图尺寸: {out_im.size[0]}x{out_im.size[1]} -> {post_resize[0]}x{post_resize[1]}"
                            )
                            out_im.resize(post_resize, Image.Resampling.LANCZOS).save(out_path, format="PNG")
                    job["meta"]["width"], job["meta"]["height"] = post_resize

                job["meta"]["elapsed_sec"] = elapsed
                out_path.with_suffix(".json").write_text(
                    json.dumps(job["meta"], ensure_ascii=False, indent=2), encoding="utf-8"
                )
                last_finished.update({
                    "job_id": job["id"],
                    "status": "done",
                    "output_url": f"/outputs/{out_path.name}",
                    "error": "",
                    "logs": job["logs"][-80:],
                    "progress": 100,
                    "step_text": f"完成 ({job['meta']['width']}×{job['meta']['height']}) · 耗时 {elapsed}s",
                    "elapsed_sec": elapsed,
                })
            elif job["status"] == "cancelled":
                last_finished.update({
                    "job_id": job["id"],
                    "status": "cancelled",
                    "output_url": last_finished.get("output_url", ""),
                    "error": "任务已被手动中止",
                    "logs": job["logs"][-80:],
                    "progress": 0,
                    "step_text": "已中止",
                    "elapsed_sec": elapsed,
                })
            else:
                hint = ""
                if ret in (-9, 137):
                    hint = "（检测到 OOM 被系统查杀，请开启 mmap、降低分辨率或增大 Swap）"
                elif ret in (-4, 132):
                    hint = "（检测到 CPU 指令集不兼容 SIGILL，请在脚本菜单执行 [6] 切换 Portable 兼容编译）"
                last_finished.update({
                    "job_id": job["id"],
                    "status": "error",
                    "output_url": last_finished.get("output_url", ""),
                    "error": f"推理进程异常退出 (exit={ret}) {hint}",
                    "logs": job["logs"][-80:],
                    "progress": 0,
                    "step_text": "失败",
                    "elapsed_sec": elapsed,
                })
        except Exception as e:
            last_finished.update({
                "job_id": job["id"],
                "status": "error",
                "error": str(e),
                "logs": job["logs"][-80:],
                "progress": 0,
                "step_text": "异常",
            })
        finally:
            with queue_cond:
                job["proc"] = None
                current_job = None


threading.Thread(target=worker_loop, daemon=True).start()


@app.get("/api/models")
def api_models():
    return JSONResponse(scan_models())


@app.get("/api/status")
def api_status():
    with queue_cond:
        q_preview = [
            {
                "id": j["id"],
                "engine": j["meta"]["engine"],
                "prompt": j["meta"]["prompt"][:60],
                "size": f"{j['meta']['gen_width']}x{j['meta']['gen_height']}",
            }
            for j in pending_queue
        ]
        if current_job:
            elapsed = int(time.time() - current_job["started_at"])
            data = {
                "running": True,
                "job_id": current_job["id"],
                "status": "running",
                "engine": current_job["meta"]["engine"],
                "prompt": current_job["meta"]["prompt"],
                "elapsed_sec": elapsed,
                "progress": current_job["progress"],
                "step_text": current_job["step_text"] or "加载权重 / 编码中...",
                "output_url": last_finished.get("output_url", ""),
                "error": "",
                "logs": current_job["logs"][-50:],
                "queue": q_preview,
                "sys": get_sys_stats(),
            }
        else:
            data = {
                "running": False,
                "job_id": last_finished.get("job_id", ""),
                "status": last_finished["status"],
                "engine": "",
                "prompt": "",
                "elapsed_sec": last_finished["elapsed_sec"],
                "progress": last_finished["progress"],
                "step_text": last_finished["step_text"],
                "output_url": last_finished["output_url"],
                "error": last_finished["error"],
                "logs": last_finished["logs"][-50:],
                "queue": q_preview,
                "sys": get_sys_stats(),
            }
    return JSONResponse(data)


@app.get("/api/gallery")
def api_gallery():
    return JSONResponse({"items": list_gallery()})


@app.delete("/api/gallery/{filename}")
def api_delete_image(filename: str):
    safe_name = Path(filename).name
    if safe_name != filename or not safe_name.endswith(".png"):
        raise HTTPException(400, "非法文件名")
    target = OUTPUTS_DIR / safe_name
    if not target.exists():
        raise HTTPException(404, "图片不存在")
    target.unlink(missing_ok=True)
    target.with_suffix(".json").unlink(missing_ok=True)
    return JSONResponse({"ok": True})


@app.post("/api/cancel")
def api_cancel():
    with queue_cond:
        if current_job and current_job.get("proc"):
            current_job["status"] = "cancelled"
            proc = current_job["proc"]
            try:
                proc.terminate()
                proc.wait(timeout=3)
            except Exception:
                try:
                    proc.kill()
                except Exception:
                    pass
            return JSONResponse({"ok": True, "msg": "已中止当前任务"})
    return JSONResponse({"ok": False, "msg": "当前无运行中的任务"})


@app.delete("/api/queue/{job_id}")
def api_remove_queue_item(job_id: str):
    with queue_cond:
        for i, j in enumerate(pending_queue):
            if j["id"] == job_id:
                del pending_queue[i]
                return JSONResponse({"ok": True})
    raise HTTPException(404, "队列中未找到该任务")


@app.post("/api/generate")
async def api_generate(
    engine: str = Form(...),
    prompt: str = Form(...),
    negative_prompt: str = Form(""),
    size_mode: str = Form("match_exact"),
    max_edge: int = Form(768),
    width: int = Form(512),
    height: int = Form(512),
    steps: int = Form(8),
    cfg_scale: float = Form(1.0),
    strength: float = Form(0.65),
    seed: int = Form(-1),
    threads: int = Form(0),
    batch_count: int = Form(1),
    use_mmap: str = Form("1"),
    use_fa: str = Form("1"),
    vae_tiling: str = Form("1"),
    model_file: str = Form(""),
    llm_file: str = Form(""),
    ref_output_name: str = Form(""),
    image: Optional[UploadFile] = File(None),
):
    if not SD_CLI.exists():
        raise HTTPException(500, "未找到 sd-cli 可执行文件，请重新安装引擎。")

    prompt = prompt.strip()
    if not prompt:
        raise HTTPException(400, "提示词不能为空")

    max_edge = max(512, min(2048, (max_edge // 32) * 32))
    batch_count = max(1, min(4, batch_count))
    max_t = os.cpu_count() or 8
    use_threads = DEFAULT_THREADS if threads <= 0 else max(1, min(max_t, threads))

    caps = get_cli_caps()
    opt_mmap = (use_mmap in ("1", "true", "on")) and ("--mmap" in caps)
    opt_fa = (use_fa in ("1", "true", "on")) and ("--diffusion-fa" in caps)
    opt_tiling = (vae_tiling in ("1", "true", "on")) and ("--vae-tiling" in caps)

    # 读取参考图（自动矫正手机照片 EXIF 旋转）
    ref_path: Optional[Path] = None
    orig_size = None
    if image and image.filename:
        raw = await image.read()
        if raw:
            im = ImageOps.exif_transpose(Image.open(BytesIO(raw))).convert("RGB")
            orig_size = im.size
            ref_path = UPLOADS_DIR / f"ref_{uuid.uuid4().hex[:10]}.png"
            ref_cap = max(1280, max_edge)
            if max(im.size) > ref_cap:
                im.thumbnail((ref_cap, ref_cap), Image.Resampling.LANCZOS)
            im.save(ref_path, format="PNG")
    elif ref_output_name.strip():
        safe = Path(ref_output_name.strip()).name
        cand = OUTPUTS_DIR / safe
        if cand.exists() and safe.endswith(".png"):
            ref_path = cand
            with Image.open(cand) as im:
                orig_size = im.size

    post_resize = None
    if ref_path and orig_size and size_mode in ("match_ratio", "match_exact"):
        ow, oh = orig_size
        scale = min(1.0, max_edge / max(ow, oh))
        gen_w = align32(int(round(ow * scale)), 256, max_edge)
        gen_h = align32(int(round(oh * scale)), 256, max_edge)
        if size_mode == "match_exact":
            post_resize = (ow, oh)
    else:
        gen_w = align32(width, 256, 2048)
        gen_h = align32(height, 256, 2048)

    models = scan_models()
    created_ids = []

    for idx in range(batch_count):
        out_name = f"{time.strftime('%Y%m%d_%H%M%S')}_{uuid.uuid4().hex[:6]}.png"
        out_path = OUTPUTS_DIR / out_name
        cur_seed = (seed + idx) if seed >= 0 else -1
        cmd = [str(SD_CLI)]

        if engine == "qwen21":
            q = models["qwen"]
            if not q["ready_t2i"]:
                raise HTTPException(400, "Qwen-Image-2.1-Turbo 权重不完整，请先运行脚本下载选项 2。")
            dit = model_file if model_file in q["dit"] else q["dit"][0]
            llm = llm_file if llm_file in q["llm"] else q["llm"][0]
            vae = q["vae"][0]

            cmd += [
                "--diffusion-model", str(_resolve_qwen_file(dit)),
                "--llm", str(_resolve_qwen_file(llm)),
                "--vae", str(_resolve_qwen_file(vae)),
                "--sampling-method", "euler",
                "--steps", str(max(4, min(30, steps))),
                "--cfg-scale", str(cfg_scale),
                "--rng", "cpu",
                "--sampler-rng", "cpu",
                "-t", str(use_threads),
                "-W", str(gen_w),
                "-H", str(gen_h),
                "-p", prompt,
                "-o", str(out_path),
            ]
            if opt_mmap:
                cmd += ["--mmap"]
            if opt_fa:
                cmd += ["--diffusion-fa"]
            if opt_tiling:
                cmd += ["--vae-tiling"]
            if "--lora-model-dir" in caps and models["loras"]:
                cmd += ["--lora-model-dir", str(MODELS_LORAS)]
            if negative_prompt.strip() and cfg_scale > 1.0:
                cmd += ["-n", negative_prompt.strip()]
            if steps == 8:
                cmd += ["--sigmas", QWEN_TURBO_SIGMAS]
            if ref_path:
                if not q["ready_edit"]:
                    raise HTTPException(400, "缺少 mmproj-F16.gguf 视觉权重，无法传图编辑。")
                cmd += ["--llm_vision", str(_resolve_qwen_file(q["mmproj"][0])), "-r", str(ref_path)]
            if cur_seed >= 0:
                cmd += ["-s", str(cur_seed)]

        elif engine == "sd":
            s = models["sd"]
            if not s["ready"]:
                raise HTTPException(400, "未找到 SD 1.5 / SDXL 权重，请先运行脚本选项 3 或放入 models/sd/。")
            ckpt = model_file if model_file in s["checkpoints"] else s["checkpoints"][0]
            cmd += [
                "-m", str(MODELS_SD / ckpt),
                "--sampling-method", "euler_a",
                "--steps", str(max(5, min(60, steps))),
                "--cfg-scale", str(cfg_scale),
                "--rng", "cpu",
                "--sampler-rng", "cpu",
                "-t", str(use_threads),
                "-W", str(gen_w),
                "-H", str(gen_h),
                "-p", prompt,
                "-o", str(out_path),
            ]
            if opt_mmap:
                cmd += ["--mmap"]
            if opt_fa:
                cmd += ["--diffusion-fa"]
            if opt_tiling:
                cmd += ["--vae-tiling"]
            if "--lora-model-dir" in caps and models["loras"]:
                cmd += ["--lora-model-dir", str(MODELS_LORAS)]
            if negative_prompt.strip():
                cmd += ["-n", negative_prompt.strip()]
            if ref_path:
                cmd += ["-i", str(ref_path), "--strength", str(max(0.05, min(1.0, strength)))]
            if cur_seed >= 0:
                cmd += ["-s", str(cur_seed)]
        else:
            raise HTTPException(400, f"未知引擎: {engine}")

        job_id = uuid.uuid4().hex[:8]
        meta = {
            "engine": engine,
            "model_file": model_file,
            "llm_file": llm_file,
            "prompt": prompt,
            "negative_prompt": negative_prompt,
            "size_mode": size_mode,
            "gen_width": gen_w,
            "gen_height": gen_h,
            "width": post_resize[0] if post_resize else gen_w,
            "height": post_resize[1] if post_resize else gen_h,
            "orig_size": f"{orig_size[0]}x{orig_size[1]}" if orig_size else "",
            "steps": steps,
            "cfg_scale": cfg_scale,
            "strength": strength,
            "seed": cur_seed,
            "threads": use_threads,
            "has_ref": bool(ref_path),
            "time": time.strftime("%Y-%m-%d %H:%M:%S"),
        }
        job = {
            "id": job_id,
            "status": "queued",
            "started_at": 0.0,
            "progress": 0,
            "step_text": "排队等待中...",
            "cmd": cmd,
            "output_path": out_path,
            "post_resize": post_resize,
            "meta": meta,
            "logs": [
                f"[JOB {job_id}] 推理尺寸: {gen_w}x{gen_h}"
                + (f" -> 精确还原为原图: {post_resize[0]}x{post_resize[1]}" if post_resize else ""),
                f"[OPT] mmap={opt_mmap} flash_attn={opt_fa} vae_tiling={opt_tiling} threads={use_threads}",
                f"[CMD] {' '.join(cmd)}",
            ],
            "proc": None,
        }
        with queue_cond:
            pending_queue.append(job)
            queue_cond.notify()
        created_ids.append(job_id)

    return JSONResponse({"ok": True, "queued": created_ids, "gen_size": [gen_w, gen_h]})


@app.get("/", response_class=HTMLResponse)
def index():
    return r"""<!DOCTYPE html>
<html lang="zh-CN">
<head>
<meta charset="UTF-8" />
<meta name="viewport" content="width=device-width, initial-scale=1.0" />
<title>全平台本地无限制生图 / 改图工坊</title>
<style>
  :root { --bg:#0b1120; --card:#151e32; --panel:#0f172a; --border:#293548; --text:#e2e8f0; --muted:#94a3b8; --accent:#38bdf8; --ok:#22c55e; --danger:#ef4444; }
  * { box-sizing: border-box; }
  body { margin:0; font-family: system-ui, -apple-system, sans-serif; background:var(--bg); color:var(--text); }
  .wrap { max-width: 1280px; margin: 0 auto; padding: 16px; }
  header { display:flex; justify-content:space-between; align-items:center; flex-wrap:wrap; gap:10px; margin-bottom:14px; border-bottom:1px solid var(--border); padding-bottom:12px; }
  header h1 { margin:0; font-size:1.22rem; color:var(--accent); }
  .stats { display:flex; gap:8px; flex-wrap:wrap; align-items:center; }
  .badge { font-size:0.76rem; padding:4px 10px; border-radius:999px; background:var(--card); border:1px solid var(--border); color:var(--muted); }
  .grid { display:grid; grid-template-columns: 445px 1fr; gap:16px; }
  @media (max-width: 960px) { .grid { grid-template-columns: 1fr; } }
  .card { background:var(--card); border:1px solid var(--border); border-radius:12px; padding:15px; }
  label { display:block; font-size:0.82rem; color:var(--muted); margin:10px 0 4px; }
  input, select, textarea { width:100%; background:var(--panel); color:var(--text); border:1px solid var(--border); border-radius:8px; padding:8px 10px; font-size:0.89rem; }
  textarea { resize:vertical; min-height:82px; }
  .row { display:grid; grid-template-columns:1fr 1fr; gap:10px; }
  .row3 { display:grid; grid-template-columns:1fr 1fr 1fr; gap:8px; }
  .checks { display:flex; gap:14px; flex-wrap:wrap; margin-top:10px; font-size:0.8rem; color:var(--muted); }
  .checks label { display:flex; align-items:center; gap:5px; margin:0; cursor:pointer; }
  .checks input { width:auto; margin:0; }
  .presets { display:flex; gap:6px; flex-wrap:wrap; margin-top:6px; }
  .chip { background:var(--panel); color:var(--muted); border:1px solid var(--border); border-radius:6px; padding:3px 8px; font-size:0.75rem; cursor:pointer; }
  .chip:hover { border-color:var(--accent); color:var(--accent); }
  .btn { cursor:pointer; border:none; border-radius:8px; padding:10px 14px; font-weight:600; font-size:0.92rem; }
  .btn-primary { background:var(--accent); color:#0b1120; width:100%; margin-top:12px; }
  .btn-danger { background:var(--danger); color:#fff; width:100%; margin-top:8px; display:none; }
  .btn-sm { padding:4px 9px; font-size:0.76rem; border-radius:6px; background:var(--panel); color:var(--text); border:1px solid var(--border); cursor:pointer; }
  .btn-sm:hover { border-color:var(--accent); color:var(--accent); }
  .dropzone { border:1.5px dashed var(--border); border-radius:10px; padding:10px; text-align:center; cursor:pointer; background:var(--panel); }
  .dropzone img { max-height:155px; max-width:100%; border-radius:6px; margin-top:6px; display:none; }
  .hint-box { margin-top:6px; padding:7px 10px; border-radius:8px; background:#0c1a2e; border:1px solid #1e3a5f; font-size:0.77rem; color:#bae6fd; display:none; }
  .prog-wrap { width:100%; height:8px; background:var(--panel); border-radius:99px; overflow:hidden; margin:8px 0 12px; border:1px solid var(--border); }
  .prog-bar { height:100%; width:0%; background:linear-gradient(90deg, #38bdf8, #22c55e); transition:width 0.3s; }
  .preview-box { min-height:380px; display:flex; align-items:center; justify-content:center; background:var(--panel); border-radius:10px; border:1px solid var(--border); overflow:hidden; }
  .preview-box img { max-width:100%; max-height:620px; display:none; }
  .log-box { margin-top:12px; background:#020617; border:1px solid var(--border); border-radius:8px; padding:10px; font-family:ui-monospace, monospace; font-size:0.75rem; height:170px; overflow-y:auto; color:#86efac; white-space:pre-wrap; word-break:break-all; }
  .queue-box { margin-top:12px; background:var(--panel); border:1px solid var(--border); border-radius:8px; padding:10px; display:none; }
  .q-item { display:flex; justify-content:space-between; align-items:center; gap:8px; padding:5px 0; border-bottom:1px dashed var(--border); font-size:0.82rem; }
  .q-item:last-child { border-bottom:none; }
  .gallery { margin-top:14px; display:grid; grid-template-columns: repeat(auto-fill, minmax(175px, 1fr)); gap:12px; }
  .g-item { background:var(--panel); border:1px solid var(--border); border-radius:8px; overflow:hidden; display:flex; flex-direction:column; }
  .g-item img { width:100%; height:155px; object-fit:cover; cursor:pointer; display:block; }
  .g-meta { padding:7px; font-size:0.75rem; color:var(--muted); }
  .g-cap { white-space:nowrap; overflow:hidden; text-overflow:ellipsis; margin-bottom:6px; color:var(--text); }
  .g-ops { display:flex; gap:5px; justify-content:space-between; }
  .err { color:#fca5a5; font-size:0.84rem; margin-top:8px; }

  /* 右上角网页内进度豆腐块（非弹窗、不遮挡页面、可关闭） */
  .prog-tile {
    position: fixed;
    top: 14px;
    right: 14px;
    width: 286px;
    max-width: calc(100vw - 28px);
    z-index: 9999;
    background: rgba(21, 30, 50, 0.96);
    backdrop-filter: blur(8px);
    border: 1px solid var(--accent);
    border-radius: 12px;
    padding: 11px 13px;
    box-shadow: 0 10px 28px rgba(0, 0, 0, 0.45);
    display: none;
  }
  .prog-tile.done { border-color: var(--ok); }
  .prog-tile.error { border-color: var(--danger); }
  .pt-head { display:flex; justify-content:space-between; align-items:center; gap:8px; margin-bottom:6px; }
  .pt-title-wrap { display:flex; align-items:center; gap:7px; font-size:0.85rem; }
  .pt-dot { width:8px; height:8px; border-radius:50%; background:var(--accent); flex-shrink:0; }
  .pt-dot.running { animation: ptPulse 1.2s infinite; }
  .pt-dot.done { background:var(--ok); }
  .pt-dot.error { background:var(--danger); }
  .pt-dot.cancelled { background:var(--muted); }
  @keyframes ptPulse { 0%,100% { opacity:1; transform:scale(1); } 50% { opacity:0.35; transform:scale(0.8); } }
  .pt-close { background:transparent; border:none; color:var(--muted); cursor:pointer; font-size:1.15rem; line-height:1; padding:0 4px; border-radius:4px; }
  .pt-close:hover { color:#fff; background:rgba(255,255,255,0.08); }
  .pt-meta { display:flex; justify-content:space-between; align-items:center; font-size:0.76rem; color:var(--muted); margin-bottom:5px; gap:6px; }
  .pt-step { white-space:nowrap; overflow:hidden; text-overflow:ellipsis; }
  .pt-pct { font-weight:700; color:var(--text); flex-shrink:0; }
  .pt-bar-wrap { width:100%; height:6px; background:var(--panel); border-radius:99px; overflow:hidden; border:1px solid var(--border); }
  .pt-bar { height:100%; width:0%; background:linear-gradient(90deg, #38bdf8, #22c55e); transition:width 0.3s; }
  .pt-foot { display:flex; justify-content:space-between; align-items:center; margin-top:7px; font-size:0.73rem; color:var(--muted); gap:6px; }
  .pt-prompt { margin-top:5px; font-size:0.72rem; color:var(--muted); white-space:nowrap; overflow:hidden; text-overflow:ellipsis; }
  .pt-stop { padding:2px 8px; font-size:0.72rem; border-radius:5px; background:rgba(239,68,68,0.18); color:#fca5a5; border:1px solid rgba(239,68,68,0.45); cursor:pointer; }
  .pt-stop:hover { background:var(--danger); color:#fff; }
</style>
</head>
<body>

<div id="progressTile" class="prog-tile" role="status" aria-live="polite">
  <div class="pt-head">
    <div class="pt-title-wrap">
      <span id="ptDot" class="pt-dot running"></span>
      <strong id="ptTitle">生成进度</strong>
    </div>
    <button type="button" class="pt-close" onclick="closeProgressTile()" title="关闭进度显示" aria-label="关闭">×</button>
  </div>
  <div class="pt-meta">
    <span id="ptStep" class="pt-step">准备中...</span>
    <span id="ptPct" class="pt-pct">0%</span>
  </div>
  <div class="pt-bar-wrap"><div id="ptBar" class="pt-bar"></div></div>
  <div id="ptPrompt" class="pt-prompt"></div>
  <div class="pt-foot">
    <span id="ptSub">已耗时 0s</span>
    <button type="button" id="ptStopBtn" class="pt-stop" style="display:none" onclick="cancelJob()">中止</button>
  </div>
</div>

<div class="wrap">
  <header>
    <h1>⚡ 全平台本地无限制生图 / 改图工坊</h1>
    <div class="stats">
      <span class="badge" id="modelBadge">读取模型...</span>
      <span class="badge" id="sysBadge">ARCH | RAM | Swap | Load</span>
      <button type="button" id="reopenTileBtn" class="btn-sm" style="display:none" onclick="reopenProgressTile()">显示进度块</button>
    </div>
  </header>

  <div class="grid">
    <div class="card">
      <form id="genForm" method="post" action="javascript:void(0)">
        <label>推理引擎</label>
        <select id="engine" name="engine" onchange="onEngineChange()">
          <option value="qwen21">Qwen-Image-2.1-Turbo (千问无限制 · 中文/精准改图)</option>
          <option value="sd">SD 1.5 / SDXL 通用模式 (小内存/纯CPU极速出图)</option>
        </select>

        <div class="row">
          <div>
            <label>主扩散权重</label>
            <select id="model_file" name="model_file"></select>
          </div>
          <div id="llmBox">
            <label>千问文本编码器 (LLM)</label>
            <select id="llm_file" name="llm_file"></select>
          </div>
        </div>

        <label>参考图（自动识别原图尺寸 · 支持直接原图修改）</label>
        <div class="dropzone" onclick="document.getElementById('imageInput').click()">
          <div id="dropText" style="font-size:0.82rem;color:var(--muted)">点击上传图片，或从下方画廊点“作参考图”</div>
          <input type="file" id="imageInput" name="image" accept="image/*" style="display:none" onchange="previewUpload(event)" />
          <input type="hidden" id="ref_output_name" name="ref_output_name" value="" />
          <img id="uploadPreview" />
        </div>
        <button type="button" id="clearImgBtn" style="display:none;margin-top:5px;background:none;border:none;color:var(--muted);cursor:pointer;font-size:0.78rem" onclick="clearUpload()">✕ 清除参考图</button>

        <div id="refSizePanel" style="display:none">
          <div class="row">
            <div>
              <label>参考图尺寸策略</label>
              <select id="size_mode" name="size_mode" onchange="updateSizePlan()">
                <option value="match_exact" selected>严格还原原图尺寸 (1:1同分辨率)</option>
                <option value="match_ratio">仅匹配原图比例 (不拉伸)</option>
                <option value="manual">手动指定下方宽高</option>
              </select>
            </div>
            <div id="maxEdgeBox">
              <label>推理长边上限 (防小机超时)</label>
              <select id="max_edge" name="max_edge" onchange="updateSizePlan()">
                <option value="512">512px (6G小机极速试图)</option>
                <option value="768">768px (小内存推荐)</option>
                <option value="1024">1024px (大内存高清)</option>
                <option value="1536">1536px (超清·较慢)</option>
                <option value="2048">2048px (原图直出·极慢)</option>
              </select>
            </div>
          </div>
          <div class="hint-box" id="sizeHint"></div>
        </div>

        <label>提示词（Prompt · 千问支持直接输入中文改图指令）</label>
        <textarea id="prompt" name="prompt" placeholder="例如：把人物衣服换成黑色风衣，背景改为雨夜霓虹街道，保留原图人脸、表情和姿势..."></textarea>
        <div id="loraWrap" style="display:none;margin-top:6px">
          <div style="font-size:0.75rem;color:var(--muted);margin-bottom:4px">点击插入已安装 LoRA：</div>
          <div class="presets" id="loraChips"></div>
        </div>

        <div id="negBox" style="display:none">
          <label>反向提示词（SD 模式生效）</label>
          <input id="negative_prompt" name="negative_prompt" value="blurry, low quality, bad anatomy, extra fingers, watermark" />
        </div>

        <div id="manualSizeWrap">
          <label>文生图 / 手动分辨率预设</label>
          <div class="presets">
            <button type="button" class="chip" onclick="setSize(512,512)">512×512</button>
            <button type="button" class="chip" onclick="setSize(576,768)">3:4 竖图</button>
            <button type="button" class="chip" onclick="setSize(768,576)">4:3 横图</button>
            <button type="button" class="chip" onclick="setSize(768,768)">768×768</button>
            <button type="button" class="chip" onclick="setSize(1024,1024)">1024×1024</button>
          </div>
          <div class="row">
            <div><label>宽度</label><input type="number" id="width" name="width" value="512" step="32" min="256" max="2048" /></div>
            <div><label>高度</label><input type="number" id="height" name="height" value="512" step="32" min="256" max="2048" /></div>
          </div>
        </div>

        <div class="row3">
          <div><label>步数 (Steps)</label><input type="number" id="steps" name="steps" value="8" min="4" max="60" /></div>
          <div><label>CFG 强度</label><input type="number" id="cfg_scale" name="cfg_scale" value="1.0" step="0.5" min="1.0" max="15.0" /></div>
          <div><label>生成张数</label><input type="number" id="batch_count" name="batch_count" value="1" min="1" max="4" /></div>
        </div>

        <div class="row3">
          <div><label>CPU 线程</label><input type="number" id="threads" name="threads" value="4" min="1" max="64" /></div>
          <div><label>种子 (-1随机)</label><input type="number" id="seed" name="seed" value="-1" /></div>
          <div id="strengthBox" style="display:none"><label>重绘幅度</label><input type="number" id="strength" name="strength" value="0.65" step="0.05" min="0.1" max="1.0" /></div>
        </div>

        <div class="checks">
          <label title="按需磁盘映射权重，6G/10G小内存必备"><input type="checkbox" id="use_mmap" name="use_mmap" value="1" checked /> mmap省内存</label>
          <label title="Flash Attention，降低内存占用并加速推理"><input type="checkbox" id="use_fa" name="use_fa" value="1" checked /> FlashAttn加速</label>
          <label title="分块解码VAE，避免高分辨率出图OOM"><input type="checkbox" id="vae_tiling" name="vae_tiling" value="1" checked /> VAE分块</label>
        </div>

        <button type="submit" class="btn btn-primary" id="submitBtn">加入队列 / 开始生成</button>
        <button type="button" class="btn btn-danger" id="cancelBtn" onclick="cancelJob()">中止当前正在运行的任务</button>
        <div class="err" id="errMsg"></div>
      </form>
    </div>

    <div class="card">
      <div style="display:flex;justify-content:space-between;align-items:center;gap:8px;flex-wrap:wrap">
        <strong id="statusTitle">等待任务</strong>
        <div style="display:flex;gap:8px;align-items:center">
          <button type="button" class="btn-sm" id="useCurrentRefBtn" style="display:none" onclick="useCurrentAsRef()">设为参考图继续改</button>
          <a id="dlLink" href="#" download class="btn-sm" style="display:none;text-decoration:none">⬇ 下载原图</a>
        </div>
      </div>
      <div class="prog-wrap"><div class="prog-bar" id="progBar"></div></div>
      <div class="preview-box">
        <span id="placeholder" style="color:var(--muted)">生成的图片将显示在这里</span>
        <img id="resultImg" />
      </div>

      <div class="queue-box" id="queueBox">
        <div style="font-size:0.82rem;color:var(--accent);margin-bottom:6px">排队中的任务：</div>
        <div id="queueList"></div>
      </div>

      <div class="log-box" id="logBox">就绪。</div>
    </div>
  </div>

  <div class="card" style="margin-top:16px">
    <div style="display:flex;justify-content:space-between;align-items:center">
      <strong>历史图库（支持连续改图 / 参数复用 / 删除）</strong>
      <button type="button" class="btn-sm" onclick="loadGallery()">刷新图库</button>
    </div>
    <div class="gallery" id="gallery"></div>
  </div>
</div>

<script>
let modelInfo = null;
let lastOutputUrl = '';
let currentShownFilename = '';
let galleryCache = [];
let refNaturalW = 0, refNaturalH = 0;

// 右上角进度豆腐块状态
let tileClosedByUser = false;
let lastSeenJobId = '';
let lastStatusSnapshot = null;

function esc(s) {
  return String(s || '').replace(/[&<>"']/g, c => ({'&':'&amp;','<':'&lt;','>':'&gt;','"':'&quot;',"'":'&#39;'}[c]));
}

function align32(v, minV, maxV) {
  v = Math.max(minV, Math.min(maxV, v));
  return Math.max(minV, Math.min(maxV, Math.round(v / 32) * 32));
}

function setSize(w, h) {
  document.getElementById('width').value = w;
  document.getElementById('height').value = h;
}

function insertLora(idx) {
  if (!modelInfo || !modelInfo.loras || !modelInfo.loras[idx]) return;
  const tag = `<lora:${modelInfo.loras[idx]}:0.8>`;
  const ta = document.getElementById('prompt');
  ta.value = ta.value.trim() ? (ta.value.trim() + ', ' + tag) : tag;
  ta.focus();
}

function closeProgressTile() {
  tileClosedByUser = true;
  document.getElementById('progressTile').style.display = 'none';
  if (lastStatusSnapshot && (lastStatusSnapshot.running || lastStatusSnapshot.status !== 'idle')) {
    document.getElementById('reopenTileBtn').style.display = 'inline-block';
  }
}

function reopenProgressTile() {
  tileClosedByUser = false;
  document.getElementById('reopenTileBtn').style.display = 'none';
  if (lastStatusSnapshot) updateProgressTile(lastStatusSnapshot);
}

function showSubmittedTile(promptText) {
  tileClosedByUser = false;
  document.getElementById('reopenTileBtn').style.display = 'none';
  const tile = document.getElementById('progressTile');
  tile.className = 'prog-tile';
  tile.style.display = 'block';
  document.getElementById('ptDot').className = 'pt-dot running';
  document.getElementById('ptTitle').textContent = '已提交队列';
  document.getElementById('ptStep').textContent = '准备启动推理...';
  document.getElementById('ptPct').textContent = '2%';
  document.getElementById('ptBar').style.width = '2%';
  document.getElementById('ptPrompt').textContent = promptText || '';
  document.getElementById('ptSub').textContent = '已耗时 0s';
  document.getElementById('ptStopBtn').style.display = 'none';
}

function updateProgressTile(st) {
  lastStatusSnapshot = st;
  const tile = document.getElementById('progressTile');
  const reopenBtn = document.getElementById('reopenTileBtn');

  if (st.running && st.job_id && st.job_id !== lastSeenJobId) {
    lastSeenJobId = st.job_id;
    tileClosedByUser = false;
  }

  if (!st.running && st.status === 'idle') {
    tile.style.display = 'none';
    reopenBtn.style.display = 'none';
    return;
  }

  if (tileClosedByUser) {
    tile.style.display = 'none';
    reopenBtn.style.display = 'inline-block';
    return;
  }

  reopenBtn.style.display = 'none';
  tile.style.display = 'block';

  const dot = document.getElementById('ptDot');
  const title = document.getElementById('ptTitle');
  const step = document.getElementById('ptStep');
  const pctEl = document.getElementById('ptPct');
  const bar = document.getElementById('ptBar');
  const promptEl = document.getElementById('ptPrompt');
  const sub = document.getElementById('ptSub');
  const stopBtn = document.getElementById('ptStopBtn');

  const qCount = (st.queue && st.queue.length) ? st.queue.length : 0;
  const pct = Math.max(0, Math.min(100, st.progress || 0));

  if (st.running) {
    tile.className = 'prog-tile';
    dot.className = 'pt-dot running';
    title.textContent = `生成中 (${st.engine || ''})`;
    step.textContent = st.step_text || '加载权重中...';
    pctEl.textContent = pct + '%';
    bar.style.width = pct + '%';
    promptEl.textContent = st.prompt || '';
    sub.textContent = `已耗时 ${st.elapsed_sec || 0}s` + (qCount ? ` · 排队 ${qCount}` : '');
    stopBtn.style.display = 'inline-block';
  } else if (st.status === 'done') {
    tile.className = 'prog-tile done';
    dot.className = 'pt-dot done';
    title.textContent = '生成完成';
    step.textContent = st.step_text || '已完成';
    pctEl.textContent = '100%';
    bar.style.width = '100%';
    sub.textContent = `总耗时 ${st.elapsed_sec || 0}s`;
    stopBtn.style.display = 'none';
  } else if (st.status === 'error') {
    tile.className = 'prog-tile error';
    dot.className = 'pt-dot error';
    title.textContent = '生成失败';
    step.textContent = st.error || '发生错误';
    pctEl.textContent = 'ERR';
    bar.style.width = '0%';
    sub.textContent = `耗时 ${st.elapsed_sec || 0}s`;
    stopBtn.style.display = 'none';
  } else if (st.status === 'cancelled') {
    tile.className = 'prog-tile';
    dot.className = 'pt-dot cancelled';
    title.textContent = '任务已中止';
    step.textContent = '已手动停止';
    pctEl.textContent = '0%';
    bar.style.width = '0%';
    sub.textContent = `耗时 ${st.elapsed_sec || 0}s`;
    stopBtn.style.display = 'none';
  }
}

function updateSizePlan() {
  const hasRef = refNaturalW > 0 && refNaturalH > 0;
  document.getElementById('refSizePanel').style.display = hasRef ? 'block' : 'none';
  const mode = document.getElementById('size_mode').value;
  const maxEdge = parseInt(document.getElementById('max_edge').value, 10) || 768;
  const manualWrap = document.getElementById('manualSizeWrap');
  const maxEdgeBox = document.getElementById('maxEdgeBox');
  const hint = document.getElementById('sizeHint');

  if (!hasRef || mode === 'manual') {
    manualWrap.style.display = 'block';
    maxEdgeBox.style.display = 'none';
    hint.style.display = 'none';
    return;
  }

  manualWrap.style.display = 'none';
  maxEdgeBox.style.display = 'block';
  const scale = Math.min(1.0, maxEdge / Math.max(refNaturalW, refNaturalH));
  const gw = align32(Math.round(refNaturalW * scale), 256, maxEdge);
  const gh = align32(Math.round(refNaturalH * scale), 256, maxEdge);
  document.getElementById('width').value = gw;
  document.getElementById('height').value = gh;

  let text = `原图：${refNaturalW}×${refNaturalH} ｜ 推理：${gw}×${gh}`;
  if (mode === 'match_exact') text += ` ｜ 输出精确还原：${refNaturalW}×${refNaturalH}`;
  else text += ` ｜ 输出：${gw}×${gh}`;
  if (gw * gh > 768 * 768 && modelInfo && modelInfo.sys.ram_total_gb <= 12) {
    text += ' （小内存提示：建议先用 512/768px 试图）';
  }
  hint.textContent = text;
  hint.style.display = 'block';
}

async function init() {
  // 自动清理残留的 GET 查询参数
  if (window.location.search) {
    window.history.replaceState({}, document.title, window.location.pathname);
  }
  try {
    const res = await fetch('/api/models');
    modelInfo = await res.json();
    const qOk = modelInfo.qwen.ready_t2i;
    const sOk = modelInfo.sd.ready;
    document.getElementById('modelBadge').textContent = `Qwen2.1: ${qOk ? '已就绪' : '未下载'} | SD: ${sOk ? '已就绪' : '未下载'}`;
    document.getElementById('threads').value = modelInfo.sys.default_threads || 4;
    document.getElementById('threads').max = modelInfo.sys.logical_cores || 16;
    document.getElementById('use_mmap').checked = !!modelInfo.sys.default_mmap;
    document.getElementById('max_edge').value = String(modelInfo.sys.default_max_edge || 768);
    if (modelInfo.loras && modelInfo.loras.length) {
      document.getElementById('loraWrap').style.display = 'block';
      document.getElementById('loraChips').innerHTML = modelInfo.loras
        .map((name, idx) => `<button type="button" class="chip" onclick="insertLora(${idx})">+ ${esc(name)}</button>`)
        .join('');
    }
    if (!qOk && sOk) document.getElementById('engine').value = 'sd';
    onEngineChange();
  } catch (err) {
    document.getElementById('errMsg').textContent = '初始化模型信息失败: ' + err;
  }
  loadGallery();
  pollStatus();
  setInterval(pollStatus, 2000);
}

function onEngineChange() {
  if (!modelInfo) return;
  const eng = document.getElementById('engine').value;
  const sel = document.getElementById('model_file');
  const llmSel = document.getElementById('llm_file');
  sel.innerHTML = '';
  llmSel.innerHTML = '';
  if (eng === 'qwen21') {
    const dList = modelInfo.qwen.dit.length ? modelInfo.qwen.dit : ['(未下载，执行脚本选项2)'];
    dList.forEach(f => { sel.innerHTML += `<option value="${esc(f)}">${esc(f)}</option>`; });
    const lList = modelInfo.qwen.llm.length ? modelInfo.qwen.llm : ['(未下载)'];
    lList.forEach(f => { llmSel.innerHTML += `<option value="${esc(f)}">${esc(f)}</option>`; });
    document.getElementById('llmBox').style.display = 'block';
    document.getElementById('steps').value = 8;
    document.getElementById('cfg_scale').value = 1.0;
    document.getElementById('negBox').style.display = 'none';
    document.getElementById('strengthBox').style.display = 'none';
  } else {
    const sList = modelInfo.sd.checkpoints.length ? modelInfo.sd.checkpoints : ['(未下载，执行脚本选项3)'];
    sList.forEach(f => { sel.innerHTML += `<option value="${esc(f)}">${esc(f)}</option>`; });
    document.getElementById('llmBox').style.display = 'none';
    document.getElementById('steps').value = 15;
    document.getElementById('cfg_scale').value = 7.0;
    document.getElementById('negBox').style.display = 'block';
    document.getElementById('strengthBox').style.display = 'block';
  }
}

function attachPreviewImg(src, labelText) {
  const img = document.getElementById('uploadPreview');
  img.onload = () => {
    refNaturalW = img.naturalWidth;
    refNaturalH = img.naturalHeight;
    document.getElementById('dropText').textContent = `${labelText} (${refNaturalW}×${refNaturalH})`;
    updateSizePlan();
  };
  img.src = src;
  img.style.display = 'inline-block';
  document.getElementById('clearImgBtn').style.display = 'inline-block';
}

function previewUpload(e) {
  const file = e.target.files[0];
  if (!file) return;
  document.getElementById('ref_output_name').value = '';
  attachPreviewImg(URL.createObjectURL(file), '已上传: ' + file.name);
}

function setGalleryAsRef(filename, url) {
  document.getElementById('imageInput').value = '';
  document.getElementById('ref_output_name').value = filename;
  attachPreviewImg(url, '已引用图库: ' + filename);
  window.scrollTo({ top: 0, behavior: 'smooth' });
}

function useCurrentAsRef() {
  if (currentShownFilename) setGalleryAsRef(currentShownFilename, '/outputs/' + currentShownFilename);
}

function clearUpload() {
  document.getElementById('imageInput').value = '';
  document.getElementById('ref_output_name').value = '';
  document.getElementById('uploadPreview').style.display = 'none';
  document.getElementById('dropText').textContent = '点击上传图片，或从下方画廊点“作参考图”';
  document.getElementById('clearImgBtn').style.display = 'none';
  refNaturalW = 0;
  refNaturalH = 0;
  updateSizePlan();
}

document.getElementById('genForm').addEventListener('submit', async (e) => {
  e.preventDefault();
  document.getElementById('errMsg').textContent = '';
  const promptVal = document.getElementById('prompt').value.trim();
  if (!promptVal) {
    document.getElementById('errMsg').textContent = '请输入提示词';
    return;
  }
  const fd = new FormData(e.target);
  if (!document.getElementById('use_mmap').checked) fd.set('use_mmap', '0');
  if (!document.getElementById('use_fa').checked) fd.set('use_fa', '0');
  if (!document.getElementById('vae_tiling').checked) fd.set('vae_tiling', '0');
  if (!(refNaturalW > 0 && refNaturalH > 0)) fd.set('size_mode', 'manual');

  const btn = document.getElementById('submitBtn');
  btn.disabled = true;
  btn.textContent = '提交中...';
  try {
    const res = await fetch('/api/generate', { method: 'POST', body: fd });
    const data = await res.json();
    if (!res.ok) {
      document.getElementById('errMsg').textContent = data.detail || '提交失败';
      return;
    }
    showSubmittedTile(promptVal);
    pollStatus();
  } catch (err) {
    document.getElementById('errMsg').textContent = '请求失败: ' + err;
  } finally {
    btn.disabled = false;
    btn.textContent = '加入队列 / 开始生成';
  }
});

async function cancelJob() {
  await fetch('/api/cancel', { method: 'POST' });
  pollStatus();
}

async function removeQueued(id) {
  await fetch('/api/queue/' + encodeURIComponent(id), { method: 'DELETE' });
  pollStatus();
}

async function pollStatus() {
  try {
    const res = await fetch('/api/status');
    if (!res.ok) return;
    const st = await res.json();
    if (st.sys) {
      const s = st.sys;
      document.getElementById('sysBadge').textContent =
        `${s.arch} | RAM ${s.ram_used_gb}/${s.ram_total_gb}G | Swap ${s.swap_used_gb}/${s.swap_total_gb}G | Load ${s.load1}`;
    }
    const logBox = document.getElementById('logBox');
    if (st.logs && st.logs.length) {
      const atBottom = logBox.scrollHeight - logBox.scrollTop - logBox.clientHeight < 40;
      logBox.textContent = st.logs.join('\n');
      if (atBottom) logBox.scrollTop = logBox.scrollHeight;
    }
    document.getElementById('progBar').style.width = (st.progress || 0) + '%';
    const cancelBtn = document.getElementById('cancelBtn');
    const title = document.getElementById('statusTitle');

    if (st.running) {
      cancelBtn.style.display = 'block';
      title.textContent = `推理中 (${st.engine}) · ${st.step_text} · 已耗时 ${st.elapsed_sec}s`;
    } else {
      cancelBtn.style.display = 'none';
      if (st.status === 'done') title.textContent = `✅ ${st.step_text || '生成完成'}`;
      else if (st.status === 'error') {
        title.textContent = '❌ 生成失败';
        document.getElementById('errMsg').textContent = st.error || '';
      } else if (st.status === 'cancelled') title.textContent = '⏹ 任务已中止';
      else title.textContent = '等待任务';
    }

    updateProgressTile(st);

    if (st.output_url && st.output_url !== lastOutputUrl) {
      lastOutputUrl = st.output_url;
      showResult(st.output_url);
      loadGallery();
    }

    const qBox = document.getElementById('queueBox');
    const qList = document.getElementById('queueList');
    if (st.queue && st.queue.length) {
      qBox.style.display = 'block';
      qList.innerHTML = st.queue.map(q =>
        `<div class="q-item"><span>[${esc(q.engine)} · ${esc(q.size)}] ${esc(q.prompt)}</span>` +
        `<button class="btn-sm" onclick="removeQueued('${esc(q.id)}')">移出</button></div>`
      ).join('');
    } else {
      qBox.style.display = 'none';
    }
  } catch (_) {}
}

function showResult(url) {
  const img = document.getElementById('resultImg');
  img.src = url + '?t=' + Date.now();
  img.style.display = 'block';
  document.getElementById('placeholder').style.display = 'none';
  const dl = document.getElementById('dlLink');
  dl.href = url;
  dl.style.display = 'inline-block';
  currentShownFilename = url.split('/').pop();
  document.getElementById('useCurrentRefBtn').style.display = 'inline-block';
}

function reuseMeta(idx) {
  const item = galleryCache[idx];
  if (!item || !item.meta) return;
  const m = item.meta;
  if (m.engine) {
    document.getElementById('engine').value = m.engine;
    onEngineChange();
  }
  if (m.model_file) document.getElementById('model_file').value = m.model_file;
  if (m.llm_file) document.getElementById('llm_file').value = m.llm_file;
  if (m.prompt) document.getElementById('prompt').value = m.prompt;
  if (m.negative_prompt) document.getElementById('negative_prompt').value = m.negative_prompt;
  if (m.width) document.getElementById('width').value = m.width;
  if (m.height) document.getElementById('height').value = m.height;
  if (m.steps) document.getElementById('steps').value = m.steps;
  if (m.cfg_scale) document.getElementById('cfg_scale').value = m.cfg_scale;
  if (m.seed !== undefined) document.getElementById('seed').value = m.seed;
  window.scrollTo({ top: 0, behavior: 'smooth' });
}

async function deleteImg(filename) {
  if (!confirm('确认删除图片 ' + filename + ' ?')) return;
  await fetch('/api/gallery/' + encodeURIComponent(filename), { method: 'DELETE' });
  loadGallery();
}

async function loadGallery() {
  try {
    const res = await fetch('/api/gallery');
    if (!res.ok) return;
    const data = await res.json();
    galleryCache = data.items || [];
    const g = document.getElementById('gallery');
    g.innerHTML = galleryCache.map((item, idx) => {
      const m = item.meta || {};
      const info = `${m.engine || ''} ${m.width || ''}×${m.height || ''} ${m.elapsed_sec ? m.elapsed_sec + 's' : ''}`;
      return `<div class="g-item">
        <img src="${esc(item.url)}" loading="lazy" onclick="showResult('${esc(item.url)}')" />
        <div class="g-meta">
          <div class="g-cap" title="${esc(m.prompt || item.filename)}">${esc(m.prompt || item.filename)}</div>
          <div style="margin-bottom:6px;font-size:0.7rem">${esc(info)}</div>
          <div class="g-ops">
            <button class="btn-sm" onclick="setGalleryAsRef('${esc(item.filename)}','${esc(item.url)}')">作参考图</button>
            <button class="btn-sm" onclick="reuseMeta(${idx})">复用</button>
            <button class="btn-sm" style="color:#fca5a5" onclick="deleteImg('${esc(item.filename)}')">删</button>
          </div>
        </div>
      </div>`;
    }).join('');
  } catch (_) {}
}

init();
</script>
</body>
</html>"""
PY_EOF
}

do_cmake_build() {
    local mode="$1"
    local ram_gb log_c build_jobs cmake_bin="cmake"
    command -v cmake3 >/dev/null 2>&1 && ! command -v cmake >/dev/null 2>&1 && cmake_bin="cmake3"

    ram_gb=$(get_ram_gb)
    log_c=$(nproc 2>/dev/null || echo 2)

    # 小内存防编译 OOM 限核保护
    if [[ "$ram_gb" -le 6 ]]; then
        build_jobs=$(( log_c > 2 ? 2 : log_c ))
    elif [[ "$ram_gb" -le 10 ]]; then
        build_jobs=$(( log_c > 3 ? 3 : log_c ))
    else
        build_jobs="$log_c"
    fi

    local native_flag="-DGGML_NATIVE=ON"
    [[ "$mode" == "portable" ]] && native_flag="-DGGML_NATIVE=OFF"

    log_info "开始编译 stable-diffusion.cpp (模式: ${mode}, 架构: $(get_arch), 并行线程: ${build_jobs})..."
    rm -rf "${APP_DIR}/src/stable-diffusion.cpp/build"
    "$cmake_bin" -S "${APP_DIR}/src/stable-diffusion.cpp" \
                 -B "${APP_DIR}/src/stable-diffusion.cpp/build" \
                 -DCMAKE_BUILD_TYPE=Release \
                 "$native_flag"
    "$cmake_bin" --build "${APP_DIR}/src/stable-diffusion.cpp/build" --config Release -j"${build_jobs}"
}

compile_engine() {
    mkdir -p "${APP_DIR}"/{src,bin,models/qwen21/{dit,llm,mmproj,vae},models/sd,models/loras,uploads,outputs}

    if [[ ! -d "${APP_DIR}/src/stable-diffusion.cpp" ]]; then
        git clone --recursive https://github.com/leejet/stable-diffusion.cpp.git "${APP_DIR}/src/stable-diffusion.cpp"
    else
        git -C "${APP_DIR}/src/stable-diffusion.cpp" pull --ff-only || true
        git -C "${APP_DIR}/src/stable-diffusion.cpp" submodule update --init --recursive
    fi

    do_cmake_build "$BUILD_MODE"

    local built_bin
    built_bin=$(find "${APP_DIR}/src/stable-diffusion.cpp/build" -maxdepth 3 -type f -name "sd-cli" -executable | head -n1 || true)
    if [[ -z "$built_bin" ]]; then
        log_err "编译完成但未找到 sd-cli 可执行文件！"
        exit 1
    fi

    # 自动自检：防止 Netcup/KVM 虚拟化或特殊 ARM 节点触发 SIGILL 非法指令
    if ! "$built_bin" --help >/dev/null 2>&1; then
        if [[ "$BUILD_MODE" == "native" ]]; then
            log_warn "检测到 Native 编译产物在当前虚拟化 CPU 上触发指令异常，自动切换 Portable 通用模式重编..."
            BUILD_MODE="portable"
            save_conf
            do_cmake_build "portable"
            built_bin=$(find "${APP_DIR}/src/stable-diffusion.cpp/build" -maxdepth 3 -type f -name "sd-cli" -executable | head -n1 || true)
        fi
    fi

    cp -f "$built_bin" "${APP_DIR}/bin/sd-cli"
    chmod +x "${APP_DIR}/bin/sd-cli"
    log_ok "sd-cli 编译并自检通过 (${APP_DIR}/bin/sd-cli, 模式: ${BUILD_MODE})"
}

write_systemd_service() {
    local log_c quota_line=""
    log_c=$(nproc 2>/dev/null || echo 4)
    if [[ "${CPU_QUOTA_PCT:-0}" -ge 50 && "${CPU_QUOTA_PCT:-0}" -le 95 ]]; then
        local total_quota=$(( log_c * CPU_QUOTA_PCT ))
        quota_line="CPUQuota=${total_quota}%"
    fi

    cat <<EOF > "$SERVICE_FILE"
[Unit]
Description=Universal Qwen-Image & SD WebUI (stable-diffusion.cpp)
After=network.target

[Service]
Type=simple
WorkingDirectory=${APP_DIR}
EnvironmentFile=${APP_DIR}/.env
ExecStart=${APP_DIR}/venv/bin/uvicorn app:app --host ${WEB_HOST} --port ${WEB_PORT} --workers 1
Nice=10
IOSchedulingClass=idle
${quota_line}
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
    systemctl daemon-reload
    systemctl enable --now "$SERVICE_NAME"
    systemctl restart "$SERVICE_NAME"
}

open_firewall_port() {
    [[ "$WEB_HOST" != "0.0.0.0" ]] && return
    if command -v ufw >/dev/null 2>&1 && ufw status 2>/dev/null | grep -q "Status: active"; then
        ufw allow "${WEB_PORT}/tcp" >/dev/null || true
        log_ok "已自动放行 UFW 端口 ${WEB_PORT}/tcp"
    elif command -v firewall-cmd >/dev/null 2>&1 && systemctl is-active --quiet firewalld; then
        firewall-cmd --permanent --add-port="${WEB_PORT}/tcp" >/dev/null || true
        firewall-cmd --reload >/dev/null || true
        log_ok "已自动放行 firewalld 端口 ${WEB_PORT}/tcp"
    fi
}

close_firewall_port() {
    if command -v ufw >/dev/null 2>&1 && ufw status 2>/dev/null | grep -q "${WEB_PORT}/tcp"; then
        ufw delete allow "${WEB_PORT}/tcp" >/dev/null 2>&1 || true
        log_ok "已回收 UFW 端口 ${WEB_PORT}/tcp"
    elif command -v firewall-cmd >/dev/null 2>&1 && systemctl is-active --quiet firewalld; then
        firewall-cmd --permanent --remove-port="${WEB_PORT}/tcp" >/dev/null 2>&1 || true
        firewall-cmd --reload >/dev/null 2>&1 || true
        log_ok "已回收 firewalld 端口 ${WEB_PORT}/tcp"
    fi
}

install_all() {
    require_root
    load_conf

    echo
    log_info "硬件体检: 架构=$(get_arch) | 指令集=$(get_cpu_features) | CPU=$(nproc)核 | 内存=$(get_ram_gb)GB | 虚拟化=$(get_virt)"
    choose_install_dir
    setup_swap_smart

    log_info "1/5 安装系统依赖..."
    install_system_deps

    log_info "2/5 编译 stable-diffusion.cpp 推理引擎..."
    compile_engine

    log_info "3/5 创建零 PyTorch 极简 Python 虚拟环境..."
    python3 -m venv "${APP_DIR}/venv"
    "${APP_DIR}/venv/bin/pip" install --upgrade pip -q
    "${APP_DIR}/venv/bin/pip" install -q fastapi "uvicorn[standard]" python-multipart pillow

    log_info "4/5 写入 WebUI 并配置网络鉴权与 CPU 调度..."
    write_webui_app
    setup_auth_and_network
    write_systemd_service

    log_info "5/5 检查防火墙端口..."
    open_firewall_port

    echo
    log_ok "全平台环境部署完成！接下来请选择菜单 [2] 或 [3] 下载无审查模型权重。"
    show_status
}

update_engine() {
    require_root
    load_conf
    [[ ! -d "$APP_DIR" ]] && { log_err "未检测到安装目录 ${APP_DIR}，请先执行安装。"; return; }
    echo "选择编译模式（若发生热迁移后报错 SIGILL 请选 2）："
    echo "  1) Native   本机极致优化模式 (默认)"
    echo "  2) Portable 跨节点兼容模式 (适合频繁迁移的 KVM VPS)"
    read -rp "请选择 [1-2, 默认 1]: " bm
    [[ "${bm:-1}" == "2" ]] && BUILD_MODE="portable" || BUILD_MODE="native"
    save_conf

    systemctl stop "$SERVICE_NAME" 2>/dev/null || true
    compile_engine
    write_webui_app
    systemctl start "$SERVICE_NAME"
    log_ok "引擎与 WebUI 更新完成，模型与历史图库已完整保留。"
}

pick_hf_mirror() {
    echo "请选择 HuggingFace 下载源："
    echo "  1) 官方直连 https://huggingface.co (Netcup/OVH/海外服务器首选，默认)"
    echo "  2) 国内镜像 https://hf-mirror.com (国内自建机/家宽选此项)"
    read -rp "请输入 [1-2, 默认 1]: " m
    if [[ "${m:-1}" == "2" ]]; then
        HF_BASE="https://hf-mirror.com"
    else
        HF_BASE="https://huggingface.co"
    fi
}

download_file() {
    local url="$1"
    local dst="$2"
    log_info "正在下载: $(basename "$dst")"
    curl -fL -C - --retry 3 --progress-bar -o "$dst" "$url"
}

hf_find_uncensored_qwen_llm() {
    local hf_base="$1"
    local tier="$2"
    python3 - "$hf_base" "$tier" <<'PY_FIND'
import json
import sys
import urllib.parse
import urllib.request

hf_base = sys.argv[1].rstrip("/")
tier = sys.argv[2]

quant_prefs = {
    "1": ["q2_k_xl", "q2_k_l", "q2_k", "iq2_m", "iq2_xs", "q3_k_s", "q3_k_m"],
    "2": ["q3_k_xl", "q3_k_m", "q3_k_l", "q3_k_s", "q2_k_xl", "q2_k", "q4_k_s", "q4_k_m"],
    "3": ["q4_k_xl", "q4_k_m", "q4_k_s", "q4_0", "q3_k_l", "q5_k_m"],
    "4": ["q8_0", "q6_k", "q5_k_m", "q4_k_xl", "q4_k_m"],
}.get(tier, ["q4_k_xl", "q4_k_m", "q4_k_s", "q3_k_m"])

queries = [
    "Qwen3-VL-8B-Instruct abliterated GGUF",
    "Qwen3-VL-8B abliterated GGUF",
    "Qwen3-VL-8B-Instruct uncensored GGUF",
    "Qwen3-VL-8B heretic GGUF",
]

headers = {"User-Agent": "qwen-sd-webui/1.0"}


def fetch_json(url):
    req = urllib.request.Request(url, headers=headers)
    with urllib.request.urlopen(req, timeout=15) as resp:
        return json.loads(resp.read().decode("utf-8"))


seen = set()
candidates = []
for q in queries:
    try:
        url = f"{hf_base}/api/models?search={urllib.parse.quote(q)}&sort=downloads&direction=-1&limit=20"
        items = fetch_json(url)
    except Exception:
        continue
    for item in items:
        rid = item.get("id") or item.get("modelId") or ""
        if not rid or rid in seen:
            continue
        low = rid.lower()
        if (
            "qwen3" in low and "vl" in low and "8b" in low and "gguf" in low
            and any(k in low for k in ("abliterat", "uncensor", "heretic", "unrestrict", "obliterat", "huihui"))
            and "thinking" not in low
        ):
            seen.add(rid)
            candidates.append(rid)

for rid in candidates[:8]:
    try:
        info = fetch_json(f"{hf_base}/api/models/{rid}")
    except Exception:
        continue
    files = [s.get("rfilename", "") for s in info.get("siblings", [])]
    ggufs = [
        f for f in files
        if f.lower().endswith(".gguf") and "mmproj" not in f.lower() and "-00001-of-" not in f.lower()
    ]
    if not ggufs:
        continue

    chosen = ""
    for pref in quant_prefs:
        matched = [f for f in ggufs if pref in f.lower()]
        if matched:
            chosen = sorted(matched, key=len)[0]
            break
    if not chosen:
        chosen = sorted(ggufs, key=len)[0]

    mmprojs = [f for f in files if f.lower().endswith(".gguf") and "mmproj" in f.lower()]
    chosen_mm = ""
    for mpref in ("f16", "bf16", "q8_0", "f32"):
        mm_m = [f for f in mmprojs if mpref in f.lower()]
        if mm_m:
            chosen_mm = sorted(mm_m, key=len)[0]
            break
    if not chosen_mm and mmprojs:
        chosen_mm = mmprojs[0]

    print(f"{rid}|{chosen}|{chosen_mm}")
    sys.exit(0)

sys.exit(1)
PY_FIND
}

hf_resolve_repo_file() {
    local hf_base="$1"
    local repo_id="$2"
    local prefs="$3"
    python3 - "$hf_base" "$repo_id" "$prefs" <<'PY_RES'
import json
import sys
import urllib.request

hf_base = sys.argv[1].rstrip("/")
repo_id = sys.argv[2]
prefs = [p.strip() for p in sys.argv[3].split(",") if p.strip()]
headers = {"User-Agent": "qwen-sd-webui/1.0"}

try:
    req = urllib.request.Request(f"{hf_base}/api/models/{repo_id}", headers=headers)
    with urllib.request.urlopen(req, timeout=15) as resp:
        info = json.loads(resp.read().decode("utf-8"))
    files = [s.get("rfilename", "") for s in info.get("siblings", [])]
    for p in prefs:
        if p in files:
            print(p)
            sys.exit(0)
    cand = [f for f in files if f.lower().endswith((".safetensors", ".gguf"))]
    for p in prefs:
        pl = p.lower()
        for f in cand:
            if pl in f.lower():
                print(f)
                sys.exit(0)
    if cand:
        print(sorted(cand, key=len)[0])
        sys.exit(0)
except Exception:
    pass

if prefs:
    print(prefs[0])
    sys.exit(0)
sys.exit(1)
PY_RES
}

cleanup_old_censored_qwen_llm() {
    local old_llms
    old_llms=$(find "${APP_DIR}/models/qwen21" -maxdepth 1 -type f -iname "*Qwen3-VL*.gguf" \
        ! -iname "*abliterat*" ! -iname "*uncensor*" ! -iname "*heretic*" ! -iname "*unrestrict*" ! -iname "*obliterat*" 2>/dev/null || true)
    if [[ -n "$old_llms" ]]; then
        echo
        log_warn "检测到旧的官方审查版千问编码器（会导致正常换装误拒）："
        echo "$old_llms"
        read -rp "是否删除旧官方审查版，强制默认使用新的无审查编码器? [Y/n]: " rm_old
        if [[ "${rm_old:-Y}" =~ ^[Yy]$ ]]; then
            while IFS= read -r f; do
                [[ -n "$f" ]] && rm -f "$f"
            done <<< "$old_llms"
            log_ok "已清理旧官方审查版编码器。"
        fi
    fi
}

download_qwen21() {
    require_root
    load_conf
    mkdir -p "${APP_DIR}/models/qwen21"/{dit,llm,mmproj,vae}
    pick_hf_mirror

    local ram_gb def_opt=2
    ram_gb=$(get_ram_gb)
    if [[ "$ram_gb" -le 7 ]]; then def_opt=1
    elif [[ "$ram_gb" -le 14 ]]; then def_opt=2
    elif [[ "$ram_gb" -le 28 ]]; then def_opt=3
    else def_opt=3
    fi

    echo
    log_info "当前物理内存 ${ram_gb}GB，请选择 Qwen-Image-2.1-Turbo 无审查权重套装（自动推荐选项 ${def_opt}）："
    echo "  1) 6GB 极限省内存套装   : DiT Q2_K (2.47G) + 无审查版 LLM Q2_K + 视觉/VAE [总计约 6.9GB]"
    echo "  2) 8-12GB 小内存平衡套装: DiT Q3_K_XL (3.61G) + 无审查版 LLM Q3_K + 视觉/VAE [总计约 8.3GB]"
    echo "  3) 16-32GB 均衡推荐套装 : DiT Q4_K_M (4.20G) + 无审查版 LLM Q4_K_M + 视觉/VAE [总计约 10.3GB]"
    echo "  4) 32GB+ 高画质独服套装 : DiT Q8_0 (7.50G) + 无审查版 LLM Q8_0 + 视觉/VAE [总计约 16.5GB]"
    read -rp "请输入 [1-4, 默认 ${def_opt}]: " qver
    qver="${qver:-$def_opt}"

    local dit_file
    case "$qver" in
        1) dit_file="qwen-image-2.1-turbo-Q2_K.gguf" ;;
        2) dit_file="qwen-image-2.1-turbo-Q3_K_XL.gguf" ;;
        4) dit_file="qwen-image-2.1-turbo-Q8_0.gguf" ;;
        *) dit_file="qwen-image-2.1-turbo-Q4_K_M.gguf" ;;
    esac

    log_info "正在从 ${HF_BASE} 自动检索社区去审查版 (Abliterated/Uncensored) Qwen3-VL-8B-Instruct GGUF..."
    local resolved llm_repo llm_file mm_file llm_url mm_url
    resolved=$(hf_find_uncensored_qwen_llm "$HF_BASE" "$qver" || true)

    if [[ -n "$resolved" ]]; then
        IFS='|' read -r llm_repo llm_file mm_file <<< "$resolved"
        log_ok "已匹配无审查编码器仓库: ${llm_repo}"
        log_ok "已匹配无审查编码器文件: ${llm_file}"
        llm_url="${HF_BASE}/${llm_repo}/resolve/main/${llm_file}"
        if [[ -n "${mm_file:-}" ]]; then
            mm_url="${HF_BASE}/${llm_repo}/resolve/main/${mm_file}"
        else
            mm_file="mmproj-F16.gguf"
            mm_url="${HF_BASE}/unsloth/Qwen3-VL-8B-Instruct-GGUF/resolve/main/mmproj-F16.gguf"
        fi
    else
        log_warn "自动检索社区去审查版 Qwen3-VL-8B-Instruct GGUF 未命中或网络超时。"
        echo "请前往 ${HF_BASE} 搜索 'Qwen3-VL-8B-Instruct abliterated GGUF' 并粘贴直链："
        read -rp "请输入无审查版 LLM GGUF 直链 URL: " llm_url
        [[ -z "$llm_url" ]] && { log_err "未提供无审查编码器直链，已取消下载（不会回退到官方审查版）。"; return; }
        llm_file=$(basename "${llm_url%%\?*}")
        mm_file="mmproj-F16.gguf"
        mm_url="${HF_BASE}/unsloth/Qwen3-VL-8B-Instruct-GGUF/resolve/main/mmproj-F16.gguf"
    fi

    log_info "开始下载 Qwen-Image-2.1-Turbo 无审查完整套件 (${dit_file} + $(basename "$llm_file") + mmproj + VAE)..."
    download_file "${HF_BASE}/unsloth/Qwen-Image-2.1-Turbo-GGUF/resolve/main/${dit_file}" \
                  "${APP_DIR}/models/qwen21/dit/${dit_file}"
    download_file "$llm_url" \
                  "${APP_DIR}/models/qwen21/llm/$(basename "$llm_file")"
    download_file "$mm_url" \
                  "${APP_DIR}/models/qwen21/mmproj/$(basename "$mm_file")"
    download_file "${HF_BASE}/unsloth/Qwen-Image-2.1-FP8/resolve/main/vae/qwen_image_2.1_vae_bf16.safetensors" \
                  "${APP_DIR}/models/qwen21/vae/qwen_image_2.1_vae_bf16.safetensors"

    cleanup_old_censored_qwen_llm
    log_ok "千问无审查套件下载完成！刷新网页即可开始无限制文生图与原图换装。"
}

download_sd() {
    require_root
    load_conf
    mkdir -p "${APP_DIR}/models/sd"
    pick_hf_mirror

    echo
    log_info "请选择要下载的无审查 SD 底模："
    echo "  1) Realistic Vision V5.1 (noVAE)  [SD 1.5 写实无审查 · ~2.13GB · 6G小机/换装推荐]"
    echo "  2) DreamShaper 8                  [SD 1.5 通用无审查 · ~2.13GB · 6G小机流畅]"
    echo "  3) RealVisXL V4.0                 [SDXL 写实无审查   · ~6.94GB · 16G+内存高清]"
    echo "  4) SD 1.5 基础无限制底模          [v1-5-pruned-emaonly · ~4.27GB · 经典基础]"
    read -rp "请输入 [1-4, 默认 1]: " sd_choice

    local repo_id prefs resolved_file
    case "${sd_choice:-1}" in
        2)
            repo_id="Lykon/DreamShaper"
            prefs="DreamShaper_8_pruned.safetensors,DreamShaper_8_fp16.safetensors,dreamshaper"
            ;;
        3)
            repo_id="SG161222/RealVisXL_V4.0"
            prefs="RealVisXL_V4.0.safetensors,RealVisXL_V4.0_Lightning.safetensors,realvisxl"
            ;;
        4)
            repo_id="stable-diffusion-v1-5/stable-diffusion-v1-5"
            prefs="v1-5-pruned-emaonly.safetensors"
            ;;
        *)
            repo_id="SG161222/Realistic_Vision_V5.1_noVAE"
            prefs="Realistic_Vision_V5.1_fp16-no-ema.safetensors,Realistic_Vision_V5.1.safetensors,realistic_vision"
            ;;
    esac

    resolved_file=$(hf_resolve_repo_file "$HF_BASE" "$repo_id" "$prefs" || true)
    [[ -z "$resolved_file" ]] && { log_err "未能解析仓库 ${repo_id} 中的权重文件，请使用菜单 [4] 手动传入直链。"; return; }

    log_info "正在下载无审查底模: ${repo_id}/${resolved_file} ..."
    download_file "${HF_BASE}/${repo_id}/resolve/main/${resolved_file}" \
                  "${APP_DIR}/models/sd/$(basename "$resolved_file")"
    log_ok "下载完成！也可使用菜单选项 [4] 下载任何第三方无限制 .safetensors / .gguf 权重与 LoRA。"
}

download_custom_url() {
    require_root
    load_conf
    echo "选择保存目录："
    echo "  1) 千问无审查文本/视觉编码器 LLM (${APP_DIR}/models/qwen21/llm) [解决误拒核心]"
    echo "  2) 千问无限制主扩散模型 DiT       (${APP_DIR}/models/qwen21/dit)"
    echo "  3) SD 1.5 / SDXL 无限制底模       (${APP_DIR}/models/sd)"
    echo "  4) LoRA 权重目录                  (${APP_DIR}/models/loras)"
    read -rp "请输入 [1-4, 默认 1]: " dchoice
    local target_dir="${APP_DIR}/models/qwen21/llm"
    case "${dchoice:-1}" in
        1) target_dir="${APP_DIR}/models/qwen21/llm" ;;
        2) target_dir="${APP_DIR}/models/qwen21/dit" ;;
        3) target_dir="${APP_DIR}/models/sd" ;;
        4) target_dir="${APP_DIR}/models/loras" ;;
        *) target_dir="${APP_DIR}/models/qwen21/llm" ;;
    esac
    mkdir -p "$target_dir"

    read -rp "请输入模型直链 URL (支持 HF / Civitai): " url
    [[ -z "$url" ]] && { log_err "URL 不能为空"; return; }
    local def_fname
    def_fname=$(basename "${url%%\?*}")
    read -rp "保存文件名 [默认: ${def_fname}]: " fname
    fname="${fname:-$def_fname}"
    [[ -z "$fname" ]] && { log_err "文件名不能为空"; return; }

    download_file "$url" "${target_dir}/${fname}"
    log_ok "模型已保存到 ${target_dir}/${fname}，刷新网页即可切换。"

    if [[ "${dchoice:-1}" == "1" ]]; then
        cleanup_old_censored_qwen_llm
    fi
}

reconfigure_auth() {
    require_root
    load_conf
    [[ ! -d "$APP_DIR" ]] && { log_err "请先执行安装"; return; }
    setup_auth_and_network
    write_systemd_service
    open_firewall_port
    show_status
}

show_status() {
    load_conf
    local arch feat rec_t log_c virt lan_ip pub_v4 pub_v6
    arch=$(get_arch)
    feat=$(get_cpu_features)
    rec_t=$(get_recommended_threads)
    log_c=$(nproc 2>/dev/null || echo 4)
    virt=$(get_virt)
    lan_ip=$(hostname -I 2>/dev/null | awk '{print $1}' || echo "127.0.0.1")
    pub_v4=$(curl -s4 --max-time 3 https://api.ipify.org 2>/dev/null || true)
    pub_v6=$(curl -s6 --max-time 3 https://api6.ipify.org 2>/dev/null || true)

    echo "============================================================"
    systemctl is-active --quiet "$SERVICE_NAME" \
        && log_ok "服务状态: 运行中 (${SERVICE_NAME})" \
        || log_warn "服务状态: 未运行"
    echo "  硬件环境: ${arch} (${feat}) | ${log_c}核 (推荐 ${rec_t} 线程) | 虚拟化: ${virt}"
    echo "  编译模式: ${BUILD_MODE} | CPU上限: $([[ "${CPU_QUOTA_PCT:-0}" -gt 0 ]] && echo "${CPU_QUOTA_PCT}%/核" || echo "不限制")"
    echo "  安装目录: ${APP_DIR}"
    if [[ "$WEB_HOST" == "127.0.0.1" ]]; then
        echo -e "  访问模式: 仅本机 (${GREEN}http://127.0.0.1:${WEB_PORT}${NC})"
        echo -e "  SSH 隧道: ${CYAN}ssh -L ${WEB_PORT}:127.0.0.1:${WEB_PORT} root@${pub_v4:-$lan_ip}${NC}"
    else
        echo -e "  局域网访问: ${GREEN}http://${lan_ip}:${WEB_PORT}${NC}"
        [[ -n "$pub_v4" && "$pub_v4" != "$lan_ip" ]] && echo -e "  公网 IPv4 : ${GREEN}http://${pub_v4}:${WEB_PORT}${NC}"
        [[ -n "$pub_v6" ]] && echo -e "  公网 IPv6 : ${GREEN}http://[${pub_v6}]:${WEB_PORT}${NC}"
    fi
    if [[ -f "${APP_DIR}/.env" ]]; then
        local a_en a_u
        a_en=$(grep '^AUTH_ENABLED=' "${APP_DIR}/.env" | cut -d= -f2- || true)
        a_u=$(grep '^AUTH_USER=' "${APP_DIR}/.env" | cut -d= -f2- || true)
        [[ "$a_en" == "1" ]] && echo "  网页鉴权: 已开启 (用户名: ${a_u} · 密码见 ${APP_DIR}/.env)" || echo "  网页鉴权: 已关闭"
    fi
    echo
    free -h
    echo
    df -h "${APP_DIR}" 2>/dev/null || df -h /
    echo "============================================================"
}

uninstall_all() {
    require_root
    load_conf
    log_warn "即将彻底卸载 ${SERVICE_NAME} 并删除目录 ${APP_DIR}！"
    read -rp "确认继续卸载? [y/N]: " confirm
    [[ ! "${confirm:-N}" =~ ^[Yy]$ ]] && { log_info "已取消卸载。"; return; }

    if systemctl list-unit-files 2>/dev/null | grep -q "^${SERVICE_NAME}.service"; then
        systemctl stop "$SERVICE_NAME" 2>/dev/null || true
        systemctl disable "$SERVICE_NAME" 2>/dev/null || true
        rm -f "$SERVICE_FILE"
        systemctl daemon-reload
        log_ok "已移除 systemd 服务。"
    fi

    if [[ -d "${APP_DIR}/outputs" ]] && [[ -n "$(ls -A "${APP_DIR}/outputs" 2>/dev/null)" ]]; then
        read -rp "是否将已生成的图片备份到 /root/qwen_sd_outputs_backup ? [Y/n]: " bak
        if [[ "${bak:-Y}" =~ ^[Yy]$ ]]; then
            mkdir -p /root/qwen_sd_outputs_backup
            cp -a "${APP_DIR}/outputs/." /root/qwen_sd_outputs_backup/
            log_ok "图库已备份至 /root/qwen_sd_outputs_backup"
        fi
    fi

    close_firewall_port
    rm -rf "$APP_DIR" "$CONF_FILE" "$SYSCTL_FILE"
    sysctl --system >/dev/null 2>&1 || true
    log_ok "已清理程序目录、配置与 sysctl 规则。"

    if [[ -f "$SWAP_FILE" ]]; then
        read -rp "是否同时关闭并删除脚本创建的 Swap 文件 (${SWAP_FILE})? [Y/n]: " rmswap
        if [[ "${rmswap:-Y}" =~ ^[Yy]$ ]]; then
            swapoff "$SWAP_FILE" 2>/dev/null || true
            rm -f "$SWAP_FILE"
            sed -i "\|${SWAP_FILE}|d" /etc/fstab
            log_ok "已清理专用 Swap。"
        fi
    fi

    log_ok "彻底卸载完成，系统已恢复干净状态。"
}

menu() {
    while true; do
        load_conf
        echo
        echo -e "${CYAN}============================================================"
        echo "  全平台本地千问/SD 无限制生图工坊 (x86_64 / ARM64 通用版)"
        echo -e "============================================================${NC}"
        echo "  1) 一键安装 (硬件体检 + 智能Swap + 自检编译 + 鉴权WebUI)"
        echo "  2) 下载千问 Qwen-Image-2.1-Turbo 无审查套件 (Abliterated LLM)"
        echo "  3) 下载 SD 1.5 / SDXL 无审查底模 (Realistic Vision / RealVisXL)"
        echo "  4) 通过直链下载第三方无限制模型与 LoRA (HF / Civitai)"
        echo "  5) 修改监听方式 / 端口 / 鉴权账密 / Netcup CPU限流"
        echo "  6) 更新引擎与 WebUI / 切换 Native-Portable 编译模式"
        echo "  7) 查看运行状态 / 内存与Swap / IPv4-IPv6 访问地址"
        echo "  8) 查看实时服务日志 (Ctrl+C 返回)"
        echo "  9) 一键彻底卸载 (含清理服务 / 防火墙 / Swap / sysctl)"
        echo "  0) 退出"
        echo "------------------------------------------------------------"
        read -rp "请选择操作 [0-9]: " choice
        case "$choice" in
            1) install_all ;;
            2) download_qwen21 ;;
            3) download_sd ;;
            4) download_custom_url ;;
            5) reconfigure_auth ;;
            6) update_engine ;;
            7) show_status ;;
            8) journalctl -u "$SERVICE_NAME" -f || true ;;
            9) uninstall_all ;;
            0) exit 0 ;;
            *) log_err "无效选项" ;;
        esac
        echo
        read -rp "按回车键返回主菜单..." _
    done
}

case "${1:-}" in
    install)       install_all ;;
    download-qwen) download_qwen21 ;;
    download-sd)   download_sd ;;
    update)        update_engine ;;
    auth)          reconfigure_auth ;;
    status)        show_status ;;
    logs)          journalctl -u "$SERVICE_NAME" -f ;;
    uninstall)     uninstall_all ;;
    *)             menu ;;
esac
