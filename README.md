# Home Server

This home server is built from two old and slow HP Pavilion laptops. It runs Linux Mint and a set of services in Docker, and I access it headless over SSH from my MacBook. I built it to learn Linux administration, Docker, and how deployment works.

## Overview

I had two HP Pavilion 15-au laptops, each with something wrong with it. Instead of throwing them out, I took the good parts from each and combined them into one working machine. I then installed Linux Mint, stripped it back to run without a monitor or keyboard, and started using it as an always-on server that I reach from my laptop over the local network.

It currently runs a hardware monitor, with a Git server, a local AI model, and a scraper planned next. This repo is my own reference for how everything is set up, and a record of the problems I ran into and how I fixed them.

## Hardware

The two donor laptops:

- **HP Pavilion 15-au077sa** (2016ish) — older, 6th-gen Intel Core i5-6200U. Had a failing battery, but a nicer screen and keyboard, an SSD, and 8GB RAM (1 of 2 slots).
- **HP Pavilion 15-au181sa** (2017ish) — newer, 7th-gen Intel Core i5-7200U and a better (but still not great) battery, and 8GB RAM (1 of 2 slots), but a slow hard drive.

Both are the same chassis family, so all parts are swappable.

Final machine:

- **CPU:** Intel Core i5-7200U (7th-gen, 2 cores / 4 threads)
- **RAM:** 16 GB DDR4 (two 8 GB sticks, dual-channel)
- **Storage:** 256 GB SanDisk SSD
- **Graphics:** Intel HD Graphics 620 (integrated)
- **OS:** Linux Mint 22.3 Cinnamon

The goal was to end up with the newer processor and the better screen and keyboard in one machine, with both sticks of RAM and the faster SSD.

## The Build

### Hardware assembly

I kept the older laptop's body, because its screen and keyboard were in better condition, and moved the newer laptop's motherboard into it. The CPU is soldered to the board, so the only way to get the newer processor was to swap the whole motherboard.

I also moved across the good battery and both RAM sticks, and kept the SSD that was already in the older machine. The newer laptop's hard drive and old motherboard went into a spare-parts pile.

Reseating the heatsink meant cleaning off the old thermal paste and applying fresh paste. After reassembly the machine booted first time and ran cool under load, so the paste job held up.

### Operating system

I chose Linux Mint Cinnamon because it is stable, beginner-friendly, and built on Ubuntu, which is the system most deployment guides and tools assume. That last point matters for what I am trying to learn.

Setup after install:

- SSH access so I can run the machine without a monitor or keyboard
- Firewall (ufw) turned on, denying incoming traffic by default
- Swappiness lowered to 10, since the machine has plenty of RAM and rarely needs swap
- Timeshift for weekly system snapshots
- TLP for better power management

## Services

| Service | Purpose | Port | How it runs |
|---------|---------|------|-------------|
| Glances | Hardware monitoring (CPU, RAM, disk, temps) | 61208 | Docker |
| Ollama | Local AI model backend | 11434 | Docker |
| Open WebUI | Browser interface for Ollama | 8080 | Docker |

More to come (see Future Plans).

### Glances

A web dashboard showing the server's CPU load, memory, disk usage, network, temperatures, and running processes. Useful because the machine is headless, so this is how I check its health from my laptop. It runs as a Docker container set to restart automatically, so it comes back after a reboot.

Alternatively, I can ssh in via the terminal and run `btop` or `htop`.

## Architecture

```
  MacBook Air                HP Pavilion (server)
  ----------                 --------------------
  Terminal / SSH  --------->  Linux Mint (headless)
  Web browser     --------->    |
                  LAN            +-- Docker
                                      |
                                      +-- Glances (port 61208)
                                      +-- Ollama (port 11434)
                                      +-- Open WebUI (port 8080)
```

I do all the work from the Mac. Code and admin happen over SSH; services are viewed in the browser over the local network.

## Problems & Solutions

### The "good" battery was also worn out

The plan was to keep the newer laptop's battery, assuming it was healthy. After the build, the battery sat at 100% for over an hour of use, then dropped fast — classic broken-gauge behaviour.

Checking the actual figures with `upower` showed why: the battery's full capacity had dropped to about 14 Wh, against an original design figure of around 41 Wh. It was reporting "100% health" only because its controller had quietly rewritten its own baseline down to match the worn capacity. So both batteries I had to choose from were near end of life. The fix is a cheap replacement; the lesson was to measure rather than assume.

### Docker would not install on Mint

Docker's official install instructions add a software repository based on your Ubuntu version's codename. Mint has its own codenames, so following the instructions directly points at a repository that does not exist, and the install fails.

The fix was to find the underlying Ubuntu base (`noble`, for Mint 22.3) and use that codename in the repository line instead of Mint's own. Worth knowing for any Ubuntu-based-but-not-Ubuntu system.

### Glances ran but served nothing

The container started fine and showed as running, but nothing loaded in the browser. Testing on the server itself with `curl` to localhost showed the port was not responding, which meant the problem was the container, not the network.

The cause was that the web-server option was being passed as an environment variable that the image did not pick up. Recreating the container with the web-server flag added as a command instead got it serving.

### Could not reach the dashboard from the Mac

Once Glances was confirmed serving on the server, the browser on the Mac still could not connect. I worked through it one layer at a time: the container was running, the port was open in the firewall, `curl` to localhost on the server worked (so the service was fine), and `ping` from the Mac reached the server (so the network was fine). That narrowed it to something on the Mac.

It turned out recent macOS versions require apps to be granted local-network permission, and Firefox had it switched off. Turning it on fixed it. The useful part was the method: check each step from the service outwards until only one possible cause is left.

### Local AI is not tenable

I successfully installed Ollama and Open WebUI, and they communicate fine, but generating text is incredibly slow.

The cause is hardware limitations. Running LLMs requires significant GPU power and VRAM. This server relies entirely on an older, low-wattage laptop CPU and integrated Intel graphics, so inference falls back entirely to the CPU. Since a dedicated GPU cannot be added to this laptop motherboard, there is no real fix. It succeeded as a deployment experiment, but fails as a practical daily tool.

## What I Learned

- Working comfortably in the Linux command line and over SSH
- The basics of Docker: images, containers, port mapping, restart policies, and why containers make services easy to run and clean up
- How a firewall denies traffic by default and why you open ports deliberately
- A proper way to debug a networked service: test each layer in turn instead of guessing
- That reading hardware figures directly (battery capacity, temperatures) beats trusting the summary a system shows you

## Future Plans

- **Git server** (Forgejo/Gitea) — host my own repositories, and learn Docker Compose
- **AI server** (Ollama) — run a small language model locally, reachable from the Mac
- **Scraper** — a scheduled job that collects and stores data I care about
- **Dashboard** — a single page linking the services together

## Setup Reference

Commands and configuration for rebuilding or recovering the server. (Pull the exact commands from terminal history as you go; this section is for your own future use.)

### Glances

```
docker run -d \
  --restart=unless-stopped \
  -p 61208:61208 \
  --pid host \
  -v /var/run/docker.sock:/var/run/docker.sock:ro \
  --name glances \
  nicolargo/glances:latest \
  glances -w
```

Open the firewall port:

```
sudo ufw allow 61208/tcp
```

View at `http://<server-ip>:61208`.

### Ollama

```
docker run -d \
  --restart=unless-stopped \
  -v ollama:/root/.ollama \
  -p 11434:11434 \
  --name ollama \
  ollama/ollama
```

### Open WebUI

```
docker run -d \
  --restart=unless-stopped \
  -p 8080:8080 \
  --add-host=host.docker.internal:host-gateway \
  -v open-webui:/app/backend/data \
  --name open-webui \
  ghcr.io/open-webui/open-webui:main
```

Open firewall port:

```
sudo ufw allow 8080/tcp
```

View at `http://<server-ip>:8080`.