# AGENTS.md — ha-blueprints

This repo holds Home Assistant automation blueprints. One YAML file = one blueprint. Below are the conventions and the non-obvious gotchas worth knowing before editing.

## File conventions

- One blueprint per file, snake_case named after the intent (e.g. `staged_power_on_toggle.yaml`).
- `blueprint.source_url:` points at the raw GitHub path on `main`.
- Top-level key order: `blueprint:` → `mode:` (+ `max_exceeded:`) → `trigger_variables:` (only if needed) → `variables:` → `trigger:` → `condition:` → `action:`.

## Selectors cheat sheet

- `entity` — `domain:` (string or list), `multiple: true` for N-of.
- `number` — `min` / `max` / `step` / `unit_of_measurement` / `mode: box | slider`.
- `duration: {}` for H/M/S input.
- `boolean:`.
- `select:` with `options: [{label, value}]`.

## Triggers — the gotchas

These cost five iterations on `auto_off_below_power_threshold.yaml` before they were understood. Read them before writing a trigger.

### Rule 1 — template-trigger entity tracking

A `template` trigger only fires when one of its *recognized* entities changes. HA recognizes only specific Jinja forms:

- `states('sensor.x')`, `is_state('sensor.x', ...)`, `state_attr('sensor.x', ...)` — literal entity ID
- `expand(<list>)` where `<list>` is statically known at automation load (an `!input` resolved via `trigger_variables:`, not a runtime expression)

Forms that **do not** register subscriptions (the trigger just sits silent):

- `<list> | map('states')` over a runtime list
- `states.<domain>` iteration (works, but subscribes to *every* entity in that domain — noisy and discouraged)
- Entity IDs built by string concatenation at runtime

### Rule 2 — `numeric_state` value_template is per-entity

With `entity_id: [a, b, c]` and a `value_template:`, HA evaluates the template *once per entity that just changed* and compares the result to `above:` / `below:`. The template is **not** an aggregate over all listed entities. Don't use it for sums.

### Rule 3 — `variables:` are not in scope inside trigger templates

`variables:` are evaluated *after* the trigger fires (for conditions and actions). Trigger templates only see `trigger_variables:`. `trigger_variables:` is evaluated at automation load time with limited templating — list-valued `!input` references work fine.

### Rule 4 — edge-triggered, not level-triggered

Both `template` and `numeric_state` fire only on transitions (false→true, or value crossing threshold). They do **not** fire if the condition is already true at automation load / HA restart. Plan for this — e.g. add a second trigger that re-arms on related state changes, or document the limitation.

## Canonical pattern: aggregate across N entities

```yaml
trigger_variables:
  entities: !input entities
  threshold: !input threshold

trigger:
  - platform: template
    value_template: >
      {{ expand(entities) | map(attribute='state') | map('float', 0) | sum
         < (threshold | float) }}
    for: !input below_for
```

Why this works: `expand(entities)` is a parser-recognized form; combined with `trigger_variables:`, HA can extract the IDs at load time and subscribe properly. `| map('float', 0)` makes unavailable sensors contribute 0 instead of raising.

## Optional entity input pattern

```yaml
enable_entity:
  name: Enable entity (optional)
  default:
  selector:
    entity:
      domain: [input_boolean, switch]
```

Empty `default:` makes the input optional. Gate it in a condition:

```yaml
- condition: template
  value_template: >
    {{ enable_entity in [none, ''] or is_state(enable_entity, 'on') }}
```

## Cross-domain on/off

`homeassistant.turn_on` / `homeassistant.turn_off` dispatch correctly across `switch`, `input_boolean`, `light`, `fan`, `media_player`, etc. Prefer these when targets span multiple domains.

## `mode:` picks

- `single` + `max_exceeded: silent` — one-shot actions where re-trigger during execution should be ignored.
- `restart` — "latest intent wins" (e.g. staged power-on interrupted by a new toggle).
- `queued` — rarely needed; only when every trigger must produce a run in order.

## Local YAML lint

`!input` is HA-specific so plain `yaml.safe_load` chokes. Use:

```python
import yaml
class L(yaml.SafeLoader): pass
L.add_constructor('!input', lambda l, n: l.construct_scalar(n) if isinstance(n, yaml.ScalarNode) else l.construct_mapping(n))
yaml.load(open('blueprint.yaml'), Loader=L)
```

## Verification workflow

No test harness; verification is manual on a real HA instance.

1. Local YAML lint (above).
2. Push to a feature branch; the raw GitHub URL is what goes in `source_url:` and what users paste into HA's importer.
3. In HA: **Settings → Automations & Scenes → Blueprints → Import Blueprint** (or **Reload** after editing a previously-imported one).
4. Create an automation from the blueprint, reproduce the intended trigger, and inspect the **trace** UI to confirm the trigger fired and conditions evaluated as expected.

## Git workflow

- Develop on a feature branch (e.g. `claude/<topic>`); land via PR to `main`.
- Commit messages describe *why*, not *what*.
