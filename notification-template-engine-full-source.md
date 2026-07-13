# notification-template-engine — Full Source Code

All code files for the fixed project, consolidated in one document.

---

## Python Modules

### `users.py`

```python
import logging

logging.basicConfig(
    level=logging.INFO,
    format="%(levelname)s: %(message)s"
)


class User:
    """
    Represents a user who receives notifications.
    """

    SUPPORTED_LOCALES = {"en", "hi", "te"}
    DEFAULT_LOCALE = "en"

    def __init__(self, name, email, locale, strict_locale=False):
        """
        Args:
            name: Display name of the user.
            email: Contact email address.
            locale: Preferred language code (e.g. "en", "hi", "te").
            strict_locale: If True, an unsupported locale raises a
                ValueError instead of silently falling back to English.
                Defaults to False to preserve backwards-compatible
                fallback behaviour, but the fallback is now explicit
                and logged instead of happening silently.
        """

        # Validate name
        if not isinstance(name, str) or not name.strip():
            raise ValueError("User name cannot be empty.")

        # Validate email
        if not isinstance(email, str) or "@" not in email:
            raise ValueError("Invalid email address.")

        # Validate locale type
        if not isinstance(locale, str) or not locale.strip():
            raise ValueError("Locale cannot be empty.")

        self.name = name.strip()
        self.email = email.strip()

        locale = locale.strip()

        # Explicit, logged locale validation instead of a silent fallback.
        if locale not in self.SUPPORTED_LOCALES:
            if strict_locale:
                raise ValueError(
                    f"Unsupported locale '{locale}'. "
                    f"Supported locales are: "
                    f"{sorted(self.SUPPORTED_LOCALES)}"
                )

            logging.warning(
                f"Unsupported locale '{locale}' for user '{self.name}'. "
                f"Falling back to default locale "
                f"'{self.DEFAULT_LOCALE}'."
            )
            self.locale = self.DEFAULT_LOCALE
        else:
            self.locale = locale

    def __str__(self):
        return (
            f"User("
            f"name='{self.name}', "
            f"email='{self.email}', "
            f"locale='{self.locale}')"
        )

    def __repr__(self):
        return self.__str__()
```

### `exceptions.py`

```python
class NotificationError(Exception):
    """Base class for all notification-engine specific errors."""


class MissingDataError(NotificationError):
    """
    Raised when data required to render a notification is missing.

    Replaces the previously unhandled KeyError that could bubble up
    from render_template() when a template referenced a placeholder
    that wasn't present in the supplied data.
    """


class TemplateRenderError(NotificationError):
    """Raised when a template cannot be rendered for any other reason."""
```

### `template_loader.py`

```python
import os
import logging
import re

# Configure logging
logging.basicConfig(
    level=logging.INFO,
    format="%(levelname)s: %(message)s"
)

# Base directory templates are loaded from. Resolved to an absolute path
# so every loaded file can be verified to stay inside it.
TEMPLATES_ROOT = os.path.abspath(
    os.path.join(os.path.dirname(__file__), "templates")
)

# Only simple, predictable identifiers are allowed for locale/event names.
# This is what actually prevents path traversal / injection: values are
# validated *before* they ever touch os.path.join, instead of trying to
# sanitize a path after the fact.
_SAFE_IDENTIFIER = re.compile(r"^[a-zA-Z0-9_]+$")

DEFAULT_LOCALE = "en"

# Supported delivery channels and the template file extension each one uses.
CHANNEL_EXTENSIONS = {
    "text": "txt",
    "email": "html",
    "sms": "txt",
}

# Supported placeholders
VALID_PLACEHOLDERS = {
    "{{name}}",
    "{{date}}",
    "{{time}}"
}


class InvalidTemplateError(ValueError):
    """Raised when a template fails validation."""


def _validate_identifier(value, label):
    """
    Ensures locale/event values are simple identifiers before they are
    used to build a filesystem path. Rejects anything containing path
    separators, '..', or other characters that could be used to escape
    the templates directory.
    """

    if not value or not isinstance(value, str):
        raise ValueError(f"Invalid {label}.")

    if not _SAFE_IDENTIFIER.match(value):
        raise ValueError(
            f"Invalid {label} '{value}'. Only letters, numbers and "
            f"underscores are allowed."
        )

    return value


def _resolve_template_path(locale, event, channel):
    """
    Builds and validates the on-disk path for a given locale/event/channel,
    guaranteeing the resolved path stays inside TEMPLATES_ROOT.
    """

    extension = CHANNEL_EXTENSIONS.get(channel)
    if extension is None:
        raise ValueError(
            f"Unsupported channel '{channel}'. Supported channels are: "
            f"{sorted(CHANNEL_EXTENSIONS)}"
        )

    candidate = os.path.abspath(
        os.path.join(TEMPLATES_ROOT, locale, f"{event}.{extension}")
    )

    # Defense in depth: even though locale/event are already restricted to
    # safe identifiers, double check the resolved path never escapes the
    # templates directory.
    if os.path.commonpath([TEMPLATES_ROOT, candidate]) != TEMPLATES_ROOT:
        raise ValueError("Resolved template path is outside templates directory.")

    return candidate


def validate_template(template):
    """
    Validate template content.
    """

    # Empty template
    if not template.strip():
        raise InvalidTemplateError("Template is empty.")

    # Find placeholders
    placeholders = set(
        re.findall(r"\{\{.*?\}\}", template)
    )

    # Invalid placeholder detection
    invalid = placeholders - VALID_PLACEHOLDERS

    if invalid:
        raise InvalidTemplateError(
            f"Invalid placeholders found: {invalid}"
        )

    return True


def load_template(locale, event, channel="text"):
    """
    Loads the notification template for the given locale, event and
    delivery channel ("text", "email" -> HTML, "sms").

    Falls back to English if the locale does not exist. locale and event
    are validated as safe identifiers before being used to build a
    filesystem path, preventing path traversal via crafted values.
    """

    locale = _validate_identifier(locale, "locale")
    event = _validate_identifier(event, "event name")

    file_path = _resolve_template_path(locale, event, channel)

    # Locale fallback
    if not os.path.exists(file_path):

        logging.warning(
            f"Locale '{locale}' not found. Falling back to "
            f"'{DEFAULT_LOCALE}'."
        )

        file_path = _resolve_template_path(DEFAULT_LOCALE, event, channel)

    # Event validation
    if not os.path.exists(file_path):
        raise FileNotFoundError(
            f"Template not found for event '{event}' (channel='{channel}')."
        )

    try:

        with open(
            file_path,
            "r",
            encoding="utf-8"
        ) as file:

            template = file.read()

    except OSError as e:

        logging.error(
            f"Unable to read template: {e}"
        )
        raise

    # Validate template
    validate_template(template)

    logging.info(
        f"Loaded template: {file_path}"
    )

    return template
```

### `notification_service.py`

```python
import html
import logging
import re

from template_loader import load_template
from exceptions import MissingDataError
from channels import ConsoleChannel

logging.basicConfig(
    level=logging.INFO,
    format="%(levelname)s: %(message)s"
)

# Maps a delivery channel name to the template variant it needs and
# whether interpolated values should be HTML-escaped.
_CHANNEL_TEMPLATE_TYPE = {
    "console": ("text", False),
    "sms": ("text", False),
    "email": ("email", True),
}

_PLACEHOLDER_PATTERN = re.compile(r"\{\{\s*(\w+)\s*\}\}")


def render_template(template, values, escape=False):
    """
    Renders a template in a single pass using a proper substitution
    engine (re.sub with a lookup callback) instead of iterating and
    calling str.replace() once per placeholder. This is both more
    efficient (one scan of the template instead of one scan per
    placeholder) and safer, since every placeholder is resolved through
    one consistent code path.

    Raises MissingDataError (instead of an unhandled KeyError) if the
    template references a placeholder that isn't present in `values`.
    """

    missing = []

    def _lookup(match):
        key = match.group(1)

        if key not in values or values[key] is None:
            missing.append(key)
            return match.group(0)

        value = str(values[key])
        return html.escape(value) if escape else value

    rendered = _PLACEHOLDER_PATTERN.sub(_lookup, template)

    if missing:
        raise MissingDataError(
            f"Missing value(s) for placeholder(s): {sorted(set(missing))}"
        )

    return rendered


def send_notification(user, event, data, channel=None):
    """
    Generates a localized notification and, if a delivery channel is
    provided, actually dispatches it (email/SMS/console) instead of
    only ever printing the rendered text.

    Args:
        user: A users.User instance.
        event: Notification event name, e.g. "interview_scheduled".
        data: Dict with the dynamic values for the template
            (currently "date" and "time"; "name" is taken from `user`).
        channel: Optional channels.NotificationChannel instance to
            dispatch through. If omitted, the message is only rendered
            and returned (backwards compatible with earlier behaviour).

    Returns:
        The rendered notification message (str).
    """

    if user is None:
        raise ValueError("User cannot be None.")

    if not isinstance(data, dict):
        raise ValueError("Notification data must be a dictionary.")

    channel_name = channel.name if channel else "console"
    template_type, should_escape = _CHANNEL_TEMPLATE_TYPE.get(
        channel_name, ("text", False)
    )

    template = load_template(
        user.locale,
        event,
        channel=template_type,
    )

    values = {
        "name": user.name,
        "date": data.get("date"),
        "time": data.get("time"),
    }

    if values["date"] is None:
        raise ValueError("Date is required.")

    if values["time"] is None:
        raise ValueError("Time is required.")

    message = render_template(
        template,
        values,
        escape=should_escape,
    )

    logging.info(
        f"Notification generated for {user.name}"
    )

    if channel is not None:
        subject = event.replace("_", " ").title()
        channel.send(user, subject, message)

    return message
```

### `channels.py`

```python
"""
Delivery channels for notifications.

Previously the "notification engine" only ever printed the rendered
message to stdout, despite the project name implying it actually
delivers notifications via email/SMS. This module gives each channel a
real send() implementation:

- EmailChannel sends via SMTP (smtplib) when credentials are configured.
- SMSChannel sends via a pluggable HTTP gateway (e.g. Twilio-style API)
  when credentials are configured.
- Both default to `dry_run=True`, which logs exactly what would be sent
  without requiring real credentials/network access. This keeps the
  project runnable out of the box and unit-testable, while making the
  dispatch path fully real once credentials are supplied.
- ConsoleChannel preserves the original print-to-stdout behaviour for
  local debugging/demo purposes.
"""

import logging
import smtplib
from abc import ABC, abstractmethod
from email.message import EmailMessage

logging.basicConfig(
    level=logging.INFO,
    format="%(levelname)s: %(message)s"
)


class DeliveryError(Exception):
    """Raised when a channel fails to deliver a notification."""


class NotificationChannel(ABC):
    """Base interface every delivery channel must implement."""

    name = "base"

    @abstractmethod
    def send(self, user, subject, message):
        """
        Deliver `message` (already rendered) to `user`.
        Returns True on success, raises DeliveryError on failure.
        """
        raise NotImplementedError


class ConsoleChannel(NotificationChannel):
    """Prints the notification to stdout. Useful for local demos/tests."""

    name = "console"

    def send(self, user, subject, message):
        print(f"\n[{self.name.upper()}] To: {user.email}")
        if subject:
            print(f"Subject: {subject}")
        print(message)
        logging.info(f"Console delivery complete for {user.name}.")
        return True


class EmailChannel(NotificationChannel):
    """
    Sends the notification as an email via SMTP.

    If no SMTP host is configured (or dry_run=True), the channel logs
    what *would* be sent instead of opening a real network connection.
    This keeps the project runnable without real credentials while
    still exercising the full dispatch code path.
    """

    name = "email"

    def __init__(
        self,
        smtp_host=None,
        smtp_port=587,
        username=None,
        password=None,
        sender="no-reply@example.com",
        dry_run=True,
    ):
        self.smtp_host = smtp_host
        self.smtp_port = smtp_port
        self.username = username
        self.password = password
        self.sender = sender
        self.dry_run = dry_run or not smtp_host

    def send(self, user, subject, message):
        if self.dry_run:
            logging.info(
                f"[DRY RUN] Would send EMAIL to {user.email} "
                f"(subject='{subject}')."
            )
            return True

        email_message = EmailMessage()
        email_message["From"] = self.sender
        email_message["To"] = user.email
        email_message["Subject"] = subject or "Notification"
        email_message.set_content(message, subtype="html")

        try:
            with smtplib.SMTP(self.smtp_host, self.smtp_port) as server:
                server.starttls()
                if self.username and self.password:
                    server.login(self.username, self.password)
                server.send_message(email_message)
        except (smtplib.SMTPException, OSError) as e:
            logging.error(f"Failed to send email to {user.email}: {e}")
            raise DeliveryError(f"Email delivery failed: {e}") from e

        logging.info(f"Email sent to {user.email}.")
        return True


class SMSChannel(NotificationChannel):
    """
    Sends the notification via an SMS gateway HTTP API.

    A `send_fn` can be injected for use with any provider's SDK/HTTP
    client (e.g. Twilio, AWS SNS). If none is configured (or
    dry_run=True), the channel logs what would be sent.
    """

    name = "sms"

    def __init__(self, send_fn=None, dry_run=True):
        self.send_fn = send_fn
        self.dry_run = dry_run or send_fn is None

    def send(self, user, subject, message):
        if self.dry_run:
            logging.info(
                f"[DRY RUN] Would send SMS to {user.email} "
                f"(no phone number field on User yet)."
            )
            return True

        try:
            self.send_fn(user, message)
        except Exception as e:
            logging.error(f"Failed to send SMS to {user.name}: {e}")
            raise DeliveryError(f"SMS delivery failed: {e}") from e

        logging.info(f"SMS sent for {user.name}.")
        return True


CHANNEL_REGISTRY = {
    "console": ConsoleChannel,
    "email": EmailChannel,
    "sms": SMSChannel,
}
```

### `main.py`

```python
from users import User
from notification_service import send_notification
from channels import ConsoleChannel, EmailChannel


def main():

    users = [

        User(
            "Vaishnavi",
            "vaish@gmail.com",
            "en"
        ),

        User(
            "Jaya",
            "jaya@gmail.com",
            "hi"
        ),

        User(
            "Anushka",
            "anushka@gmail.com",
            "te"
        )
    ]

    # Different notification events
    events = [

        "interview_scheduled",
        "interview_reminder",
        "interview_cancelled",
        "interview_completed"

    ]

    data = {

        "date": "10 July",
        "time": "5 PM"

    }

    # ConsoleChannel prints notifications, matching the previous
    # print-only behaviour. Swap this for EmailChannel(...) with real
    # SMTP credentials (dry_run=False) to actually deliver email.
    channel = ConsoleChannel()

    for event in events:

        print("\n")
        print("=" * 60)
        print(f"EVENT : {event.upper()}")
        print("=" * 60)

        for user in users:

            try:

                send_notification(
                    user,
                    event,
                    data,
                    channel=channel,
                )

            except Exception as e:

                print(
                    f"Error sending notification "
                    f"to {user.name}: {e}"
                )


if __name__ == "__main__":
    main()
```

### `test_notification.py`

```python
import unittest

from users import User
from notification_service import send_notification, render_template
from template_loader import load_template
from exceptions import MissingDataError
from channels import ConsoleChannel, EmailChannel, SMSChannel


class TestNotificationSystem(unittest.TestCase):

    def setUp(self):

        self.data = {
            "date": "10 July",
            "time": "5 PM"
        }

        self.events = [
            "interview_scheduled",
            "interview_reminder",
            "interview_cancelled",
            "interview_completed"
        ]

    # --------------------------
    # Valid Notification Tests
    # --------------------------

    def test_english_notification(self):

        user = User("Vaishnavi", "vaish@gmail.com", "en")

        message = send_notification(
            user,
            "interview_scheduled",
            self.data
        )

        self.assertIn("Hello Vaishnavi", message)

    def test_hindi_notification(self):

        user = User("Jaya", "jaya@gmail.com", "hi")

        message = send_notification(
            user,
            "interview_scheduled",
            self.data
        )

        self.assertIn("नमस्ते Jaya", message)

    def test_telugu_notification(self):

        user = User("Anushka", "anu@gmail.com", "te")

        message = send_notification(
            user,
            "interview_scheduled",
            self.data
        )

        self.assertIn("హలో Anushka", message)

    # --------------------------
    # Locale Fallback
    # --------------------------

    def test_locale_fallback(self):

        user = User(
            "Alex",
            "alex@gmail.com",
            "fr"
        )

        # User() falls back to "en" at creation time already.
        self.assertEqual(user.locale, "en")

        message = send_notification(
            user,
            "interview_scheduled",
            self.data
        )

        self.assertIn("Hello Alex", message)

    def test_strict_locale_raises(self):

        with self.assertRaises(ValueError):
            User(
                "Alex",
                "alex@gmail.com",
                "fr",
                strict_locale=True,
            )

    # --------------------------
    # Placeholder Replacement / Templating engine
    # --------------------------

    def test_placeholder_replacement(self):

        user = User(
            "Vaishnavi",
            "vaish@gmail.com",
            "en"
        )

        message = send_notification(
            user,
            "interview_scheduled",
            self.data
        )

        self.assertIn("10 July", message)
        self.assertIn("5 PM", message)

    def test_render_template_single_pass_repeated_placeholder(self):

        template = "{{name}} - {{name}} - {{date}}"
        result = render_template(
            template,
            {"name": "Sam", "date": "1 Jan"},
        )
        self.assertEqual(result, "Sam - Sam - 1 Jan")

    def test_render_template_missing_value_raises_missing_data_error(self):

        with self.assertRaises(MissingDataError):
            render_template("Hello {{name}}", {})

    def test_render_template_html_escapes_values(self):

        template = "Hello {{name}}"
        result = render_template(
            template,
            {"name": "<script>alert(1)</script>"},
            escape=True,
        )
        self.assertNotIn("<script>", result)
        self.assertIn("&lt;script&gt;", result)

    # --------------------------
    # Invalid Event
    # --------------------------

    def test_invalid_event(self):

        user = User(
            "Vaishnavi",
            "vaish@gmail.com",
            "en"
        )

        with self.assertRaises(FileNotFoundError):

            send_notification(
                user,
                "random_event",
                self.data
            )

    # --------------------------
    # Path traversal protection
    # --------------------------

    def test_locale_path_traversal_rejected(self):

        with self.assertRaises(ValueError):
            load_template("../../../etc", "interview_scheduled")

    def test_event_path_traversal_rejected(self):

        with self.assertRaises(ValueError):
            load_template("en", "../../../etc/passwd")

    def test_event_with_null_byte_rejected(self):

        with self.assertRaises(ValueError):
            load_template("en", "interview_scheduled\x00.txt")

    # --------------------------
    # Missing Date / Time -> no unhandled KeyError
    # --------------------------

    def test_missing_date(self):

        user = User(
            "Vaishnavi",
            "vaish@gmail.com",
            "en"
        )

        with self.assertRaises(ValueError):

            send_notification(
                user,
                "interview_scheduled",
                {
                    "time": "5 PM"
                }
            )

    def test_missing_time(self):

        user = User(
            "Vaishnavi",
            "vaish@gmail.com",
            "en"
        )

        with self.assertRaises(ValueError):

            send_notification(
                user,
                "interview_scheduled",
                {
                    "date": "10 July"
                }
            )

    def test_missing_data_never_raises_bare_key_error(self):
        """
        Regression test: send_notification must never crash with a raw,
        unhandled KeyError. Any missing-data condition should surface as
        a ValueError/MissingDataError instead.
        """

        user = User("Vaishnavi", "vaish@gmail.com", "en")

        try:
            send_notification(user, "interview_scheduled", {})
        except KeyError:
            self.fail("send_notification leaked a raw KeyError")
        except (ValueError, MissingDataError):
            pass

    # --------------------------
    # Invalid User
    # --------------------------

    def test_invalid_user(self):

        with self.assertRaises(ValueError):

            User(
                "",
                "abc@gmail.com",
                "en"
            )

    # --------------------------
    # Invalid Email
    # --------------------------

    def test_invalid_email(self):

        with self.assertRaises(ValueError):

            User(
                "Vaishnavi",
                "wrongemail",
                "en"
            )

    # --------------------------
    # Unicode Handling
    # --------------------------

    def test_unicode_name(self):

        user = User(
            "वैष्णवी",
            "abc@gmail.com",
            "hi"
        )

        message = send_notification(
            user,
            "interview_scheduled",
            self.data
        )

        self.assertIn("वैष्णवी", message)

    # --------------------------
    # Multiple Events
    # --------------------------

    def test_all_events(self):

        user = User(
            "Vaishnavi",
            "abc@gmail.com",
            "en"
        )

        for event in self.events:

            message = send_notification(
                user,
                event,
                self.data
            )

            self.assertIsInstance(message, str)

    # --------------------------
    # Exception Handling
    # --------------------------

    def test_none_user(self):

        with self.assertRaises(ValueError):

            send_notification(
                None,
                "interview_scheduled",
                self.data
            )

    # --------------------------
    # HTML templates (email channel)
    # --------------------------

    def test_html_template_loaded_for_email_channel(self):

        user = User("Vaishnavi", "vaish@gmail.com", "en")
        channel = EmailChannel(dry_run=True)

        message = send_notification(
            user,
            "interview_scheduled",
            self.data,
            channel=channel,
        )

        self.assertIn("<html", message.lower())
        self.assertIn("10 July", message)

    def test_all_locales_have_html_templates(self):

        for locale in User.SUPPORTED_LOCALES:
            for event in self.events:
                template = load_template(locale, event, channel="email")
                self.assertIn("<html", template.lower())

    # --------------------------
    # Dispatch channels
    # --------------------------

    def test_console_channel_dispatch(self):

        user = User("Vaishnavi", "vaish@gmail.com", "en")
        channel = ConsoleChannel()

        message = send_notification(
            user,
            "interview_scheduled",
            self.data,
            channel=channel,
        )
        self.assertIsInstance(message, str)

    def test_email_channel_dry_run_does_not_require_network(self):

        user = User("Vaishnavi", "vaish@gmail.com", "en")
        channel = EmailChannel(dry_run=True)

        result = channel.send(user, "Subject", "Body")
        self.assertTrue(result)

    def test_sms_channel_dry_run_does_not_require_network(self):

        user = User("Vaishnavi", "vaish@gmail.com", "en")
        channel = SMSChannel(dry_run=True)

        result = channel.send(user, "Subject", "Body")
        self.assertTrue(result)


if __name__ == "__main__":
    unittest.main()
```

## Templates

Each locale (`en`, `hi`, `te`) has the same 4 events, each with a `.txt` (console/SMS) and `.html` (email) variant. All 24 files are listed below.

### Locale: `en`

**`templates/en/interview_scheduled.txt`**

```text
Hello {{name}},

Your interview has been scheduled.

Date: {{date}}

Time: {{time}}

Thank you,
Interview Team
```

**`templates/en/interview_scheduled.html`**

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>Interview Scheduled</title>
</head>
<body style="font-family: Arial, sans-serif; color: #222;">
  <div style="max-width: 480px; margin: 0 auto; padding: 24px; border: 1px solid #eee;">
    <p>Hello {{name}},</p>
    <p>Your interview has been scheduled.</p>
    <p><strong>Date:</strong> {{date}}</p>
    <p><strong>Time:</strong> {{time}}</p>
    <p>Thank you,<br>Interview Team</p>
  </div>
</body>
</html>
```

**`templates/en/interview_reminder.txt`**

```text
Hello {{name}},

This is a reminder that your interview is scheduled on {{date}} at {{time}}.

Please be available on time.

Thank you,
Interview Team
```

**`templates/en/interview_reminder.html`**

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>Interview Reminder</title>
</head>
<body style="font-family: Arial, sans-serif; color: #222;">
  <div style="max-width: 480px; margin: 0 auto; padding: 24px; border: 1px solid #eee;">
    <p>Hello {{name}},</p>
    <p>This is a reminder that your interview is scheduled on {{date}} at {{time}}.</p>
    <p>Please be available on time.</p>
    <p>Thank you,<br>Interview Team</p>
  </div>
</body>
</html>
```

**`templates/en/interview_cancelled.txt`**

```text
Hello {{name}},

Your interview scheduled on {{date}} at {{time}} has been cancelled.

We will contact you soon with a new schedule.

Thank you,
Interview Team
```

**`templates/en/interview_cancelled.html`**

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>Interview Cancelled</title>
</head>
<body style="font-family: Arial, sans-serif; color: #222;">
  <div style="max-width: 480px; margin: 0 auto; padding: 24px; border: 1px solid #eee;">
    <p>Hello {{name}},</p>
    <p>Your interview scheduled on {{date}} at {{time}} has been cancelled.</p>
    <p>We will contact you soon with a new schedule.</p>
    <p>Thank you,<br>Interview Team</p>
  </div>
</body>
</html>
```

**`templates/en/interview_completed.txt`**

```text
Hello {{name}},

Your interview conducted on {{date}} at {{time}} has been completed successfully.

Thank you for participating.

Regards,
Interview Team
```

**`templates/en/interview_completed.html`**

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>Interview Completed</title>
</head>
<body style="font-family: Arial, sans-serif; color: #222;">
  <div style="max-width: 480px; margin: 0 auto; padding: 24px; border: 1px solid #eee;">
    <p>Hello {{name}},</p>
    <p>Your interview conducted on {{date}} at {{time}} has been completed successfully.</p>
    <p>Thank you for participating.</p>
    <p>Regards,<br>Interview Team</p>
  </div>
</body>
</html>
```

### Locale: `hi`

**`templates/hi/interview_scheduled.txt`**

```text
नमस्ते {{name}},

आपका इंटरव्यू निर्धारित किया गया है।

दिनांक: {{date}}

समय: {{time}}

धन्यवाद,
इंटरव्यू टीम
```

**`templates/hi/interview_scheduled.html`**

```html
<!DOCTYPE html>
<html lang="hi">
<head>
  <meta charset="UTF-8">
  <title>Interview Scheduled</title>
</head>
<body style="font-family: Arial, sans-serif; color: #222;">
  <div style="max-width: 480px; margin: 0 auto; padding: 24px; border: 1px solid #eee;">
    <p>नमस्ते {{name}},</p>
    <p>आपका इंटरव्यू निर्धारित किया गया है।</p>
    <p><strong>दिनांक:</strong> {{date}}</p>
    <p><strong>समय:</strong> {{time}}</p>
    <p>धन्यवाद,<br>इंटरव्यू टीम</p>
  </div>
</body>
</html>
```

**`templates/hi/interview_reminder.txt`**

```text
नमस्ते {{name}},

यह याद दिलाने के लिए है कि आपका इंटरव्यू {{date}} को {{time}} बजे निर्धारित है।

कृपया समय पर उपलब्ध रहें।

धन्यवाद,
इंटरव्यू टीम
```

**`templates/hi/interview_reminder.html`**

```html
<!DOCTYPE html>
<html lang="hi">
<head>
  <meta charset="UTF-8">
  <title>Interview Reminder</title>
</head>
<body style="font-family: Arial, sans-serif; color: #222;">
  <div style="max-width: 480px; margin: 0 auto; padding: 24px; border: 1px solid #eee;">
    <p>नमस्ते {{name}},</p>
    <p>यह याद दिलाने के लिए है कि आपका इंटरव्यू {{date}} को {{time}} बजे निर्धारित है।</p>
    <p>कृपया समय पर उपलब्ध रहें।</p>
    <p>धन्यवाद,<br>इंटरव्यू टीम</p>
  </div>
</body>
</html>
```

**`templates/hi/interview_cancelled.txt`**

```text
नमस्ते {{name}},

{{date}} को {{time}} बजे निर्धारित आपका इंटरव्यू रद्द कर दिया गया है।

हम नए समय-सारणी के साथ आपसे जल्द संपर्क करेंगे।

धन्यवाद,
इंटरव्यू टीम
```

**`templates/hi/interview_cancelled.html`**

```html
<!DOCTYPE html>
<html lang="hi">
<head>
  <meta charset="UTF-8">
  <title>Interview Cancelled</title>
</head>
<body style="font-family: Arial, sans-serif; color: #222;">
  <div style="max-width: 480px; margin: 0 auto; padding: 24px; border: 1px solid #eee;">
    <p>नमस्ते {{name}},</p>
    <p>{{date}} को {{time}} बजे निर्धारित आपका इंटरव्यू रद्द कर दिया गया है।</p>
    <p>हम नए समय-सारणी के साथ आपसे जल्द संपर्क करेंगे।</p>
    <p>धन्यवाद,<br>इंटरव्यू टीम</p>
  </div>
</body>
</html>
```

**`templates/hi/interview_completed.txt`**

```text
नमस्ते {{name}},

{{date}} को {{time}} बजे आयोजित आपका इंटरव्यू सफलतापूर्वक पूरा हो गया है।

भाग लेने के लिए धन्यवाद।

सादर,
इंटरव्यू टीम
```

**`templates/hi/interview_completed.html`**

```html
<!DOCTYPE html>
<html lang="hi">
<head>
  <meta charset="UTF-8">
  <title>Interview Completed</title>
</head>
<body style="font-family: Arial, sans-serif; color: #222;">
  <div style="max-width: 480px; margin: 0 auto; padding: 24px; border: 1px solid #eee;">
    <p>नमस्ते {{name}},</p>
    <p>{{date}} को {{time}} बजे आयोजित आपका इंटरव्यू सफलतापूर्वक पूरा हो गया है।</p>
    <p>भाग लेने के लिए धन्यवाद।</p>
    <p>सादर,<br>इंटरव्यू टीम</p>
  </div>
</body>
</html>
```

### Locale: `te`

**`templates/te/interview_scheduled.txt`**

```text
హలో {{name}},

మీ ఇంటర్వ్యూ షెడ్యూల్ చేయబడింది.

తేదీ: {{date}}

సమయం: {{time}}

ధన్యవాదాలు,
ఇంటర్వ్యూ టీమ్
```

**`templates/te/interview_scheduled.html`**

```html
<!DOCTYPE html>
<html lang="te">
<head>
  <meta charset="UTF-8">
  <title>Interview Scheduled</title>
</head>
<body style="font-family: Arial, sans-serif; color: #222;">
  <div style="max-width: 480px; margin: 0 auto; padding: 24px; border: 1px solid #eee;">
    <p>హలో {{name}},</p>
    <p>మీ ఇంటర్వ్యూ షెడ్యూల్ చేయబడింది.</p>
    <p><strong>తేదీ:</strong> {{date}}</p>
    <p><strong>సమయం:</strong> {{time}}</p>
    <p>ధన్యవాదాలు,<br>ఇంటర్వ్యూ టీమ్</p>
  </div>
</body>
</html>
```

**`templates/te/interview_reminder.txt`**

```text
నమస్తే {{name}},

మీ ఇంటర్వ్యూ {{date}}న {{time}}కు షెడ్యూల్ చేయబడిందని ఇది ఒక రిమైండర్.

దయచేసి సమయానికి అందుబాటులో ఉండండి.

ధన్యవాదాలు,
ఇంటర్వ్యూ టీమ్
```

**`templates/te/interview_reminder.html`**

```html
<!DOCTYPE html>
<html lang="te">
<head>
  <meta charset="UTF-8">
  <title>Interview Reminder</title>
</head>
<body style="font-family: Arial, sans-serif; color: #222;">
  <div style="max-width: 480px; margin: 0 auto; padding: 24px; border: 1px solid #eee;">
    <p>హలో {{name}},</p>
    <p>మీ ఇంటర్వ్యూ {{date}}న {{time}}కు షెడ్యూల్ చేయబడిందని ఇది ఒక రిమైండర్.</p>
    <p>దయచేసి సమయానికి అందుబాటులో ఉండండి.</p>
    <p>ధన్యవాదాలు,<br>ఇంటర్వ్యూ టీమ్</p>
  </div>
</body>
</html>
```

**`templates/te/interview_cancelled.txt`**

```text
నమస్తే {{name}},

{{date}}న {{time}}కు షెడ్యూల్ చేసిన మీ ఇంటర్వ్యూ రద్దు చేయబడింది.

కొత్త షెడ్యూల్‌తో మేము త్వరలో మిమ్మల్ని సంప్రదిస్తాము.

ధన్యవాదాలు,
ఇంటర్వ్యూ టీమ్
```

**`templates/te/interview_cancelled.html`**

```html
<!DOCTYPE html>
<html lang="te">
<head>
  <meta charset="UTF-8">
  <title>Interview Cancelled</title>
</head>
<body style="font-family: Arial, sans-serif; color: #222;">
  <div style="max-width: 480px; margin: 0 auto; padding: 24px; border: 1px solid #eee;">
    <p>హలో {{name}},</p>
    <p>{{date}}న {{time}}కు షెడ్యూల్ చేసిన మీ ఇంటర్వ్యూ రద్దు చేయబడింది.</p>
    <p>కొత్త షెడ్యూల్‌తో మేము త్వరలో మిమ్మల్ని సంప్రదిస్తాము.</p>
    <p>ధన్యవాదాలు,<br>ఇంటర్వ్యూ టీమ్</p>
  </div>
</body>
</html>
```

**`templates/te/interview_completed.txt`**

```text
నమస్తే {{name}},

{{date}}న {{time}}కు నిర్వహించిన మీ ఇంటర్వ్యూ విజయవంతంగా పూర్తయింది.

పాల్గొన్నందుకు ధన్యవాదాలు.

శుభాకాంక్షలతో,
ఇంటర్వ్యూ టీమ్
```

**`templates/te/interview_completed.html`**

```html
<!DOCTYPE html>
<html lang="te">
<head>
  <meta charset="UTF-8">
  <title>Interview Completed</title>
</head>
<body style="font-family: Arial, sans-serif; color: #222;">
  <div style="max-width: 480px; margin: 0 auto; padding: 24px; border: 1px solid #eee;">
    <p>హలో {{name}},</p>
    <p>{{date}}న {{time}}కు నిర్వహించిన మీ ఇంటర్వ్యూ విజయవంతంగా పూర్తయింది.</p>
    <p>పాల్గొన్నందుకు ధన్యవాదాలు.</p>
    <p>శుభాకాంక్షలతో,<br>ఇంటర్వ్యూ టీమ్</p>
  </div>
</body>
</html>
```
