# GBEmu — émulateur Game Boy pour Casio Graph 35+E II

🇬🇧 [English version](README.md) · 🇫🇷 Français

Émulateur Game Boy (DMG, monochrome) écrit en C pour la **Casio Graph 35+E II**
(processeur SH-4A, écran 128×64), avec le **fxSDK / gint** de la communauté
Planète Casio.

> **État du projet : expérimental.** Un add-in `.g1a` compile, se lance sur une
> vraie calculatrice, exécute une ROM MBC1 de 64 Ko et affiche l'image. Il est
> **très lent** (vitesse non encore chiffrée, voir [Benchmark](#2-benchmark)) et
> **sans son**. Ce n'est pas un émulateur complet.

## Sommaire

0. [Avant de commencer](#0-avant-de-commencer) — légal, prérequis, build
1. [Émulation](#1-émulation) — architecture, ce qui marche, ce qui manque
2. [Benchmark](#2-benchmark) — mesurer la vitesse, régler les options
3. [Erreurs possibles](#3-erreurs-possibles) — symptôme → cause → solution
4. [Tests](#4-tests) — valider le cœur sur PC et sur big-endian
5. [Pistes d'amélioration](#5-pistes-damélioration)

---

## 0. Avant de commencer

### Légal

**Aucune ROM n'est fournie et aucune ne doit être publiée dans ce dépôt**
(le `.gitignore` exclut `*.gb`, `src/rom.c` et `include/rom.h`, qui contiennent
la ROM). N'utilise que des ROMs dumpées depuis des cartouches que tu possèdes, et
renseigne-toi sur la copie privée dans ton pays. Les ROMs de test de Blargg
utilisées pour valider le cœur sont des outils de test libres, téléchargés à la
demande par un script, pas inclus.

*Game Boy* est une marque de Nintendo, *Casio* une marque de Casio. Ce projet
n'est affilié à aucune des deux.

### Prérequis

- Une **Casio Graph 35+E II** (testé uniquement sur ce modèle).
- Linux, ou **Windows 11 avec WSL2 (Ubuntu)**. Le fxSDK n'existe pas pour
  Windows natif.
- Paquets système :
  ```sh
  sudo apt install curl git python3 build-essential pkg-config cmake \
      python3-pil libpng-dev libusb-1.0-0-dev libudisks2-dev libsdl2-dev \
      libncurses-dev libarchive-dev libzip-dev libpopt-dev
  ```
- Le **fxSDK + gint** (≥ 2.9 ; testé avec fxSDK 2.11.0, gint 2.11.0,
  sh-elf-gcc 14.1.0), installés via GiteaPC :
  ```sh
  curl "https://git.planet-casio.com/Lephenixnoir/GiteaPC/raw/branch/master/install.sh" -o /tmp/giteapc-install.sh
  bash /tmp/giteapc-install.sh        # accepter l'ajout au PATH, puis rouvrir le terminal

  giteapc install Lephenixnoir/fxsdk Lephenixnoir/sh-elf-binutils Lephenixnoir/sh-elf-gcc
  giteapc install Lephenixnoir/OpenLibm Vhex-Kernel-Core/fxlibc
  giteapc install Lephenixnoir/sh-elf-gcc      # 2e fois : normal, termine la libstdc++
  giteapc install Lephenixnoir/gint
  ```
  La compilation de GCC prend longtemps (15–30 min).

### Construire

```sh
git clone <url-de-ce-depot> gbemu && cd gbemu

# 1. Convertir TA ROM en tableau C (placé en flash, pas en RAM).
#    Mets le chemin entre guillemets s'il contient des espaces ou des '&'.
python3 tools/rom2c.py "chemin/vers/ta_rom.gb" src/rom.c include/rom.h

# 2. Compiler
fxsdk build-fx
```

`rom2c.py` affiche le titre, le type de cartouche (MBC) et prévient si le type
n'est pas géré. Le fichier `GBEmu.g1a` est produit par le build (cherche-le avec
`find . -name '*.g1a'`).

> Ce dépôt a une structure proche d'un projet créé par `fxsdk new`. Si
> `fxsdk build-fx` se plaint de la structure avec ta version du fxSDK, crée un
> projet vierge (`fxsdk new MonProjet`), puis copies-y `src/`, `include/` et
> remplace son `CMakeLists.txt` par celui-ci.

### Envoyer sur la calculatrice

- **Linux natif** : `fxsdk send GBEmu.g1a` (nécessite `p7`/`fxlink`).
- **WSL** : `fxsdk send` ne fonctionne pas (pas d'USB dans WSL). Copie le
  `.g1a` côté Windows (explorateur → `\\wsl$\Ubuntu\home\<toi>\gbemu\…`) puis
  transfère-le avec le logiciel officiel Casio, calculatrice en mode
  **LINK → RECV**.

### Touches

| Calculatrice | Game Boy |
|---|---|
| Flèches | Croix directionnelle |
| SHIFT | A |
| ALPHA | B |
| EXE | START |
| OPTN | SELECT |
| F1 | Bascule affichage **réduit** ⇄ **recadré** |
| F2 / F3 | Déplace la fenêtre à gauche / droite (mode recadré) |
| F5 / F6 | Déplace la fenêtre vers le haut / bas (mode recadré) |
| EXIT | Quitter |

---

## 1. Émulation

### Contraintes de la cible

| | Game Boy | Graph 35+E II |
|---|---|---|
| CPU | SM83, 4,19 MHz | SH-4A, **big-endian** |
| Écran | 160×144, 4 teintes | 128×64, 1 bit (4 niveaux simulés par le moteur `dgray` de gint) |
| RAM utile | 8 Ko + 8 Ko VRAM | région statique très réduite ; le tas de gint est plus grand |

### Architecture

```
include/gb.h     structures (CPU, MMU, PPU, timer), accesseurs de registres
src/cpu.c        cœur SM83 : décodage par champs x/y/z/p/q, préfixe CB, interruptions
src/mmu.c        bus mémoire, MBC 0/1/2/3/5, E/S, DMA OAM, joypad, port série (hook)
src/ppu.c        machine à états LCD (modes 0-3) + rendu ligne par ligne
src/timer.c      DIV / TIMA / TMA / TAC
src/gb.c         assemblage de la machine et boucle d'une image
src/main.c       frontend gint : affichage, clavier, options de compilation
tools/rom2c.py   ROM .gb → src/rom.c + include/rom.h
tools/test_host.c, tools/blargg_test.c, tools/run_blargg.sh   tests sur PC
```

### Choix techniques qui comptent

**Registres CPU sans union (endianness).** Les paires `AF/BC/DE/HL` sont de
simples `uint16_t` ; les octets haut/bas sont extraits par décalage/masque
(`r_a()`, `set_b()`…). Une version précédente utilisait des `union` qui
supposent le little-endian : elle passait tous les tests sur PC (x86) mais
**cassait l'émulation sur la calculatrice** (SH-4 en big-endian, cible
`sh3eb-elf`). Reproduit et vérifié sous QEMU big-endian : voir [Tests](#4-tests).

**Mémoire.** `gb_t` (~22 Ko) est alloué par `malloc`, jamais en variable
statique : en statique, l'édition de liens échoue (`region 'ram' overflowed`).
Le framebuffer est stocké à **2 bits par pixel** (5 760 octets au lieu de
23 040). La ROM est un tableau `const` en flash.

**Affichage 160×144 → 128×64.** Deux modes (F1) :
- *réduit* : plus-proche-voisin, tout l'écran Game Boy visible, mais
  environ 1 ligne sur 2 est perdue — le texte est souvent peu lisible ;
- *recadré* : pixels natifs 1:1 sur une fenêtre 128×64 déplaçable à la main —
  net, mais partiel.

Les 4 teintes Game Boy correspondent aux 4 niveaux du moteur `dgray`
(écriture directe dans les deux plans de bits). Sans `dgray`
(`GBEMU_NO_GRAY`), repli sur un tramage ordonné (Bayer 4×4) en 1 bit.

**PPU.** Le fond est rendu par tuile (une lecture VRAM tous les 8 pixels, accès
direct à `vram[]` sans passer par le bus). Quand le LCD est éteint, les bits de
mode de `STAT` sont forcés à 0, comme sur le matériel.

### Ce qui est implémenté

- Jeu d'instructions SM83 complet, préfixe CB, interruptions, HALT, EI différé.
- MBC : aucun, MBC1, MBC2 (banques ROM), MBC3 (sans horloge), MBC5.
- Fond, fenêtre, sprites 8×8 et 8×16, retournements X/Y, priorité fond/sprite,
  palettes BGP/OBP0/OBP1, DMA OAM.
- Timer, joypad avec interruption, port série (capture pour les tests).
- Démarrage direct à `0x0100` avec les registres d'après le boot ROM (le boot
  ROM n'est pas émulé).

### Ce qui manque ou est approximatif

- **Aucun son.**
- Pas de sauvegarde : la SRAM de cartouche est allouée (plafonnée à 32 Ko) mais
  jamais écrite sur la mémoire de la calculatrice.
- MBC3 : pas d'horloge temps réel. MBC2 : pas de RAM interne.
- Pas de Game Boy Color (mode DMG uniquement ; `rom2c.py` prévient).
- DMA OAM instantané ; accès mémoire non exacts au cycle près.
- Priorité des sprites : ordre d'OAM, pas la règle « X le plus petit gagne ».
- Compteur de ligne de la fenêtre simplifié.
- Interruptions `STAT` sans détection de front (peuvent être redemandées).
- Timer : pas de délai d'un cycle au rechargement de `TIMA`.
- Jeux testés : **un seul** (une ROM MBC1 de 64 Ko). La compatibilité générale
  est inconnue. ROM > 1 Mo : non testé.

### Validation

Suites de Blargg exécutées sur le cœur compilé pour PC :

| Suite | Résultat |
|---|---|
| `cpu_instrs` (les 11 tests individuels) | ✅ passent (x86 **et** big-endian) |
| `instr_timing` | ✅ passe (x86 **et** big-endian) |
| `mem_timing` / `01-read_timing` | ❌ échoue (accès mémoire non cycle-exacts) |
| autres suites (`interrupt_time`, `oam_bugs`, `halt_bug`, …) | non exécutées |

Ces suites valident le cœur d'instructions, **pas** le PPU ni la compatibilité
avec de vrais jeux.

---

## 2. Benchmark

La vitesse n'a **pas encore été mesurée** : on sait seulement qu'elle est très
inférieure à 100 %. Un mode benchmark est fourni pour la chiffrer sur ta
calculatrice, sans outil particulier (juste un chronomètre).

### Principe

Il exécute exactement **300 images** Game Boy (≈ 5 s de temps de jeu à
59,7 images/s) avec **rendu et affichage à chaque image** (pire cas
volontaire), entre un écran « GO ! » et un écran « FINI ».

### Procédure

1. Dans `CMakeLists.txt`, décommente `set(GBEMU_BENCHMARK ON)`.
2. Reconstruis proprement :
   ```sh
   rm -rf build-fx && fxsdk build-fx
   ```
3. Transfère et lance l'add-in. Appuie sur **EXE**.
4. Lance ton chrono quand **« GO ! »** apparaît, arrête-le quand **« FINI »**
   apparaît.
5. Calcule : **vitesse = 5 / T × 100 %** (T en secondes). Temps réel = 5 s.

### Résultats

À compléter avec tes mesures :

| Configuration | T (s) | Vitesse |
|---|---|---|
| Défaut (gris, réduit) | | |
| `GBEMU_NO_GRAY` | | |

Précise aussi le jeu utilisé (le coût dépend du jeu) et le modèle de
calculatrice.

### Options de réglage

À modifier en tête de `CMakeLists.txt`, puis `rm -rf build-fx && fxsdk build-fx` :

| Option | Effet |
|---|---|
| `set(GBEMU_FRAMESKIP N)` | Affiche 1 image sur N+1 (défaut 1). L'émulation garde son rythme, seul l'affichage est espacé. Sans effet sur le benchmark, qui affiche toujours tout. |
| `set(GBEMU_NO_GRAY ON)` | Désactive les 4 niveaux de gris : moins de travail par image, rendu moins fidèle. |
| `set(GBEMU_BENCHMARK ON)` | Compile le benchmark à la place du jeu. |

`main.c` appelle aussi `clock_set_speed(CLOCK_SPEED_F3)` pour demander à gint
un mode de fréquence plus rapide ; son effet réel sur ce modèle n'a pas été
mesuré. Si l'add-in devient instable, retire cet appel (et celui de
`CLOCK_SPEED_DEFAULT` à la sortie).

---

## 3. Erreurs possibles

Toutes ont été rencontrées pendant le développement.

### Installation de la chaîne d'outils

| Symptôme | Cause | Solution |
|---|---|---|
| Le `curl … install.sh \| bash` n'installe rien | URL en `/src/branch/` : renvoie la page HTML, pas le script | Utiliser `/raw/branch/` (voir §0) ; télécharger puis `bash` le fichier |
| `fxsdk: command not found` alors que `giteapc list` montre fxsdk | `~/.local/bin` absent du `PATH` | `echo 'export PATH="$PATH:$HOME/.local/bin"' >> ~/.bashrc && source ~/.bashrc` |
| `Package 'libpng16' … not found` | Paquet de dev manquant | `sudo apt install libpng-dev` |
| `Package 'ncurses' … not found` | idem | `sudo apt install libncurses-dev` |
| Autre `Package 'XXX' not found` | idem | `sudo apt install libXXX-dev` |
| `sh-elf-gcc` s'installe deux fois | Normal | Rien à faire |

### Compilation

| Symptôme | Cause | Solution |
|---|---|---|
| `fatal error: gb.h: No such file or directory` | `include/` non déclaré, ou `CMakeLists.txt` du template resté en place | Utiliser le `CMakeLists.txt` de ce dépôt (`target_include_directories(... include)`) |
| Seul `main.c` est compilé | Liste `SOURCES` du template non remplacée | Idem |
| `region 'ram' overflowed by N bytes` | Grosse structure en variable statique (`.bss`) | Allouer sur le tas : c'est déjà le cas (`malloc(sizeof(gb_t))`). Ne pas remettre `gb_t` en `static` |
| `implicit declaration of function 'ppu_set_pixel'` | Ancien `gb.h` en place (mélange de versions) | Remplacer tous les fichiers par la même version, `rm -rf build-fx` |
| `implicit declaration of function 'sprintf'` | `<stdio.h>` manquant (si tu ajoutes du code de debug) | `#include <stdio.h>` |
| `redefinition of 'xxx'` / erreur bizarre dans `stddef.h` | Fichier édité à la main en plusieurs fois, accolades ou fonction en double | Repartir d'un fichier propre du dépôt |
| `rom.c`/`rom.h` introuvables | ROM pas encore convertie | `python3 tools/rom2c.py "ta_rom.gb" src/rom.c include/rom.h` |
| `rom2c.py` : `No such file or directory` avec un chemin | Espace ou `&` non protégé | Mettre le chemin entre guillemets |

### Envoi

| Symptôme | Cause | Solution |
|---|---|---|
| `p7 is not installed or not available` | `p7` absent | Non bloquant pour la compilation ; transférer le `.g1a` autrement |
| `fxsdk send` échoue sous WSL | Pas d'accès USB dans WSL2 | Copier le `.g1a` côté Windows et utiliser le logiciel Casio |

### À l'exécution

| Symptôme | Cause probable | Solution |
|---|---|---|
| L'add-in apparaît mais ne se lance pas / revient au menu | Plantage au démarrage | Compiler une version avec des affichages de progression (`dtext` + `getkey()` à chaque étape de `main`) pour localiser l'étape qui échoue |
| Écran blanc, le CPU tourne (compteur de cycles cohérent) mais `LCDC` bit 7 à 0 | **Bug d'endianness** si tu modifies les registres CPU avec des `union` ; ou jeu qui attend un comportement matériel manquant | Ne jamais lire les octets d'une paire via une `union` (voir §1) ; valider sous big-endian (§4) |
| `malloc(gb_t) a echoue` affiché | Pas assez de tas | Fermer d'autres add-ins ; ne pas agrandir `gb_t` |
| Le jeu est très lent | Cœur interprété sur un CPU modeste | §2 : mesurer, puis `GBEMU_FRAMESKIP`, `GBEMU_NO_GRAY` ; voir aussi §5 |
| Texte illisible | Mode réduit (144 lignes → 64) | **F1** pour le mode recadré, **F2/F3/F5/F6** pour déplacer la fenêtre |
| Graphismes faux / jeu qui part en vrille | Type de cartouche ou fonction non gérés | Lire l'avertissement de `rom2c.py`, vérifier [les limites](#ce-qui-manque-ou-est-approximatif) |
| `rom2c.py` avertit « Game Boy Color » | ROM CGB | Seul le mode DMG est émulé : affichage probablement incorrect |

---

## 4. Tests

Le cœur (`cpu/mmu/ppu/timer/gb`) ne dépend pas de gint et se compile sur PC.

```sh
make test      # tests unitaires CPU + rendu d'une mini-ROM faite main
make blargg    # télécharge les ROMs de test de Blargg et lance cpu_instrs + instr_timing
```

### Vérifier sur une architecture big-endian

C'est le test qui aurait détecté le bug d'endianness. Exemple avec s390x
(big-endian) émulé par QEMU, sur Ubuntu :

```sh
sudo apt install gcc-s390x-linux-gnu qemu-user
s390x-linux-gnu-gcc -static -std=c11 -O2 -Iinclude \
    src/cpu.c src/mmu.c src/ppu.c src/timer.c src/gb.c tools/blargg_test.c -o blargg_be
qemu-s390x ./blargg_be "tools/gb-test-roms/cpu_instrs/individual/06-ld r,r.gb" 15
```

La sortie doit se terminer par `[OK]`. **Fais-le après toute modification du
cœur CPU.**

---

## 5. Pistes d'amélioration

Non implémentées, non testées :

- **Ne rendre que les lignes affichées.** En mode réduit, seules ~64 lignes sur
  144 sont utilisées par l'écran ; le PPU les dessine toutes.
- **Écrire les pixels par octets entiers** plutôt que par lecture-modification-
  écriture de chaque pixel de 2 bits.
- **Chemin rapide de `mmu_read`** (table de pages) pour éviter la cascade de
  comparaisons à chaque accès.
- Son, sauvegarde SRAM, horloge MBC3, sprites à la priorité exacte.
- Passer plus de suites de test (`mem_timing`, `interrupt_time`, `halt_bug`,
  `oam_bugs`).

## Crédits

- [fxSDK / gint](https://git.planet-casio.com/Lephenixnoir/fxsdk) et GiteaPC,
  par Lephenixnoir et la communauté [Planète Casio](https://www.planet-casio.com).
- ROMs de test de Blargg (validation uniquement, non incluses), récupérées via
  le dépôt `retrio/gb-test-roms`.

## Licence

À définir : ajoute un fichier `LICENSE` avant de publier (par exemple MIT ou
GPL-3.0 selon ce que tu souhaites).
