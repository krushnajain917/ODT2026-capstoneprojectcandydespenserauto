from mfrc522 import MFRC522
import time

# ── Pin wiring ──────────────────────────────────────────
SCK_PIN  = 18   # SCK  → D18
MOSI_PIN = 23   # MOSI → D23
MISO_PIN = 19   # MISO → D19
RST_PIN  = 4    # RST  → D4
CS_PIN   = 21   # SDA  → D21

# ── Your 3 known tag UIDs → colour zones ────────────────
# Assign these to your 3 physical wheel zones however you like
KNOWN_TAGS = {
    (0xAA, 0x8A, 0x74, 0x10, 0x44): "RED",
    (0x7A, 0x9A, 0x5F, 0x10, 0xAF): "GREEN",
    (0x3C, 0x0B, 0xD5, 0x06, 0xE4): "BLUE",
}

STABLE_NEEDED = 12   # same UID this many times in a row = wheel stopped
POLL_MS       = 50   # ms between reads

# ── Init reader ─────────────────────────────────────────
print("Initialising RFID reader...")
reader = MFRC522(sck=SCK_PIN, mosi=MOSI_PIN, miso=MISO_PIN,
                 rst=RST_PIN, cs=CS_PIN)
print("Ready. Hold or spin a tag near the reader.\n")

last_uid     = None
stable_count = 0
announced    = False   # prevents repeated "STOPPED" prints

while True:
    stat, _ = reader.request(reader.REQIDL)

    if stat == reader.OK:
        stat, uid = reader.anticoll()

        if stat == reader.OK:
            uid_t   = tuple(uid)
            uid_hex = '0x' + ''.join('%02X' % b for b in uid_t)
            colour  = KNOWN_TAGS.get(uid_t, "UNKNOWN-TAG")

            if uid_t == last_uid:
                stable_count += 1
            else:
                # New tag seen — reset stability
                stable_count = 1
                last_uid     = uid_t
                announced    = False
                print(f"  Saw: {uid_hex}  ({colour})")

            # Stability threshold reached
            if stable_count >= STABLE_NEEDED and not announced:
                announced = True
                print(f"\n*** WHEEL STOPPED ***")
                print(f"    Colour : {colour}")
                print(f"    UID    : {uid_hex}")
                print(f"    Reads  : {stable_count} consecutive\n")
                # ← In game code: uart.write(f"WHEEL:{colour}\n")

    else:
        # No tag in field — wheel moved away or nothing present
        if last_uid is not None:
            stable_count = 0
            last_uid     = None
            announced    = False

    time.sleep_ms(POLL_MS)
