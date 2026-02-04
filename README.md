
This is just personal dashboard. I develop by helping by AI


🛡️ Smart Camera Dashboard (Advanced Camera Card + Frigate)

  - Optimized Home Assistant camera dashboard
  - Snapshot by default
  - Live only when needed
  - Client-aware (Desktop/Mobile aware)
  - Low network & CPU usage
  - Supports multiple cameras & events timeline

📸 UI Overview (Screenshot Description)
```mathematica
┌────────────────────────────────────────────┐
│ 🟢 Front Gate     🟡 Back Yard            │  ← Status Badges (top)
├────────────────────────────────────────────┤
│                                            │
│        📸 Snapshot / 🎥 Live View         │
│        (advanced-camera-card)              │
│                                            │
├────────────────────────────────────────────┤
│  Timeline: Events from all cameras         │
│  - Front Gate: Person detected             │
│  - Back Yard: Motion                       │
└────────────────────────────────────────────┘
```


Behavior

  - Default: snapshot (no live stream)
  - Person detected: live view (max 60s)
  - Click badge: switch active live camera
  - Timeline shows events from all cameras
  - Live stream opens only on active desktop client

🧩 Requirements

  - Home Assistant
  - Frigate NVR
  - go2rtc
  - advanced-camera-card (via HACS)
  - Browser Mod (via HACS)
  - button-card (for badges)

🧠 Architecture Concept

  - One main camera card
  - Multiple cameras
  - Status badges per camera
  - Live streaming is client-aware
  - No global automation forcing live

1️⃣ Helpers
```yaml
input_select:
  camera_active:
    name: Active Camera
    options:
      - none
      - front_gate
      - back_yard

input_boolean:
  camera_live:
    name: Camera Live Mode

timer:
  camera_live_timer:
    duration: "00:01:00"
```

2️⃣ Status Badges (Top of Card)
```yaml
type: horizontal-stack
cards:
  - type: custom:button-card
    name: Front Gate
    icon: mdi:cctv
    entity: binary_sensor.front_gate_person
    state:
      - value: "off"
        color: green
      - value: "on"
        color: orange
    tap_action:
      action: call-service
      service: input_select.select_option
      data:
        entity_id: input_select.camera_active
        option: front_gate

  - type: custom:button-card
    name: Back Yard
    icon: mdi:cctv
    entity: binary_sensor.back_yard_person
    state:
      - value: "off"
        color: green
      - value: "on"
        color: orange
    tap_action:
      action: call-service
      service: input_select.select_option
      data:
        entity_id: input_select.camera_active
        option: back_yard
```

3️⃣ Advanced Camera Card (Main View)
```yaml
type: custom:advanced-camera-card
title: Smart Cameras

cameras:
  - camera_entity: camera.front_gate
    name: Front Gate
    live_provider: go2rtc
    priority: 1
    frigate:
      url: http://frigate:5000
    triggers:
      entities:
        - binary_sensor.front_gate_person

  - camera_entity: camera.back_yard
    name: Back Yard
    live_provider: go2rtc
    priority: 2
    frigate:
      url: http://frigate:5000
    triggers:
      entities:
        - binary_sensor.back_yard_person

view:
  default: image

  # Render live ONLY on active desktop browser
  triggers:
    actions:
      trigger: live
      untrigger: image
    render_entities:
      - binary_sensor.browser_desktop_active
      - input_boolean.camera_live

image:
  mode: snapshot
  refresh_seconds: 300

live:
  preload: false
  lazy_unload: true
  max_seconds: 60

timeline:
  enabled: true
  events:
    cameras: all
    media: clips
    limit: 10
```

4️⃣ Automations
Person Detection → Start Live
```yaml
automation:
  - alias: Camera Person Detected
    trigger:
      - platform: state
        entity_id:
          - binary_sensor.front_gate_person
          - binary_sensor.back_yard_person
        to: "on"
    action:
      - service: input_boolean.turn_on
        target:
          entity_id: input_boolean.camera_live
      - service: input_select.select_option
        data:
          entity_id: input_select.camera_active
          option: >
            {% if 'front_gate' in trigger.entity_id %}
              front_gate
            {% else %}
              back_yard
            {% endif %}
```
Live Timer (60s Cooldown)
```yaml
automation:
  - alias: Camera Live Timer Start
    trigger:
      - platform: state
        entity_id: input_boolean.camera_live
        to: "on"
    action:
      - service: timer.start
        target:
          entity_id: timer.camera_live_timer

  - alias: Camera Live Timer End
    trigger:
      - platform: event
        event_type: timer.finished
        event_data:
          entity_id: timer.camera_live_timer
    action:
      - service: input_boolean.turn_off
        target:
          entity_id: input_boolean.camera_live
```

5️⃣ Browser Mod (Client Awareness)
Required Entities

Example (names depend on your setup):

  - binary_sensor.browser_desktop_active
  - sensor.browser_desktop_device_type

Only active desktop clients will render live streams.

Mobile clients:
  - Snapshot only
  - No auto live
  - Tap-only if enabled manually

6️⃣ Events from Other Cameras (Timeline)

This is handled natively by advanced-camera-card.
```yaml
timeline:
  enabled: true
  events:
    cameras: all
```
Result

 - Current camera stays visible
 - Events from other cameras appear in timeline
 - No forced camera switching
 - No extra live streams

✅ Final Result

✔ Snapshot by default (near-zero bandwidth)

✔ Live only on person detection

✔ Live limited to 60 seconds

✔ Badge click switches live camera

✔ Timeline shows events from all cameras

✔ Live stream opens only on active desktop client

✔ Mobile clients stay lightweight

✔ Safe for multi-dashboard / wall displays

🏁 Notes

This setup is suitable for:

 - Smart homes
 - Wall-mounted dashboards
 - Low-power servers
 - Multi-client environments
 - Designed to scale beyond 2 cameras

📎 References

https://card.camera

https://github.com/dermotduffy/advanced-camera-card

https://github.com/thomasloven/hass-browser_mod

https://docs.frigate.video
