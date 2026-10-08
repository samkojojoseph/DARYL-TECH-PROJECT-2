# DARYL-TECH-PROJECT-2
CONTACT BOOK
import csv
import os
import re
import shutil
from typing import Optional

# Name of the CSV file that stores every contact record.
FILE_NAME = "contacts.csv"
# Column order used both when writing the CSV and when reading it back.
FIELDNAMES = ["ID", "Name", "Phone", "Email"]

# Precompiled regex patterns — compiling once here is faster than
# re-compiling the pattern every time a function is called.
NAME_PATTERN = re.compile(r"[A-Za-z ]+")
EMAIL_PATTERN = re.compile(r"^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$")

# A local Ghana number: 10 digits, starts with 0, and the digit right
# after the 0 must be 1-9 — this is what blocks fake numbers like
# "0055512345" that only look valid because they're 10 digits long.
GHANA_LOCAL_PATTERN = re.compile(r"^0[1-9]\d{8}$")

# Common country codes for international contacts, mapped to
# (country name, how many digits should follow the code). These lengths
# are the typical national number length once the leading 0 is dropped
# (e.g. Ghana's local "0551234567" becomes "+233551234567").
COUNTRY_CODES: dict[str, tuple[str, Optional[int]]] = {
    "233": ("Ghana", 9),
    "234": ("Nigeria", 10),
    "225": ("Ivory Coast", 8),
    "228": ("Togo", 8),
    "254": ("Kenya", 9),
    "27": ("South Africa", 9),
    "44": ("United Kingdom", 10),
    "1": ("USA / Canada", 10),
}


# --------------------------------------------------
# FILE SETUP
# --------------------------------------------------

def create_file() -> None:
    """Create contacts.csv with just a header row if it doesn't exist yet."""
    try:
        # Only create the file the first time the program runs; if it's
        # already there we don't want to wipe out existing contacts.
        if not os.path.exists(FILE_NAME):
            with open(FILE_NAME, "w", newline="", encoding="utf-8") as file:
                csv.writer(file).writerow(FIELDNAMES)
    except PermissionError:
        # Raised if the OS blocks us from creating a file in this folder.
        print("Error: You do not have permission to create the file.")
    except OSError:
        # Catch-all for other filesystem problems (disk full, bad path, etc.).
        print("Error: The file could not be created.")


def backup_file() -> None:
    """Copy contacts.csv to contacts.csv.bak before we overwrite it.

    This runs right before update/delete rewrite the whole file, so if
    something goes wrong mid-write there's still a recoverable copy.
    """
    if os.path.exists(FILE_NAME):
        try:
            shutil.copyfile(FILE_NAME, FILE_NAME + ".bak")
        except OSError:
            # A failed backup shouldn't stop the actual save from proceeding.
            pass


# --------------------------------------------------
# VALIDATION
# --------------------------------------------------

def validate_name(name: str) -> bool:
    """Name must be non-empty and contain only letters and spaces."""
    return bool(name) and NAME_PATTERN.fullmatch(name) is not None


def validate_email(email: str) -> bool:
    """Email must match the standard local-part@domain.tld shape."""
    return EMAIL_PATTERN.fullmatch(email) is not None


def strip_phone_spacing(raw: str) -> str:
    """Remove spaces and dashes people naturally type in a phone number.

    "055 954 4787" and "055-954-4787" should both be treated the same
    as "0559544787" — this collapses them all to just the digits.
    """
    return re.sub(r"[\s-]", "", raw)


def validate_local_phone(phone: str) -> bool:
    """A Ghana local number: exactly 10 digits, starting 0 then 1-9."""
    return GHANA_LOCAL_PATTERN.fullmatch(phone) is not None


def validate_international_digits(digits: str, code: str) -> bool:
    """Check the digits typed after a country code are the right length.

    If the code isn't one of our known ones (a custom code the user
    typed in), fall back to a loose 7-12 digit sanity check instead of
    rejecting it outright.
    """
    if not digits.isdigit():
        return False
    expected_length = COUNTRY_CODES.get(code, ("Unknown", None))[1]
    if expected_length is None:
        return 7 <= len(digits) <= 12
    return len(digits) == expected_length


# --------------------------------------------------
# READ / WRITE
# --------------------------------------------------

def read_records() -> list[dict]:
    """Load every contact from the CSV file as a list of dicts.

    Each dict has keys ID, Name, Phone, Email — this is what
    csv.DictReader gives us for free based on the header row.
    """
    try:
        with open(FILE_NAME, "r", newline="", encoding="utf-8") as file:
            return list(csv.DictReader(file))
    except FileNotFoundError:
        # File got deleted after the program started — recreate it
        # instead of crashing, and return an empty contact list.
        print("Error: Contact file does not exist. Creating a new one.")
        create_file()
        return []
    except PermissionError:
        print("Error: You do not have permission to read the file.")
        return []
    except csv.Error:
        # Raised if the file exists but isn't valid CSV (e.g. hand-edited
        # and broken).
        print("Error: The contact file is corrupted.")
        return []
    except OSError:
        print("Error: The file could not be read.")
        return []


def write_records(records: list[dict]) -> bool:
    """Overwrite the CSV file with the given list of contact dicts.

    Used by update and delete, which both need to rewrite the whole
    file rather than append. Returns True on success so callers know
    whether to print a success message.
    """
    try:
        # Snapshot the old file first in case this write fails partway.
        backup_file()
        with open(FILE_NAME, "w", newline="", encoding="utf-8") as file:
            writer = csv.DictWriter(file, fieldnames=FIELDNAMES)
            writer.writeheader()
            writer.writerows(records)
        return True
    except PermissionError:
        print("Error: You do not have permission to modify the file.")
    except OSError:
        print("Error: The file could not be written.")
    return False


# --------------------------------------------------
# DUPLICATE CHECKS
# --------------------------------------------------

def phone_exists(records: list[dict], phone: str, exclude_id: Optional[str] = None) -> bool:
    """True if some other contact already has this phone number.

    exclude_id lets update_contact check against everyone EXCEPT the
    record currently being edited, so re-saving your own number
    doesn't falsely flag as a duplicate.
    """
    return any(r["Phone"] == phone and r["ID"] != exclude_id for r in records)


def email_exists(records: list[dict], email: str, exclude_id: Optional[str] = None) -> bool:
    """Same idea as phone_exists, but for email (compared case-insensitively)."""
    return any(
        r["Email"].lower() == email.lower() and r["ID"] != exclude_id
        for r in records
    )


# --------------------------------------------------
# PHONE NUMBER ENTRY (local or international)
# --------------------------------------------------

def choose_country_code() -> Optional[str]:
    """Show a numbered list of countries and return the code the user picks.

    Returns None if the user's input didn't match anything, so the
    caller can just loop back and ask again.
    """
    codes_list = list(COUNTRY_CODES.items())

    print("Select a country:")
    for position, (code, (name, _)) in enumerate(codes_list, start=1):
        print(f"  {position}. {name} (+{code})")
    other_option = len(codes_list) + 1
    print(f"  {other_option}. Other (type in a country code)")

    selection = input("Enter a number: ").strip()
    if not selection.isdigit():
        return None

    index = int(selection)
    if 1 <= index <= len(codes_list):
        return codes_list[index - 1][0]

    if index == other_option:
        custom_code = input("Enter the country code (digits only, no +): ").strip()
        if custom_code.isdigit() and 1 <= len(custom_code) <= 3:
            # Remember it for this session so re-checks (e.g. duplicate
            # look-ups later) know its expected length too.
            COUNTRY_CODES.setdefault(custom_code, ("Custom", None))
            return custom_code

    return None


def prompt_phone(records: list[dict], exclude_id: Optional[str] = None,
                  current: Optional[str] = None) -> Optional[str]:
    """Ask for a phone number as either a local or international format.

    current is the contact's existing phone number when this is called
    from update_contact — pressing Enter at the type prompt keeps it
    unchanged, signalled by returning None. When adding a brand-new
    contact, current is None and the user must supply a real number.
    """
    while True:
        if current is not None:
            print(f"Current phone: {current}")

        print("Phone number type:")
        print("  1. Local Ghana number (e.g. 0551234567)")
        print("  2. International number (with a country code)")
        prompt_text = "Choose 1 or 2"
        if current is not None:
            prompt_text += " (Enter to keep current)"
        choice = input(f"{prompt_text}: ").strip()

        if current is not None and choice == "":
            return None

        if choice == "1":
            phone = strip_phone_spacing(input("Enter phone number: ").strip())
            if not validate_local_phone(phone):
                print("Invalid number! Must be 10 digits, start with 0, "
                      "and the next digit can't be 0 (e.g. 0551234567).")
                continue

        elif choice == "2":
            code = choose_country_code()
            if code is None:
                print("Invalid selection.")
                continue
            country_name = COUNTRY_CODES[code][0]
            digits = strip_phone_spacing(
                input(f"Enter the {country_name} number after +{code}: ").strip()
            )
            if not validate_international_digits(digits, code):
                expected_length = COUNTRY_CODES[code][1]
                if expected_length is None:
                    print("Invalid number! Enter 7 to 12 digits.")
                else:
                    print(f"Invalid number! {country_name} numbers need "
                          f"{expected_length} digits after +{code}.")
                continue
            phone = f"+{code}{digits}"

        else:
            print("Invalid choice! Enter 1 or 2.")
            continue

        if phone_exists(records, phone, exclude_id=exclude_id):
            print("A contact with this phone number already exists.")
            continue

        return phone


# --------------------------------------------------
# ADD CONTACT
# --------------------------------------------------

def add_contact() -> None:
    """Prompt for a new contact's details, validate them, and append to the CSV."""
    print("\n========== ADD CONTACT ==========")
    # Load existing contacts up front so we can check for duplicates
    # before accepting the phone/email the user types in.
    records = read_records()

    # Keep asking until a valid name is entered.
    while True:
        name = input("Enter name: ").strip()
        if validate_name(name):
            break
        print("Invalid name! Name must contain letters and spaces only.")

    # current=None here means there's no existing number to fall back
    # on, so prompt_phone will keep looping until it gets a real one.
    phone = prompt_phone(records)

    # Same two-part check for email: format, then not already taken.
    while True:
        email = input("Enter email: ").strip()
        if not validate_email(email):
            print("Invalid email! Example: john@gmail.com")
            continue
        if email_exists(records, email):
            print("A contact with this email already exists.")
            continue
        break

    try:
        # New ID is one higher than the current highest ID (or 1 if the
        # file is empty). default=0 handles the empty-list case safely.
        new_id = str(max((int(r["ID"]) for r in records), default=0) + 1)
        # Append mode ("a") adds a single row without touching the rest
        # of the file — faster than rewriting everything for an add.
        with open(FILE_NAME, "a", newline="", encoding="utf-8") as file:
            csv.writer(file).writerow([new_id, name, phone, email])
        print("\nContact added successfully!")
    except (PermissionError, OSError, ValueError):
        print("Error: The contact could not be saved.")


# --------------------------------------------------
# VIEW CONTACTS
# --------------------------------------------------

def view_contacts() -> None:
    """Print every contact as a neatly aligned table."""
    print("\n========== CONTACT LIST ==========")
    records = read_records()

    if not records:
        print("No contacts found.")
        return

    # Work out how wide each column needs to be so every row lines up —
    # start with the header's width, then grow it for any longer value.
    widths = {field: len(field) for field in FIELDNAMES}
    for record in records:
        for field in FIELDNAMES:
            widths[field] = max(widths[field], len(record[field]))

    # ljust pads each value with spaces up to the column's width.
    header = "  ".join(field.ljust(widths[field]) for field in FIELDNAMES)
    print(header)
    print("-" * len(header))
    for record in records:
        print("  ".join(record[field].ljust(widths[field]) for field in FIELDNAMES))


# --------------------------------------------------
# UPDATE CONTACT
# --------------------------------------------------

def update_contact() -> None:
    """Find a contact by ID and let the user edit any of their fields."""
    print("\n========== UPDATE CONTACT ==========")
    records = read_records()

    if not records:
        print("No contacts available.")
        return

    contact_id = input("Enter contact ID to update: ").strip()
    # next(..., None) grabs the first matching record, or None if no
    # contact has that ID — avoids writing a manual loop with a flag.
    target = next((r for r in records if r["ID"] == contact_id), None)

    if target is None:
        print("Error: Contact ID not found.")
        return

    print("Leave a field empty to keep the old value.")

    # Empty input = keep the old name, otherwise it must pass validation.
    while True:
        new_name = input(f"New name ({target['Name']}): ").strip()
        if new_name == "" or validate_name(new_name):
            if new_name:
                target["Name"] = new_name
            break
        print("Invalid name! Name must contain letters and spaces only.")

    # prompt_phone returns None to mean "keep the current number" —
    # only overwrite target["Phone"] when it actually returns a new one.
    new_phone = prompt_phone(records, exclude_id=contact_id, current=target["Phone"])
    if new_phone is not None:
        target["Phone"] = new_phone

    while True:
        new_email = input(f"New email ({target['Email']}): ").strip()
        if new_email == "":
            break
        if not validate_email(new_email):
            print("Invalid email! Example: john@gmail.com")
            continue
        if email_exists(records, new_email, exclude_id=contact_id):
            print("Another contact already uses this email.")
            continue
        target["Email"] = new_email
        break

    # target is a reference into records, so it's already updated in
    # place — we just need to persist the whole list back to disk.
    if write_records(records):
        print("Contact updated successfully!")


# --------------------------------------------------
# DELETE CONTACT
# --------------------------------------------------

def delete_contact() -> None:
    """Find a contact by ID, confirm with the user, then remove it."""
    print("\n========== DELETE CONTACT ==========")
    records = read_records()

    if not records:
        print("No contacts available.")
        return

    contact_id = input("Enter contact ID to delete: ").strip()
    target = next((r for r in records if r["ID"] == contact_id), None)

    if target is None:
        print("Error: Contact ID not found.")
        return

    # .lower() means Y, y, Yes, YES, yes all count as confirmation —
    # anything else (including just pressing Enter) cancels.
    confirm = input(f"Delete {target['Name']} (ID {contact_id})? [y/N]: ").strip().lower()
    if confirm not in ("y", "yes"):
        print("Deletion cancelled.")
        return

    # Rebuild the list without the deleted contact rather than mutating
    # while iterating, which avoids skipping items.
    remaining = [r for r in records if r["ID"] != contact_id]

    if write_records(remaining):
        print("Contact deleted successfully!")


# --------------------------------------------------
# MAIN MENU
# --------------------------------------------------

def main() -> None:
    """Show the menu in a loop until the user chooses to exit."""
    # Make sure the CSV file exists before anything else runs.
    create_file()

    # Maps a menu number straight to the function that handles it —
    # avoids a long if/elif chain and makes adding new options easy.
    menu_actions = {
        "1": add_contact,
        "2": view_contacts,
        "3": update_contact,
        "4": delete_contact,
    }

    while True:
        print("\n================================")
        print("       STUDENT CONTACT BOOK")
        print("================================")
        print("1. Add Contact")
        print("2. View Contacts")
        print("3. Update Contact")
        print("4. Delete Contact")
        print("5. Exit")
        print("================================")

        choice = input("Enter your choice (1-5): ").strip()

        if choice == "5":
            print("\nThank you for using Student Contact Book!")
            print("Program closed.")
            break

        action = menu_actions.get(choice)
        if action:
            action()
        else:
            print("Error: Invalid choice. Please select a number from 1 to 5.")


# Only run main() if this file is executed directly (not imported as a module).
if __name__ == "__main__":
    try:
        main()
    except KeyboardInterrupt:
        # Lets Ctrl+C exit cleanly instead of showing a stack trace.
        print("\n\nProgram interrupted by user.")
    except Exception:
        # Last-resort safety net so an unexpected bug doesn't crash ugly.
        print("\nAn unexpected error occurred. Please try again.")
