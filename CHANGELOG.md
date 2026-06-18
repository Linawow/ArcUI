# ArcUI — Changelog

## 3.7.2
### New Features
- **Castbar** — A brand-new player cast bar, with per-cast-type profiles, an optional Auto Share toggle so one cast type's look carries across the others, full support for empowered spells (proper stage segments and timing), and threshold-based color changes. Big thanks to Sadraii, who created the original cast bar module this was expanded from.
- **Dynamic Cooldowns** — A new per-group option that compacts your cooldown icons the same way Dynamic Auras does: icons drop out and the rest slide together based on whether they're ready or on cooldown. Works hand-in-hand with Dynamic Auras.
- **Smooth Movement** — When a dynamic group rearranges, icons now glide smoothly into their new spot instead of snapping, with an adjustable speed. Opt-in per group.
- **Icon Order — First Come, First Served** — Choose how a dynamic group orders its icons: classic Priority order, or First Come First Served, where the icon that became active first keeps its spot and new ones line up after it instead of everything reshuffling.
- **Custom Icon Stacks — Start Full & Recharge** — Custom timer icons can now show full stacks from the start before the first cast, plus a new "Timer Complete" generator with "Recharge until full" to build charge-style stack behavior.
- **What's New Window** — ArcUI now shows a changelog after each update so you always know what changed. Toggle it off in Settings.

### Improvements
- **Bar Performance** — The buff/debuff/stack bar tracking engine was rebuilt from the ground up for smoother updates and noticeably lower CPU use, especially when tracking lots of auras at once.
- **Lower CPU Spikes** — Big reductions in the CPU hitch when leaving combat and when players join your party or raid.
- **Cleaner Custom Icon Options** — The Custom Icons (timer) settings panel now only shows options that actually apply to timers, with the Active / Not Active states behaving correctly and "Hide at 0" working properly for stacks.
- **Totem Dynamic Placement** — Empty totem slots now collapse and compact with Dynamic Auras, keeping your totem icons tidy.

### Bug Fixes
- **Reverse Swipe While Aura Active** — Fixed the swipe reverting to its normal direction when you left combat while the aura was still active; it now stays reversed for the full duration.
- **Charge Spell Placement** — Fixed dynamic placement sometimes failing on charge spells, where an icon wouldn't collapse or return as a charge was spent or came back.

## 3.7.1
### New Features
- **Totem Slots** — Track each of your totem slots as its own icon showing whatever currently occupies it and its remaining duration — totems, ground effects, guardians, pets, and more. Turn it on in the Arc Auras panel; a centered "Totems" group is created for you, and you can drag any slot into another group.
- **Duration Override** — On a cooldown icon, show a totem's remaining time, or a custom duration you set, when you cast the spell — then it switches back to the real cooldown when it ends. Works on CD Manager icons and your Arc spell and trinket icons. Off by default; found in a cooldown icon's options.

### Improvements
- **Custom Icons** — The glow now shows while a custom timer is active, and custom icons stay full color instead of graying out. Added the full set of glow options (type, color, scale, speed, offsets, and more) with a live preview button.
- **Custom Icons** — Added a Desaturate toggle for both the Active and Not Active states, so you control exactly when an icon grays out.
- **Cooldown Reminder** — You can now trigger a reminder a set number of seconds after you cast a spell, or after it becomes ready. The timing is now a direct number entry instead of a slider.

### Bug Fixes
- **Custom Icons** — Fixed the Active / Not Active state flickering, especially on timers that track stacks.
- **Cooldown Reminder** — Fixed the lag and CPU spike that happened while typing the seconds value.
- **Totems** — Empty totem slots now stay hidden, and slots you turn off are remembered per specialization.
