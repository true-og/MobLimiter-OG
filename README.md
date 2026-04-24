MobLimiter-OG
=============

TrueOG Network fork of [MobLimiter](https://github.com/NerdNu/MobLimiter). Ported to Gradle 8.14.3 / Purpur API 1.19.4 with additional population controls and admin visibility.

Changes over upstream
---------------------

* **Gradle 8.14.3 build** targeting Purpur API `1.19.4-R0.1-SNAPSHOT`. Uses [Template-OG](https://github.com/true-og/Template-OG) tooling (Spotless, Checkstyle, Shadow, Eclipse integration).
* **Tamed mob limiting** — new `limit_tamed` config flag (default `false`). When `true`, tamed mobs are no longer exempt from age / spawn / chunk-unload limiters.
* **Elder guardian limiting** — new `limit_elder_guardian` config flag (default `false`). When `true`, elder guardians are no longer exempt.
* **Vanilla persistence respect** — new `respect_persistence` flag (default `true`). Honors vanilla `PersistenceRequired` so spawn-egg, `/summon`, and trader-llama mobs are exempt from limiters, matching vanilla "never despawn" semantics. Natural-spawn mobs (spawners, mob spawning) are not persistent and remain subject to limits.
* **Armed-mob culling** — new `limit_armed` flag (default `true`). Mobs carrying armor, a main-hand item, or an off-hand item are culled instead of protected. Closes the abuse path where players drop gear to mobs to keep them loaded indefinitely (vanilla sets `PersistenceRequired` on pickup). When `true` this overrides `respect_persistence` for armed mobs. Named / tamed / elder guardian exemptions still apply.
* **Enchanted-item age extension** — new `enchanted_age_multiplier` (default `2.0`). Mobs holding or wearing any enchanted item get their configured `age` limit multiplied by this value before culling. `1.0` disables the feature; `age: -1` mobs are unaffected.
* **Debug-gated despawn console log** — every plugin-initiated despawn logs to the server console when `debug` is on, including entity type, reason (`age limit`, `chunk unload cull`, `spawn cap radius`, `spawn cap chunk`), world, and block coordinates. (Upstream's separate `despawn_log_console` option was removed in favor of gating on `debug`.)
* **Despawn player notifications** — new `despawn_notify_players` flag (default `false`) and matching permission `moblimiter.notify` (default `op`). When enabled, every despawn is broadcast to online players holding the permission.
* **Configurable notify message** — `despawn_notify_message` template with `&`-color codes and placeholders `%mob%`, `%reason%`, `%world%`, `%x%`, `%y%`, `%z%`.

Version 2 of MobLimiter, featuring three configurable Limiter engines that control the mob population in different ways.

While the original version of MobLimiter was primarily designed to cull mobs on chunk unload, with limited support to
prevent new spawns in real time, MobLimiter 2 is designed from the ground up to be more proactive about managing the
spawning and removal of entities. The Limiters are:

* **Age:** Mobs can be configured to have a maximum lifespan (in ticks) that results in the entity being killed and
dropping its items after the limit is reached. ("Special" mobs are exempt.) MobLimiter will try to not kill breeding
pairs of farm animals if it can, by not killing farm animals if there would be less than two in the chunk.

* **Spawning:** MobLimiter can be configured to limit the spawning of new mobs in real time, checking the number of the
applicable mob type in a "view distance" (as a chunk radius) as well as in an individual chunk and blocking the addition
of extra mobs beyond the limit.

* **Entity Unload:** Similar to version 2.0, the new entity-unload system checks for entities unloading within chunks and culls them down to numbers specified in the config, if desired.

MobLimiter also offers spawner modification protection to prevent unwanted changes to the mobs a spawner produces, as well as mob age locking for when you want to keep your baby a baby forever.

Configuration
-------------

### General Settings

* `radius`: The "view distance" to check for mobs, as a chunk radius (e.g. 3 would be a 7x7 area)
* `breeding_ticks`: Farm animal breeding cooldown in ticks (-1 to disable)
* `growth_ticks`: Ticks for a farm animal to grow up (-1 to disable)
* `logblock`: Enable LogBlock support. More below.
* `debug`: Print debugging info to console
* `limit_tamed`: If `true`, tamed mobs are subject to all limiters. Default `false`.
* `limit_elder_guardian`: If `true`, elder guardians are subject to all limiters. Default `false`.
* `respect_persistence`: If `true`, honor vanilla `PersistenceRequired` (spawn eggs, `/summon`, trader llamas). Default `true`.
* `limit_armed`: If `true`, cull mobs carrying armor or hand items; overrides `respect_persistence` for armed mobs. Default `true`.
* `enchanted_age_multiplier`: Multiplier applied to a mob's configured `age` when it is holding or wearing an enchanted item. Default `2.0`.
* `despawn_notify_players`: If `true`, broadcast every despawn to online players with `moblimiter.notify`. Default `false`.
* `despawn_notify_message`: Template for the notification. Supports `&`-color codes and placeholders `%mob%`, `%reason%`, `%world%`, `%x%`, `%y%`, `%z%`.


### Default Limits

The `defaults` block defines limits that will globally apply to any mob type that doesn't have an explicit override
defined in the `limits` block. (Undefined values fall back to `-1`, for disabled.) Specific mob limits *inherit* the
default block, with any defined fields overriding the value from `defaults`.

```
defaults:
  age: 18000 #15 minutes in ticks
  max: 200 #200 in "view distance"
  chunk_max: 50 #50 in a single chunk
  cull: 4 #cull mobs down to this maximum on chunk unload
```

* `age`: Enable age limiting and remove the mob after a number of ticks. (e.g. 18000 for 15 minutes)
* `max`: The maximum number of a mob type to be allowed to spawn in a "view distance" defined by `radius`.
* `chunk_max`: The maximum number of a mob type to be allowed to spawn in a single chunk.
* `cull`: If set to a value other than `-1`, the number of mobs to *not* be removed on chunk unload.


### Individual Mob Limits

The `limits` block allows you to specify limits that apply to individual mob types. These inherit the values defined in
`defaults`, overriding the values.

Mob types are named using their Bukkit EntityType string, with the exception of sheep, which are addressed in the form of `sheep_white` or `sheep_red` so they can be handled individually for farming purposes.

```
limits:
  skeleton:
    max: 100
    chunkMax: 30
    age: 12000
  cow:
    chunkMax: 75
    age: 12000
  horse:
    age: -1
  villager:
    max: 200
    chunkMax: 50
    age: -1
```


### Farm Animal Breeding Tweaks

The `breeding_ticks` and `growth_ticks` fields define how many ticks a farm animal will remain a baby and the breeding 
cooldown, respectively. If your server is running at a full 20 ticks per second, a value of 400 for each would make
the respective values approximately 20 seconds.

If the value is set to zero, there will be no delay and the condition will be instantaneous. A value of -1 will disable
tampering with vanilla breeding behavior.

This function only affects farm animals, and ignores other breedable entities like ocelots, wolves and villagers.

### Prevent Spawner Modification

MobLimiter blocks the modification of spawners through the use of spawn eggs. You can allow a player to modify them by
granting the permission `moblimiter.spawners.bypass`. You can also customize which spawn eggs are blocked in the config
under the `spawn_eggs` section.

### Mob Age Locking

You're able to lock the age of passive baby mobs by naming them with a nametag. These babies will never grow up, even
if fed with food. If one of these mobs is killed, a log will be made with the person who killed it, the type of mob,
location, and any features specific to that mob (coat colour, owner, etc.).

### Special Mobs

MobLimiter will not remove any mobs that are deemed to be "special" in some way that may make their removal undesirable.

The criteria include:

* Mobs with custom names, such as from a name tag

* Tamed mobs *(unless `limit_tamed: true`)*

* Elder guardians. (Regular guardians can be limited, but Elder ones won't be touched unless `limit_elder_guardian: true`.)

* Mobs wearing armor or holding items in their main or off hand *(unless `limit_armed: true`, the default; players would otherwise abuse this by dropping gear to mobs to keep them loaded)*. Mobs holding or wearing any enchanted item still get culled under `limit_armed: true`, but receive an extended age threshold via `enchanted_age_multiplier` (default `2.0` = double the configured age).

* Mobs with vanilla `PersistenceRequired` set, including spawn-egg placements, `/summon` commands, and trader llamas *(unless `respect_persistence: false`)*. Armed mobs are still culled when `limit_armed: true`, since picking up an item also sets `PersistenceRequired` in vanilla and would otherwise reopen the abuse path.


### LogBlock Integration

If LogBlock is running on the server, you can enable LogBlock integration by setting the `logblock` field to true in the
 config file. When enabled, mob removals will be tracked as kills in LogBlock when MobLimiter performs a chunk unload 
 cull or age limit kill.

Age limit kills are logged with a weapon of `watch` and chunk unload culling uses `gold sword`, both using a "player" 
name of `MobLimiter`.


### Commands

* `/moblimiter` — Lists all subcommands. Available to all users.

* `/moblimiter help` — Prints a description of what MobLimiter does. Available to all users.

* `/moblimiter reload` — Reload the plugin configuration. Requires `moblimiter.reload`.

* `/moblimiter count` — Count all living entities in your chunk and view radius. Requires `moblimiter.count`.

* `/moblimiter limits` — Print all configured limits. Requires `moblimiter.limits`.

* `/moblimiter check` — Inspect the mob you're looking at, printing its age, limits and statuses. Requires `moblimiter.check`.

All commands can be accessed with the `moblimiter.*` permission node.

### Permissions

* `moblimiter.reload` — `/moblimiter reload`
* `moblimiter.count` — `/moblimiter count`
* `moblimiter.limits` — `/moblimiter limits`
* `moblimiter.check` — `/moblimiter check`
* `moblimiter.spawners.bypass` — Modify spawners with spawn eggs.
* `moblimiter.notify` — Receive in-game despawn notifications when `despawn_notify_players` is enabled.
* `moblimiter.*` — All of the above.

