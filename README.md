# Omarchy Kraken Elite

> Local adaptation of [k61n/omarchy-kraken](https://github.com/k61n/omarchy-kraken) for the
> **NZXT Kraken 2023 Elite** (USB `1e71:300c`, 640×640 LCD). Plugin id `local.kraken-elite`,
> state in `~/.local/state/omarchy-kraken-elite`. liquidctl lists this model as "(broken)";
> speed curves use the same protocol as the Kraken Z, LCD uploads are less proven.

An Omarchy bar widget for the NZXT Kraken 2023 Elite, built on
[liquidctl](https://github.com/liquidctl/liquidctl).

- Fan icon in the bar. Click it for live liquid, CPU and GPU temperatures and fan speeds.
- Cooling profiles: Silent, Auto, Performance.
- Show live CPU, GPU and liquid temperatures on the cooler's display (the default), or the
  firmware's own liquid-temperature screen. Static images still work from the command line;
  GIFs don't on 2.x firmware.
- Saved settings are reapplied at login and after resume.

## Install

Install the dependencies:

```bash
sudo pacman -S liquidctl python-pillow
```

Then link this checkout into the plugin folder and enable it:

```bash
ln -sfn "$PWD" ~/.config/omarchy/plugins/local.kraken-elite
omarchy plugin enable local.kraken-elite
```

> [!WARNING]
> Enabling the plugin applies its saved settings straight away, replacing the cooler's
> current curve and display. On a new install, that's the Auto profile and the system info
> display. Removing the plugin doesn't bring the previous curve back.

Update by pulling into this checkout, then run `omarchy restart shell`.

## Remove

Use **Remove Plugin** in the menu, or `omarchy plugin remove local.kraken-elite`. The cooler keeps
its current curve until power-off and its display keeps the current image, and saved settings
and image copies stay in `~/.local/state/omarchy-kraken-elite`.

To remove everything, reset the display first:

```bash
liquidctl --match "Kraken 2023 Elite" set lcd screen liquid
omarchy plugin remove local.kraken-elite
rm -rf ~/.local/state/omarchy-kraken-elite
rm -f "$XDG_RUNTIME_DIR/omarchy-kraken-elite-restored"
```

## Development

Replace the installed copy with a link to your checkout. Saving a file reloads the plugin;
if a change doesn't apply, run `omarchy restart shell`.

```bash
ln -sfn "$PWD" ~/.config/omarchy/plugins/local.kraken-elite
omarchy plugin validate "$PWD"
```

## Profiles

Curves live in `profiles.json` as `[liquid °C, duty %]` points. The cooler's firmware
runs the curve itself, so no daemon has to keep polling. Curves follow **liquid**
temperature, not CPU temperature, so they react gradually to load. Above 59 °C
liquidctl forces 100%. After editing a curve, reapply it with `bin/krakenctl restore`.

## Command line

```bash
bin/krakenctl status                # sensors and saved settings as JSON
bin/krakenctl profile silent        # silent | auto | performance
bin/krakenctl screen info           # live CPU/GPU/liquid screen
bin/krakenctl screen pick           # open the file chooser
bin/krakenctl screen set ~/cat.gif
bin/krakenctl screen liquid
bin/krakenctl restore
```

## System info screen

The cooler's firmware can only draw liquid temperature itself, so the info screen is drawn
by `krakenctl` and uploaded as a 640×640 image (about 0.7 s per upload). The background
service checks every 2 s and uploads only when a shown value changes: at most every 10 s
for a change of 2 °C or more, and at most once a minute for 1 °C wobble. Tune this with
the `INFO_*` constants in `bin/krakenctl`.

Two layouts, chosen with `INFO_LAYOUT`: `omarchy` (default) draws the Omarchy logo mark
(from omarchy.org/brand) with the CPU temperature in its centre; `gauge` shows one large
CPU number. Both have a ring around the edge that fills and changes colour with CPU temperature. Whether the cooler stores uploads in RAM or flash
isn't documented, which is why updates are kept sparse.

## Notes

- Static images are center-cropped to 640×640 and copied to `~/.local/state/omarchy-kraken-elite/images`,
  so restores keep working if the original file moves. GIFs can be up to 24 MB after resizing.
- Settings are applied through liquidctl's `--direct-access`, because the kernel
  driver's hwmon files are only writable by root. Sensors are read from hwmon, except
  NVIDIA GPU temperature, which comes from `nvidia-smi`.
- Settings are restored by a background service inside the Omarchy shell, so they come
  back after you log in. Until then, the cooler uses whatever it last had or its firmware
  default.
- Fan and pump percentages are of rated top speed: 2800 rpm for the pump and 2000 rpm for
  the fan (display only). If your fans are rated differently, change `FAN_MAX_RPM` in
  `bin/krakenctl`.
- GIFs at 640×640 are four times larger than on the Z series, so long ones hit the 24 MB limit sooner.
