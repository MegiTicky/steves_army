# Steve's Army

**Enlisted-style military squad command for Minecraft 1.20.1**

[Forge | TaCZ optional]

---

### What it does

Command up to 9 AI soldiers with modern firearms. Squad modes (Follow/Hold), intelligent cover system (seek, peek, suppress), ping wheel for tactical orders, and optional TaCZ gun integration.

---

### Key features

| | |
|---|---|
| **Squad command** | G to toggle Follow/Hold, Middle Mouse for ping wheel (look-at, move-to, enemy-spotted) |
| **Cover AI** | Soldiers find, evaluate, and reposition between cover points. Half-cover crouch-peek, full-cover side-step peek, suppression mechanics |
| **Combat** | Realistic accuracy (aim before shoot), inventory-based ammo, TaCZ support via reflection |
| **Respawn as squadmate** | Die as player? Respawn as one of your soldiers |

---

### Tech

- Minecraft 1.20.1, Forge 47.4.0, Parchment mappings
- Optional TaCZ dependency (`mandatory=false`)
- 9-slot soldier inventory with GUI
- `/stevesarmy_debug` and `/stevesarmy_cover` commands

---

*This is an early peek at the mod. Code architecture is under active development.*