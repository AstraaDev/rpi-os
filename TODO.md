# 🖥️ Bare Metal OS pour Raspberry Pi 4 — Roadmap vers DOOM

> **Architecture cible** : ARM Cortex-A72 (AArch64) — Raspberry Pi 4B  
> **Objectif final** : Lancer un ELF natif, clavier/souris/écran/son, puis DOOM

---

## PHASE 0 — Environnement de développement

- [ ] Installer une toolchain cross-compile AArch64
  - `aarch64-none-elf-gcc` (ARM GNU toolchain ou crosstool-ng)
  - `aarch64-none-elf-ld`, `objcopy`, `objdump`
- [ ] Installer QEMU pour émulation rapide
  - `qemu-system-aarch64 -M raspi4b`
- [ ] Créer la structure de projet
  ```
  /src
    /boot
    /kernel
    /drivers
    /lib
  /linker
  /tools
  Makefile
  ```
- [ ] Écrire un `Makefile` de base (cross-compile → ELF → binary)
- [ ] Préparer une carte SD bootable (FAT32 + firmware Broadcom)
  - Récupérer `bootcode.bin`, `start4.elf`, `fixup4.dat` depuis le repo officiel RPi
  - Écrire un `config.txt` minimal (`arm_64bit=1`, `kernel=kernel8.img`)

---

## PHASE 1 — Boot & entrée en AArch64

- [ ] Comprendre la séquence de boot RPi4
  - GPU (VPU) boot → charge `start4.elf` → charge `kernel8.img` à `0x80000`
  - Le CPU entre en EL2 (hypervisor mode) par défaut
- [ ] Écrire le point d'entrée assembleur (`boot/start.S`)
  - Vérifier le core ID (`MPIDR_EL1`), parquer les cores 1/2/3 en `wfe`
  - Passer de EL2 → EL1 (ou rester EL2 selon le choix d'architecture)
  - Initialiser le Stack Pointer
  - Mettre la BSS à zéro
  - Sauter vers `kernel_main()` en C
- [ ] Écrire le linker script (`linker/linker.ld`)
  - Adresse de base `0x80000`
  - Sections : `.text`, `.rodata`, `.data`, `.bss`
- [ ] Tester : kernel minimal qui boucle infiniment → vérifier avec QEMU

---

## PHASE 2 — UART (debug console)

- [ ] Identifier l'UART PL011 du RPi4 (UART0, base `0xFE201000`)
  - Désactiver le mini-UART (UART1) qui partage les GPIO
- [ ] Configurer les GPIO 14/15 en mode ALT0 (UART0 TX/RX)
  - Lire le BCM2711 datasheet pour les registres GPIO FSEL/GPPUD
- [ ] Implémenter le driver UART (`drivers/uart.c`)
  - Init : baud rate, disable FIFO, enable TX/RX
  - `uart_putc()`, `uart_puts()`, `uart_getc()`
- [ ] Implémenter `printf()` minimaliste (ou rerouter vers UART)
- [ ] Tester : afficher "Hello Bare Metal!" via minicom/screen sur le host

---

## PHASE 3 — MMU & mémoire virtuelle

- [ ] Comprendre la MMU AArch64 (tables de pages 4 niveaux, granule 4KB)
- [ ] Écrire les tables de translation initiales (`boot/mmu.c`)
  - Identity mapping de la RAM (0x00000000 → 0xFFFFFFFF)
  - Mapping device memory (périphériques BCM2711 à `0xFE000000+`)
  - Attributs : Normal memory (cacheable) vs Device memory (nGnRnE)
- [ ] Activer le cache L1/L2 (instructions + données)
- [ ] Activer la MMU via `SCTLR_EL1`
- [ ] Implémenter un allocateur mémoire physique simple (buddy ou bitmap)
- [ ] Implémenter `kmalloc()` / `kfree()` (slab ou bump allocator au début)
- [ ] Tester : allouer/libérer de la mémoire, vérifier l'absence de crash

---

## PHASE 4 — Interruptions & timer

- [ ] Étudier le GIC-400 (Generic Interrupt Controller) du BCM2711
- [ ] Initialiser le GIC (`drivers/gic.c`)
  - Distributor (GICD) + CPU Interface (GICC)
- [ ] Écrire la table des vecteurs d'exception (`boot/vectors.S`)
  - `VBAR_EL1` pointer vers la table
  - Handlers : Synchronous, IRQ, FIQ, SError pour EL1t/EL1h/EL0
- [ ] Implémenter le handler IRQ générique avec dispatch
- [ ] Configurer le System Timer BCM2711 (ou ARM Generic Timer `CNTPCT_EL0`)
  - Générer une IRQ toutes les N ms
- [ ] Implémenter `sleep_ms()`, `get_tick_ms()`
- [ ] Tester : blinker une LED sur GPIO ou afficher un compteur toutes les secondes

---

## PHASE 5 — Chargeur d'ELF

- [ ] Implémenter un parser ELF64 (`kernel/elf_loader.c`)
  - Vérifier le magic number et l'architecture (`EM_AARCH64`)
  - Parser les Program Headers (type `PT_LOAD`)
  - Copier chaque segment à son `p_vaddr`
  - Récupérer l'entry point (`e_entry`)
- [ ] Créer un espace d'adressage utilisateur (tables de pages séparées)
  - Mode EL0 pour les processus utilisateur
- [ ] Implémenter le saut vers l'entry point via `eret` vers EL0
- [ ] Implémenter des syscalls de base (`write`, `exit`, `read`)
  - Handler SVC dans la table des vecteurs
- [ ] Tester : compiler un hello world en C avec `aarch64-none-elf-gcc`, le charger et l'exécuter

---

## PHASE 6 — Écran (Framebuffer via Mailbox VideoCore)

- [ ] Comprendre le protocole Mailbox RPi (canal 8 = property tags)
- [ ] Implémenter le driver Mailbox (`drivers/mailbox.c`)
  - `mailbox_call()` avec alignement 16 octets
- [ ] Requête framebuffer via property tags
  - `TAG_SET_PHYSICAL_SIZE`, `TAG_SET_VIRTUAL_SIZE`
  - `TAG_SET_DEPTH` (32 bpp), `TAG_GET_FRAMEBUFFER`
- [ ] Obtenir le pointeur framebuffer, la pitch et la résolution
- [ ] Implémenter les primitives graphiques (`drivers/fb.c`)
  - `fb_put_pixel()`, `fb_fill_rect()`, `fb_clear()`
- [ ] Charger une police bitmap (PSF1 ou PSF2) pour le texte
  - `fb_putchar()`, `fb_puts()`
- [ ] Tester : afficher des couleurs, du texte, un dégradé

---

## PHASE 7 — USB & clavier/souris (HID)

> ⚠️ **La partie la plus complexe du projet** — le RPi4 utilise un contrôleur USB Synopsys DWC OTG via le VL805 (USB 3.0 hub chip)

- [ ] Étudier le contrôleur DWC2/DWC3 (Synopsys DesignWare)
  - Alternative : utiliser un driver existant (ex: port de U-Boot ou d'un OS existant)
- [ ] Implémenter le driver DWC OTG (`drivers/usb/`)
  - Initialisation host mode
  - Énumération USB (réinitialisation bus, adressage, descripteurs)
  - Control transfers, Interrupt transfers
- [ ] Implémenter la stack USB HID
  - Parser les HID Report Descriptors
  - Driver clavier (keycodes → ASCII/scancodes)
  - Driver souris (boutons, axes relatifs X/Y)
- [ ] Exposer une API simple : `keyboard_getkey()`, `mouse_get_state()`
- [ ] Tester : afficher les touches pressées à l'écran

> 💡 **Raccourci possible** : utiliser un clavier/souris PS/2 via adaptateur USB→PS/2, le contrôleur i8042 étant bien plus simple à implémenter.

---

## PHASE 8 — Son (PWM ou HDMI Audio)

### Option A — PWM Audio (jack 3.5mm)
- [ ] Configurer les GPIO 40/41 en mode PWM (ALT0)
- [ ] Implémenter le driver PWM (`drivers/pwm.c`)
  - Configurer le clock manager du BCM2711 (PLLD = 500MHz)
  - Initialiser PWM0 channel 1 et 2 (stéréo)
- [ ] Implémenter un buffer audio circulaire
- [ ] Générer un signal PCM (16-bit, 44100Hz ou 22050Hz) via DMA
  - Configurer un canal DMA BCM2711 pour transfert PWM continu
- [ ] Exposer une API : `audio_play_buffer(samples, len)`
- [ ] Tester : jouer un beep, puis un fichier WAV hardcodé

### Option B — HDMI Audio (via Mailbox)
- [ ] Utiliser les property tags VideoCore pour activer l'audio HDMI
- [ ] Envoyer des samples PCM via le Mailbox (plus simple mais latence)

---

## PHASE 9 — Système de fichiers (pour charger DOOM)

- [ ] Implémenter un driver SD card (EMMC/SDHOST du BCM2711)
  - Protocole SD SPI ou SDIO natif
  - Commandes SD : CMD0, CMD8, ACMD41, CMD2, CMD3, CMD17/18
- [ ] Implémenter un parser FAT32
  - Lire le MBR et le VBR
  - Parser les clusters, la FAT, les entrées de répertoire
  - `fat_open()`, `fat_read()`, `fat_close()`
- [ ] Tester : lire un fichier texte depuis la carte SD et l'afficher

---

## PHASE 10 — DOOM 🔥

- [ ] Récupérer le source de DOOM (id Software, Chocolate Doom ou doomgeneric)
  - **doomgeneric** est idéal : il abstrait le rendu/input avec une interface simple
- [ ] Porter doomgeneric sur le kernel
  - `DG_Init()` → init framebuffer
  - `DG_DrawFrame()` → blitter le framebuffer 320×200 vers l'écran
  - `DG_SleepMs()` → timer
  - `DG_GetTicksMs()` → tick counter
  - `DG_GetKey()` → keyboard input
  - `DG_SetWindowTitle()` → no-op
- [ ] Implémenter `libc` minimale nécessaire à DOOM
  - `malloc/free`, `memcpy`, `memset`, `strlen`, `strcmp`, `sprintf`
  - `fopen/fread/fclose` → rerouter vers FAT32
  - `exit()` → halt CPU
- [ ] Fournir le WAD (`doom1.wad` shareware ou commercial)
  - Le lire depuis la carte SD
- [ ] Compiler DOOM avec la toolchain AArch64
- [ ] Linker DOOM dans le kernel ou le charger comme ELF
- [ ] Tester et déboguer 🎮

---

## PHASE 11 — (Bonus) Réseau

- [ ] Driver Ethernet Gig (LAN7515 USB→Ethernet sur RPi4 via USB ou le GENET intégré)
  - Préférer le GENET (Broadcom GENET) connecté directement au SoC
  - Base : `0xFD580000`
- [ ] Implémenter la stack réseau
  - Ethernet II (frames)
  - ARP
  - IPv4
  - UDP
  - TCP (ou lwIP pour aller plus vite)
  - DHCP client
  - DNS client
- [ ] Implémenter un socket API minimal
- [ ] Tester : ping, HTTP GET simple

---

## Ressources clés

| Ressource | Lien |
|---|---|
| BCM2711 datasheet | [RPi Hardware](https://datasheets.raspberrypi.com/bcm2711/bcm2711-peripherals.pdf) |
| ARM Architecture Reference Manual (AArch64) | [ARM Developer](https://developer.arm.com/documentation/ddi0487) |
| RPi bare metal examples (bztsrc) | [github.com/bztsrc/raspi3-tutorial](https://github.com/bztsrc/raspi3-tutorial) |
| doomgeneric | [github.com/ozkl/doomgeneric](https://github.com/ozkl/doomgeneric) |
| RPi4 bare metal (isometimes) | [github.com/isometimes/rpi4-osdev](https://github.com/isometimes/rpi4-osdev) |
| OSDev Wiki | [wiki.osdev.org](https://wiki.osdev.org) |
| Writing an OS in Rust (ARM) | Référence pour la gestion mémoire |
| GIC-400 TRM | [ARM InfoCenter](https://developer.arm.com/documentation/ddi0471) |

---

## Ordre de priorité recommandé

```
Phase 0 → 1 → 2 → 4 → 6 → 5 → 3 → 9 → 7 → 8 → 10 → 11
  env     boot uart  irq  fb  elf mmu  fs  usb sound doom net
```

> Commencer par UART avant MMU : déboguer sans console = cauchemar.  
> L'USB est la partie la plus longue — prévoir 2-3x plus de temps que le reste.
