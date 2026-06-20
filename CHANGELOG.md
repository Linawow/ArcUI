## 3.7.3

### New Features

- **Patch 12.1 Support**: ArcUI is now compatible with patch 12.1. That patch is brand new, so some new errors may show up there that did not happen before; please report anything you run into so it can be fixed quickly.
- **Share Castbar Across Characters**: Optional setting, off by default, that uses one castbar look on every character, starting from the castbar you already have set up, with each character keeping its own on-screen position unless you also share the position.
- **Castbar Import and Export**: Share your full castbar setup as a string and load it on another character, or bundle it into your bars export so colors, fonts, per-cast-type profiles, thresholds, and position travel together.
- **Import a Castbar as a Saved Skin**: When a shared string includes a castbar, the import lets you either replace your live castbar or save the incoming one as a named skin you can apply later.
- **Hide Blizzard Castbar**: Optional toggle, off by default, that hides the default Blizzard castbar, and turning it back on restores the bar without reloading.
- **Movable Spell Icon**: Optional setting, off by default, that lets you drag the castbar's spell icon to a custom position while the options panel is open, with a reset button to restore it.
- **Shorten Long Spell Names**: Optional setting, off by default, that trims spell names longer than a chosen length so they fit on the castbar.
- **Resource Bar Text Color by Value**: Optional, off by default: resource bar value text can change color based on how full the resource is, with up to four color zones plus a base color and a choice of Fill or Drain direction.

### Improvements

- **Lighter Casting Updates**: The castbar now listens only for your own casting events, reducing background work during play.

### Bug Fixes

- **Cooldown Display Stability**: Back-end fixes to make the cooldown display less likely to stop working partway through a dungeon or raid.
- **Cooldown Group Positioning**: Back-end improvements to cooldown group icon placement, to help reduce icons doubling up, overlapping, or leaving stray empty gaps after talent changes, when opening the options panel, or on login.
- **Castbar No Longer Lingers After a Failed Cast**: The castbar now correctly clears when a cast is rejected, queued, or fails instead of staying on screen.

## 3.7.2.a

### New Features

- **Castbar**: A brand-new player cast bar, with per-cast-type profiles, an optional Auto Share toggle so one cast type's look carries across the others, full support for empowered spells (proper stage segments and timing), and threshold-based color changes. Big thanks to Sadraii, who created the original cast bar module this was expanded from.
- **Dynamic Cooldowns**: A new per-group option that compacts your cooldown icons the same way Dynamic Auras does: icons drop out and the rest slide together based on whether they're ready or on cooldown. Works hand-in-hand with Dynamic Auras.
- **Smooth Movement**: When a dynamic group rearranges, icons now glide smoothly into their new spot instead of snapping, with an adjustable speed. Opt-in per group.
- **Icon Order: First Come, First Served**: Choose how a dynamic group orders its icons: classic Priority order, or First Come First Served, where the icon that became active first keeps its spot and new ones line up after it instead of everything reshuffling.
- **Custom Icon Stacks: Start Full & Recharge**: Custom timer icons can now show full stacks from the start before the first cast, plus a new "Timer Complete" generator with "Recharge until full" to build charge-style stack behavior.
- **What's New Window**: ArcUI now shows a changelog after each update so you always know what changed. Toggle it off in Settings.

### Improvements

- **Bar Performance**: The buff/debuff/stack bar tracking engine was rebuilt from the ground up for smoother updates and noticeably lower CPU use, especially when tracking lots of auras at once.
- **Lower CPU Spikes**: Big reductions in the CPU hitch when leaving combat and when players join your party or raid.
- **Cleaner Custom Icon Options**: The Custom Icons (timer) settings panel now only shows options that actually apply to timers, with the Active / Not Active states behaving correctly and "Hide at 0" working properly for stacks.
- **Totem Dynamic Placement**: Empty totem slots now collapse and compact with Dynamic Auras, keeping your totem icons tidy.

### Bug Fixes

- **Reverse Swipe While Aura Active**: Fixed the swipe reverting to its normal direction when you left combat while the aura was still active; it now stays reversed for the full duration.
- **Charge Spell Placement**: Fixed dynamic placement sometimes failing on charge spells, where an icon wouldn't collapse or return as a charge was spent or came back.
- **Hide CDM Icon staying hidden**: Fixed the Blizzard cooldown frame coming back when a bar had "Hide CDM Icon" turned on, after logging in or reloading, when entering or leaving combat, and when opening the options panel. It now stays hidden at all times, including free-floating icons.
