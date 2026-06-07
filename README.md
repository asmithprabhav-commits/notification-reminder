
  python "pop up notifier.py" -m "Water the plants" -t "Reminder"
  python "pop up notifier.py" -m "Stretch" -I 60    # repeat every 60 minutes
"""
from __future__ import annotations

import argparse
import logging
import time
from pathlib import Path
from typing import Optional

from plyer import notification


def send_notification(title: str, message: str, app_icon: Optional[str] = None, timeout: int = 10) -> None:
    """Send a system notification using plyer."""
    notification.notify(title=title, message=message, app_icon=app_icon, timeout=timeout)


def parse_args() -> argparse.Namespace:
    p = argparse.ArgumentParser(description="Popup notifier (plyer)")
    p.add_argument("-t", "--title", default="Reminder", help="Notification title")
    p.add_argument("-m", "--message", required=True, help="Notification message")
    p.add_argument("-n", "--name", default=None, help="Your name for personalization")
    p.add_argument("--tone", choices=("friendly", "cheerful", "formal"), default="friendly", help="Tone of the message")
    p.add_argument("--category", choices=("default", "plants", "health", "break"), default="default", help="Category to pick emoji/templates")
    p.add_argument("--emoji", action="store_true", help="Include an emoji matching the category")
    p.add_argument("--snooze", type=int, default=0, help="If >0, prompt to snooze this notification (minutes)")
    p.add_argument("-i", "--icon", default=None, help="Path to an icon file (.ico or .png)")
    p.add_argument("-o", "--timeout", type=int, default=10, help="Notification timeout in seconds")
    p.add_argument("-I", "--interval", type=float, default=0.0, help="Repeat interval in minutes (0 = once)")
    p.add_argument("-c", "--count", type=int, default=0, help="Number of times to repeat (0 = infinite when interval>0)")
    return p.parse_args()


def main() -> None:
    logging.basicConfig(level=logging.INFO, format="%(asctime)s %(levelname)s: %(message)s")
    args = parse_args()

    icon_path = None
    if args.icon:
        p = Path(args.icon)
        if p.exists():
            icon_path = str(p)
        else:
            logging.warning("Icon path not found: %s (ignoring)", args.icon)

    def build_humanized(title: str, message: str, tone: str, name: Optional[str], category: str, emoji: bool) -> tuple[str, str]:
        import random

        emojis = {
            "plants": "🌱",
            "health": "💪",
            "break": "☕",
            "default": "🔔",
        }

        templates = {
            "friendly": [
                ("Hey{who}", "It's time: {msg}"),
                ("Friendly reminder{who}", "Don't forget: {msg}"),
            ],
            "cheerful": [
                ("Good news{who}!", "{msg} — go do it 😄"),
                ("Heads up{who}!", "A quick reminder: {msg}"),
            ],
            "formal": [
                ("Reminder{who}", "This is a reminder: {msg}"),
                ("Notice{who}", "Please attend to: {msg}"),
            ],
        }

        who = f", {name}" if name else ""
        tpl_title, tpl_msg = random.choice(templates.get(tone, templates["friendly"]))
        composed_title = tpl_title.format(who=who)
        composed_msg = tpl_msg.format(msg=message)
        if emoji:
            composed_title = f"{emojis.get(category, emojis['default'])} {composed_title}"
        return composed_title, composed_msg

    try:
        if args.interval and args.interval > 0:
            interval_sec = float(args.interval) * 60.0
            sent = 0
            logging.info("Starting repeating notifications every %.2f minutes", args.interval)
            while True:
                title, message = build_humanized(args.title, args.message, args.tone, args.name, args.category, args.emoji)
                logging.info("Sending notification: %s - %s", title, message)
                send_notification(title, message, app_icon=icon_path, timeout=args.timeout)
                sent += 1
                if args.count > 0 and sent >= args.count:
                    logging.info("Sent %d notifications; exiting", sent)
                    break
                time.sleep(interval_sec)
        else:
            title, message = build_humanized(args.title, args.message, args.tone, args.name, args.category, args.emoji)
            logging.info("Sending one notification: %s - %s", title, message)
            send_notification(title, message, app_icon=icon_path, timeout=args.timeout)

            if args.snooze and args.snooze > 0:
                try:
                    logging.info("Press Enter to snooze for %d minutes, or Ctrl+C to exit", args.snooze)
                    input()
                    logging.info("Snoozing for %d minutes...", args.snooze)
                    time.sleep(args.snooze * 60)
                    title2, message2 = build_humanized(args.title, args.message, args.tone, args.name, args.category, args.emoji)
                    logging.info("Sending snoozed notification: %s - %s", title2, message2)
                    send_notification(title2, message2, app_icon=icon_path, timeout=args.timeout)
                except KeyboardInterrupt:
                    logging.info("Snooze canceled by user")
    except KeyboardInterrupt:
        logging.info("Interrupted by user; exiting")


if __name__ == "__main__":
    main()
