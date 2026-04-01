# Home Assistant Configuration

Personal Home Assistant setup tracked as code.

## Structure

- `ha-config/` — Areas, devices, automations, dashboards, helpers, scripts
- `ha-mcp/` — [ha-mcp](https://github.com/homeassistant-ai/ha-mcp) (git submodule)

## Setup

Clone with submodules:

```bash
git clone --recurse-submodules <repo-url>
```

If you already cloned without `--recurse-submodules`:

```bash
git submodule update --init
```

## Updating ha-mcp

Pull the latest upstream changes:

```bash
cd ha-mcp
git pull origin master
cd ..
git add ha-mcp
git commit -m "chore: update ha-mcp submodule"
```

Or as a one-liner:

```bash
git submodule update --remote ha-mcp && git add ha-mcp && git commit -m "chore: update ha-mcp submodule"
```
