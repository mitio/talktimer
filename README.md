# Talk Timer

No-frills talk timer in a single HTML file.

## Installation

If you have internet, just open <https://ddimitrov.name/talktimer/>.

For offline usage, clone the repo or get the files somehow and open `index.html`.

## Usage

Opening the timer initially will show a configuration screen. You can set the duration, an optional
"warning" threshold, whether to show seconds or not and so on.

One the timer is started, you can quickly restart it with the `↺` button, put it in fullscreen with
`↗` or stop it and go back to the configuration screen with `⨉`.

### Background color

The background will be green while there's still time left. When time elapses, the background turns
red and the countdown continues in the negative numbers (e.g. -1 minute, -2 minutes, etc.) If
there's a warning threshold configured, once it's reached, the background will turn from green to a
shade of orange, before turning red at 0.

### Font size

The timer tries to automatically determine the largest font size it can use and will set that. You
can override that with the optional font height setting. If you play with that, make sure you check
how the text looks with different digits on the screen.

### Links to preset timers

You can open the timer with a number of minutes as a parameter and the timer will start directly:

<https://ddimitrov.name/talktimer/?5>

Alternatively, you can get a link to a full configuration from the home screen and share that. The
timer will start immediately.

Here is an example suitable for lightning talks:

<https://ddimitrov.name/talktimer/?time=5&time-in=minutes&warn-at=1&warn-time-in=minutes&hide-seconds=false&font-height-percentage=70>

### License and contributions

This repo is licensed under [the MIT license](LICENSE.md).

All contributions are welcome, provided we keep it simple :)

Good luck with your presentations!
