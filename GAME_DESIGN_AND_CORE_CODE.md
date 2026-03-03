# Night Terrors (Original Audio-Driven Survival Horror) — Design + Core Implementation (Godot 4 / GDScript)

> **Copyright note:** I can’t help you copy or use copyrighted FNAF 4 assets from third-party sites. This design is intentionally built for **fully original** art/audio/lore while preserving the **high-level gameplay structure** you requested.

## 1) Full Game Design

## Engine Choice
- **Godot 4 + GDScript**
- Reason: fast prototyping, clean state-machine scripting, lightweight audio routing (buses), easy export.

## Core Scenes / Nodes
- `Main.tscn`
  - `GameManager` (autoload/singleton recommended)
  - `TimeOfNight`
  - `PlayerController`
  - `ThreatDirector`
  - `AudioManager`
  - `UIRoot`
- `Bedroom.tscn`
  - Camera anchor points: `View_LeftDoor`, `View_RightDoor`, `View_Closet`, `View_Bed`
  - Interact colliders/zones for each threat point
- `Monsters/`
  - `HallwayStalkerA.tscn`
  - `HallwayStalkerB.tscn`
  - `ClosetCrawler.tscn`
  - `BedShadows.tscn`
  - `PrimeNightmare.tscn`

## Systems and Interactions

### A) Time Progression System
- Night runs from **12:00 AM → 6:00 AM** over fixed real time (default **480 sec = 8 min**).
- Emits events each in-game hour and on completion.
- `ThreatDirector` reads normalized night progress (`0.0..1.0`) to scale aggressiveness.

### B) Player Controller + Interaction
- Player can snap-look among four positions:
  - Left Door
  - Right Door
  - Closet
  - Bed
- Inputs:
  - Move viewpoint (keys or mouse-swipe)
  - Flashlight (tap/hold)
  - Hold door shut (only when at left/right/closet door zone)
- Rules:
  - If breathing at door: **hold door**, **do not flashlight**.
  - If no breathing: quick flashlight to verify/reset threat.
  - Bed shadows cleared by periodic flashlight checks.

### C) Monster AI (State Machine per monster)
- Shared states:
  - `IDLE`, `ADVANCING`, `AT_POINT`, `RETREATING`, `ATTACKING`
- State transitions depend on:
  - Timers + night difficulty
  - Player actions (flashlight, door hold)
  - Audio-informed “decision windows” (e.g., breathing lockout)

### D) Audio Manager (Audio-First)
- No constant music in gameplay loop.
- Uses positional and routed SFX:
  - Left/right footsteps (directional)
  - Breathing at door (proximity-critical)
  - Closet scrapes, bed whisper/chitter
- Dynamic ambience intensity rises as `night_progress` increases.
- Sliders:
  - Master / SFX / Ambient / UI

### E) UI + Menus
- Main menu:
  - New Game, Continue, Night Select, Options
- In-night UI:
  - Clock display (12 AM–6 AM)
  - Subtle danger feedback (vignette pulse, heartbeat amplitude)
- Death/jumpscare overlay with retry.

### F) Night Structure / Difficulty
- **Night 1:** Hallway Stalker A tutorial.
- **Night 2:** Add Hallway Stalker B.
- **Night 3:** Add Closet Crawler.
- **Night 4:** Add Bed Shadows.
- **Night 5:** All active + faster windows.
- **Night 6+ (optional):** Prime Nightmare only (flashlight mostly ineffective).

---

## 2) Core Code (Godot 4 / GDScript)

## 2.1 Time-of-Night System (`scripts/time_of_night.gd`)
```gdscript
extends Node
class_name TimeOfNight

signal hour_changed(hour_index: int, display_text: String)
signal night_completed

@export var real_seconds_per_night: float = 480.0 # 8 minutes default
@export var start_hour: int = 12
@export var end_hour: int = 6

var elapsed: float = 0.0
var running: bool = false
var _last_hour_bucket: int = -1

func start_night(duration_override: float = -1.0) -> void:
	if duration_override > 0.0:
		real_seconds_per_night = duration_override
	elapsed = 0.0
	running = true
	_last_hour_bucket = -1
	_emit_hour_if_needed()

func stop_night() -> void:
	running = false

func _process(delta: float) -> void:
	if not running:
		return

	elapsed += delta
	_emit_hour_if_needed()

	if elapsed >= real_seconds_per_night:
		running = false
		emit_signal("night_completed")

func get_progress_01() -> float:
	return clamp(elapsed / real_seconds_per_night, 0.0, 1.0)

func get_current_display_hour() -> String:
	var bucket := _get_hour_bucket()
	return _bucket_to_clock_text(bucket)

func _emit_hour_if_needed() -> void:
	var bucket := _get_hour_bucket()
	if bucket != _last_hour_bucket:
		_last_hour_bucket = bucket
		emit_signal("hour_changed", bucket, _bucket_to_clock_text(bucket))

func _get_hour_bucket() -> int:
	# 6 segments: 12,1,2,3,4,5 then completion at 6
	var p := get_progress_01()
	return int(floor(p * 6.0)) # 0..6

func _bucket_to_clock_text(bucket: int) -> String:
	if bucket <= 0:
		return "12:00 AM"
	if bucket >= 6:
		return "6:00 AM"
	return "%d:00 AM" % bucket
```

## 2.2 Player Movement + Interaction (`scripts/player_controller.gd`)
```gdscript
extends Node
class_name PlayerController

signal view_changed(new_view: String)
signal flashlight_used(view: String)
signal door_hold_changed(view: String, is_holding: bool)

@export var turn_cooldown: float = 0.15
@export var flashlight_battery_drain_per_sec: float = 0.0 # optional system

enum View { LEFT_DOOR, RIGHT_DOOR, CLOSET, BED }

var current_view: View = View.LEFT_DOOR
var can_turn: bool = true
var holding_door: bool = false

func _ready() -> void:
	_emit_view_changed()

func _unhandled_input(event: InputEvent) -> void:
	if event.is_action_pressed("look_left"):
		_cycle_view(-1)
	elif event.is_action_pressed("look_right"):
		_cycle_view(1)
	elif event.is_action_pressed("look_closet"):
		set_view(View.CLOSET)
	elif event.is_action_pressed("look_bed"):
		set_view(View.BED)

	if event.is_action_pressed("flashlight"):
		emit_signal("flashlight_used", _view_name(current_view))

	if event.is_action_pressed("hold_door"):
		if _can_hold_in_current_view():
			holding_door = true
			emit_signal("door_hold_changed", _view_name(current_view), true)

	if event.is_action_released("hold_door"):
		if holding_door:
			holding_door = false
			emit_signal("door_hold_changed", _view_name(current_view), false)

func set_view(v: View) -> void:
	if not can_turn or v == current_view:
		return
	current_view = v
	_emit_view_changed()
	can_turn = false
	await get_tree().create_timer(turn_cooldown).timeout
	can_turn = true

func _cycle_view(dir: int) -> void:
	var idx := int(current_view) + dir
	if idx < 0:
		idx = 3
	elif idx > 3:
		idx = 0
	set_view(idx)

func _can_hold_in_current_view() -> bool:
	return current_view in [View.LEFT_DOOR, View.RIGHT_DOOR, View.CLOSET]

func _emit_view_changed() -> void:
	emit_signal("view_changed", _view_name(current_view))

func _view_name(v: View) -> String:
	match v:
		View.LEFT_DOOR:
			return "left_door"
		View.RIGHT_DOOR:
			return "right_door"
		View.CLOSET:
			return "closet"
		View.BED:
			return "bed"
		_:
			return "unknown"
```

## 2.3 Monster AI Template (Hallway Stalker A) (`scripts/hallway_stalker_a.gd`)
```gdscript
extends Node
class_name HallwayStalkerA

signal request_jumpscare(monster_name: String)
signal play_audio_cue(cue_name: String, position_tag: String)

# Tunables (exposed for balancing)
@export var base_advance_interval: float = 6.0
@export var min_advance_interval: float = 2.0
@export var breathing_window: float = 2.5
@export var attack_grace_after_wrong_flash: float = 0.3

enum State { IDLE, ADVANCING, AT_DOOR, RETREATING, ATTACKING }
var state: State = State.IDLE

var difficulty_scale: float = 0.0 # set by ThreatDirector (0..1+)
var at_door: bool = false
var player_at_left_door: bool = false
var player_holding_left_door: bool = false

func start() -> void:
	state = State.IDLE
	_schedule_next_advance()

func set_difficulty(scale: float) -> void:
	difficulty_scale = scale

func on_player_view_changed(view: String) -> void:
	player_at_left_door = (view == "left_door")

func on_door_hold_changed(view: String, is_holding: bool) -> void:
	if view == "left_door":
		player_holding_left_door = is_holding
		if state == State.AT_DOOR and is_holding:
			_retreat_from_door()

func on_flashlight_used(view: String) -> void:
	if view != "left_door":
		return

	# Key rule: if breathing is present (monster at door), flashing causes death.
	if state == State.AT_DOOR:
		state = State.ATTACKING
		await get_tree().create_timer(attack_grace_after_wrong_flash).timeout
		emit_signal("request_jumpscare", "HallwayStalkerA")
		return

	# If not at door, flashlight can force a slight delay/retreat cue.
	if state == State.ADVANCING:
		state = State.RETREATING
		emit_signal("play_audio_cue", "hsa_hiss_retreat", "left_hall")
		await get_tree().create_timer(1.0).timeout
		state = State.IDLE
		_schedule_next_advance()

func _schedule_next_advance() -> void:
	var t := lerp(base_advance_interval, min_advance_interval, clamp(difficulty_scale, 0.0, 1.0))
	t += randf_range(-0.75, 0.75)
	await get_tree().create_timer(max(0.5, t)).timeout
	_attempt_advance()

func _attempt_advance() -> void:
	if state == State.ATTACKING:
		return

	state = State.ADVANCING
	emit_signal("play_audio_cue", "hsa_footsteps", "left_hall")

	await get_tree().create_timer(1.0).timeout

	# Chance to arrive at door scales with difficulty.
	var p_arrive := 0.45 + difficulty_scale * 0.35
	if randf() <= p_arrive:
		_enter_door_breathing_phase()
	else:
		state = State.IDLE
		_schedule_next_advance()

func _enter_door_breathing_phase() -> void:
	state = State.AT_DOOR
	at_door = true
	emit_signal("play_audio_cue", "hsa_breathing", "left_door")

	var timer := get_tree().create_timer(breathing_window)
	await timer.timeout

	# If player did not hold door during breathing window, attack.
	if state == State.AT_DOOR and not player_holding_left_door:
		state = State.ATTACKING
		emit_signal("request_jumpscare", "HallwayStalkerA")

func _retreat_from_door() -> void:
	if state != State.AT_DOOR:
		return
	at_door = false
	state = State.RETREATING
	emit_signal("play_audio_cue", "hsa_retreat_steps", "left_hall")
	await get_tree().create_timer(1.2).timeout
	state = State.IDLE
	_schedule_next_advance()
```

---

## 3) Extending Template to Full Cast
- `Hallway Stalker B`:
  - Reuse same script with separate tuning profile (faster late-night spikes, different cue set).
- `Closet Crawler`:
  - Same state machine, but target point is closet occupancy meter.
  - Requires sustained door hold at closet to reset.
- `Bed Shadows`:
  - Continuous accumulation variable instead of hallway approach.
  - Flashlight on bed decrements stacks; stack cap triggers “Large Bed Entity” jumpscare.
- `Prime Nightmare`:
  - Single AI with multi-entry threat routing.
  - Flashlight has no repel effect; only correct door-hold timing + quick repositioning buys time.

## 4) Additional Nights + Difficulty Scaling
- Create `NightConfig` resource per night:
  - monster enabled flags
  - AI difficulty multipliers
  - special rules (e.g., prime-only, flashlight immunity)
- `ThreatDirector` loads config at night start and pushes values into each AI.
- Scale examples:
  - advance interval reduction
  - breathing window shrink
  - attack probability increase
  - fake cues frequency increase (higher nights)

## 5) Recommended Original Asset Naming Convention
Use your own packs with names like:
- `monster_hsa_idle`, `monster_hsa_door`, `sfx_hsa_breathing_left`
- `monster_hsb_*`, `monster_closetcrawler_*`, `monster_bedshadows_*`, `monster_prime_*`
- `ui_clock_digits`, `bg_room_ambient`, `fx_vignette_danger`

This keeps all content original while preserving the intended mechanical tension.
