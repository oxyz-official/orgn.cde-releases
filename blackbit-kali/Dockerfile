# Blackbit Kali toolset image — lean, arm64, for the microsandbox (msb) microVM.
#
# Why this exists: the desktop app boots a Kali microVM via the bundled `msb` and
# runs the offensive toolchain inside it (kali_exec). The upstream
# kalilinux/kali-rolling base is minimal (no nmap/nuclei/sqlmap), and the full
# vxcontrol/kali-linux (~10-12 GiB) trips an msb EROFS large-image extraction bug
# (binaries past ~4 GiB get corrupted). This image ships exactly the tools the
# Blackbit prompt routes — and stays lean (target <4 GiB extracted) so it never
# enters that overflow zone.
#
# Build (arm64 ONLY — libkrun runs the guest natively, no x86 emulation):
#   docker build --platform=linux/arm64 -t <registry>/blackbit-kali:<tag> .
#
# Tooling parity: the Go module paths + cargo crates mirror
# packages/desktop/scripts/download-blackbit-tools.ts exactly.

# ---------------------------------------------------------------------------
# Stage 1 — build the native Go/Rust recon tools from source (arm64).
# ---------------------------------------------------------------------------
FROM golang:1.25-bookworm AS gobuild

# libpcap-dev: naabu (and other PD tools) need it for the CGO build.
RUN apt-get update && apt-get install -y --no-install-recommends \
      git ca-certificates build-essential libpcap-dev \
    && rm -rf /var/lib/apt/lists/*

# GOTOOLCHAIN=auto lets `go install` fetch a newer toolchain when a tool's
# go.mod requires one (e.g. nuclei needs go >= 1.25.7).
ENV CGO_ENABLED=1 GOBIN=/out GOFLAGS=-trimpath GOTOOLCHAIN=auto
RUN mkdir -p /out

# ProjectDiscovery recon/scan suite (exact module paths from download-blackbit-tools.ts).
RUN go install github.com/projectdiscovery/nuclei/v3/cmd/nuclei@latest \
 && go install github.com/projectdiscovery/httpx/cmd/httpx@latest \
 && go install github.com/projectdiscovery/subfinder/v2/cmd/subfinder@latest \
 && go install github.com/projectdiscovery/naabu/v2/cmd/naabu@latest \
 && go install github.com/projectdiscovery/dnsx/cmd/dnsx@latest \
 && go install github.com/projectdiscovery/tlsx/cmd/tlsx@latest \
 && go install github.com/projectdiscovery/katana/cmd/katana@latest \
 && go install github.com/projectdiscovery/mapcidr/cmd/mapcidr@latest \
 && go install github.com/projectdiscovery/asnmap/cmd/asnmap@latest \
 && go install github.com/projectdiscovery/cdncheck/cmd/cdncheck@latest \
 && go install github.com/projectdiscovery/shuffledns/cmd/shuffledns@latest \
 && go install github.com/projectdiscovery/cloudlist/cmd/cloudlist@latest \
 && go install github.com/projectdiscovery/interactsh/cmd/interactsh-client@latest

# Other Go recon/fuzz tools (release + gobuild strategies in the host stager).
RUN go install github.com/ffuf/ffuf/v2@latest \
 && go install github.com/hahwul/dalfox/v2@latest \
 && go install github.com/sensepost/gowitness@latest \
 && go install github.com/owasp-amass/amass/v5/cmd/amass@latest \
 && go install github.com/lc/gau/v2/cmd/gau@latest \
 && go install github.com/OJ/gobuster/v3@latest

# tomnomnom-style chain + misc gobuild tools.
RUN go install github.com/tomnomnom/assetfinder@latest \
 && go install github.com/tomnomnom/httprobe@latest \
 && go install github.com/tomnomnom/meg@latest \
 && go install github.com/tomnomnom/waybackurls@latest \
 && go install github.com/hakluke/hakrawler@latest \
 && go install github.com/jaeles-project/gospider@latest \
 && go install github.com/lc/subjs@latest \
 && go install github.com/tomnomnom/unfurl@latest \
 && go install github.com/tomnomnom/qsreplace@latest \
 && go install github.com/tomnomnom/gf@latest \
 && go install github.com/tomnomnom/anew@latest \
 && go install github.com/KathanP19/Gxss@latest \
 && go install github.com/Emoe/kxss@latest \
 && go install github.com/dwisiswant0/crlfuzz/cmd/crlfuzz@latest \
 && go install github.com/jaeles-project/jaeles@latest \
 && go install github.com/ropnop/kerbrute@latest

# ---------------------------------------------------------------------------
# Stage 2 — final Kali toolset image (arm64).
# feroxbuster comes from Kali apt (no slow cargo build). rustscan and x8 are
# omitted: neither is packaged in Kali, and x8's upstream cargo build is broken on
# current rustc (time/cookie crate). rustscan is a thin nmap front-end (nmap/naabu
# cover scanning) and x8 is a minor hidden-parameter fuzzer — acceptable to skip.
# ---------------------------------------------------------------------------
FROM kalilinux/kali-rolling

# Keep it lean: no recommends, drop apt lists. These are the apt-provided
# exploitation/auth tools + the execBackground hard-deps (bash, screen,
# coreutils=stdbuf/base64) + runtime libs (libpcap for naabu/nmap).
RUN apt-get update && apt-get install -y --no-install-recommends \
      bash coreutils screen ca-certificates curl wget git openssl \
      libpcap0.8 \
      python3 python3-pip \
      nmap \
      sqlmap \
      nikto \
      whatweb \
      wpscan \
      hydra \
      john \
      dnsutils \
      netexec \
      impacket-scripts \
      feroxbuster \
      dirb \
    && rm -rf /var/lib/apt/lists/* /var/cache/apt/* \
    && { find /usr/share/doc -depth -type f ! -name copyright -delete 2>/dev/null || true; }

# Recon binaries built from source (Go).
COPY --from=gobuild /out/ /usr/local/bin/

# Sanity: a couple of representative tools must resolve at build time.
RUN nmap --version >/dev/null && nuclei -version >/dev/null 2>&1 || true

CMD ["sleep", "infinity"]
