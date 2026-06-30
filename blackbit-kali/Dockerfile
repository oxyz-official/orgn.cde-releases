# Blackbit Kali toolset image — arm64, for the microsandbox (msb) microVM.
#
# Why this exists: the desktop app boots a Kali microVM via the bundled `msb` and
# runs the offensive toolchain inside it (kali_exec). The upstream
# kalilinux/kali-rolling base is minimal; the full vxcontrol/kali-linux (~10-12
# GiB) trips an msb EROFS large-image extraction bug (binaries past ~4 GiB get
# corrupted). This image carries the tools the Blackbit agent actually uses while
# staying under ~4 GiB extracted so it never enters that overflow zone.
#
# Deliberately NOT preinstalled (they alone would push past 4 GiB): the giant DATA
# sets `seclists` (~1.3 GiB) and `exploitdb`/`searchsploit` (~2 GiB). They are
# runtime-installable inside the persistent microVM — `apt-get install -y seclists
# exploitdb` — and state survives for the session.
#
# Build (arm64 ONLY — libkrun runs the guest natively, no x86 emulation):
#   docker build --platform=linux/arm64 -t <registry>/blackbit-kali:lean .

# ---------------------------------------------------------------------------
# Stage 1 — build the native Go recon tools from source (arm64).
# Module paths mirror packages/desktop/scripts/download-blackbit-tools.ts.
# ---------------------------------------------------------------------------
FROM golang:1.25-bookworm AS gobuild

RUN apt-get update && apt-get install -y --no-install-recommends \
      git ca-certificates build-essential libpcap-dev \
    && rm -rf /var/lib/apt/lists/*

# GOTOOLCHAIN=auto lets `go install` fetch a newer toolchain when a tool's go.mod
# requires one (e.g. nuclei needs go >= 1.25.7). libpcap-dev: naabu CGO build.
ENV CGO_ENABLED=1 GOBIN=/out GOFLAGS=-trimpath GOTOOLCHAIN=auto
RUN mkdir -p /out

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

RUN go install github.com/ffuf/ffuf/v2@latest \
 && go install github.com/hahwul/dalfox/v2@latest \
 && go install github.com/sensepost/gowitness@latest \
 && go install github.com/owasp-amass/amass/v5/cmd/amass@latest \
 && go install github.com/lc/gau/v2/cmd/gau@latest \
 && go install github.com/OJ/gobuster/v3@latest

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
# Stage 1b — build x8 (hidden-parameter discovery). No arm64 prebuilt binary is
# published, and a plain `cargo install x8` breaks (cookie/time dependency
# resolution). `--locked` uses x8's own Cargo.lock, which pins compatible deps.
# ---------------------------------------------------------------------------
FROM rust:1-bookworm AS x8build
RUN cargo install x8 --locked --root /usr/local

# ---------------------------------------------------------------------------
# Stage 2 — final Kali toolset image (arm64).
# ---------------------------------------------------------------------------
FROM kalilinux/kali-rolling

# apt: the Blackbit kali_exec toolset. --no-install-recommends + doc-strip to stay
# lean. metasploit-framework is the heaviest single package kept; the giant data
# sets (seclists/exploitdb) are left out and installed at runtime if needed.
RUN apt-get update && apt-get install -y --no-install-recommends \
      bash coreutils screen ca-certificates curl wget git unzip openssl libpcap0.8 \
      python3 python3-pip default-jre-headless dnsutils \
      nmap masscan tcpdump \
      sqlmap commix dirsearch wfuzz nikto whatweb wpscan arjun wafw00f \
      sstimap xsstrike paramspider waymore testssl.sh \
      hydra john hashcat \
      netexec impacket-scripts smbmap smbclient enum4linux-ng ldap-utils responder \
      evil-winrm mimikatz certipy-ad \
      metasploit-framework \
      feroxbuster dirb wordlists \
      socat proxychains4 sslscan dnsrecon theharvester sublist3r \
    && rm -rf /var/lib/apt/lists/* /var/cache/apt/* \
    && { find /usr/share/doc -depth -type f ! -name copyright -delete 2>/dev/null || true; }

# Python/git tools the Blackbit prompt routes that are NOT in Kali apt.
# ghauri via pip; the rest cloned to /opt with a PATH wrapper.
RUN pip3 install --break-system-packages --no-cache-dir "git+https://github.com/r0oth3x49/ghauri" \
 && for t in "jwt_tool:ticarpi/jwt_tool:jwt_tool.py" \
             "ssrfmap:swisskyrepo/SSRFmap:ssrfmap.py" \
             "corsy:s0md3v/Corsy:corsy.py" \
             "smuggler:defparam/smuggler:smuggler.py" \
             "oralyzer:r0075h3ll/Oralyzer:oralyzer.py"; do \
      name="${t%%:*}"; rest="${t#*:}"; repo="${rest%%:*}"; entry="${rest##*:}"; \
      git clone --depth 1 "https://github.com/$repo" "/opt/$name" \
      && if [ -f "/opt/$name/requirements.txt" ]; then \
           pip3 install --break-system-packages --no-cache-dir -r "/opt/$name/requirements.txt" || true; \
         fi \
      && printf '#!/bin/sh\nexec python3 /opt/%s/%s "$@"\n' "$name" "$entry" > "/usr/local/bin/$name" \
      && chmod +x "/usr/local/bin/$name"; \
    done

# SecLists in its own layer (keeps total ~3.5 GiB, under the EROFS ceiling, and
# isolates this large data set into a single <4 GiB blob). The agent's deep-recon
# mode depends on medium/large SecLists at /usr/share/seclists. exploit-db is
# deliberately NOT baked in — it's ~2 GiB, ages quickly, and is runtime-installable
# (`apt-get install -y exploitdb`) when an exploitation task needs searchsploit.
RUN apt-get update && apt-get install -y --no-install-recommends seclists \
    && rm -rf /var/lib/apt/lists/* /var/cache/apt/*

# rustscan — fast port discovery. Not in Kali apt; use the published aarch64-linux
# prebuilt binary.
RUN curl -fsSL -o /tmp/rustscan.zip \
      https://github.com/bee-san/RustScan/releases/download/2.4.1/aarch64-linux-rustscan.zip \
    && unzip -o /tmp/rustscan.zip -d /tmp/rustscan \
    && find /tmp/rustscan -type f -name 'rustscan' -exec install -m 0755 {} /usr/local/bin/rustscan \; \
    && rm -rf /tmp/rustscan /tmp/rustscan.zip

# Native Go recon binaries + the x8 build.
COPY --from=gobuild /out/ /usr/local/bin/
COPY --from=x8build /usr/local/bin/x8 /usr/local/bin/x8

# Sanity check a representative tool from each install path.
RUN nmap --version >/dev/null && msfconsole --version >/dev/null 2>&1 \
    && rustscan --version >/dev/null && x8 --version >/dev/null 2>&1 || true

CMD ["sleep", "infinity"]
