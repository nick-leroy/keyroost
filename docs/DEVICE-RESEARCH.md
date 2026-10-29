# Per-device research plan

A runbook for a future Claude to work through **when each device arrives**.
Two independent research threads. Keep findings in this file (it's the durable
record). This is an **outline**, not a finished design — go only as deep as each
step needs.

Hardware in play (all on hand): Token2 Molto2 (`349e:0300`), YubiKey 5
(`1050:0407`, fw 5.7.1) ×2, SoloKeys Solo 2 (`1209:beee`, Trussed, fw 2.964).
Nothing here is "incoming" any more — the Solo 2 carried the first successful
FIDO F1–F5 run (`docs/BRINGUP.md`), and the two-YubiKey case is what the
correlation work was verified against. Token2 PIN+ / FIDO2+ coverage
comes mostly from the vendor (PIN+ OTP over USB, #155, was also checked on a
PIN+ FIDO + PGP key, R3.4), and the OnlyKey facts from the
vendor and device surveys — see the identity note below; it is the interesting case.

- **Thread A — Device identity** (for local "friendly names"). Narrow, technical,
  has a privacy gate.
- **Thread B — Capabilities & day-to-day security uses** (feeds future in-UI
  guidance). Broad, user-facing, lower technical risk but needs accuracy review.

---

# Thread A — Per-device identity for friendly names

## The question

Let users label each physical unit locally ("yubikey 1", "Molto 1"). That needs
a **stable, per-unit identifier** we can read **without writing to the device**
and **without undermining the key's anti-tracking design**.

## Principles / constraints (hard, not preferences)

1. **Read-only.** No writing markers/UUIDs/largeBlob/credentials to establish identity.
   The name a user can store on a key (`name set --store key`,
   [PROTOCOL-device-label.md](PROTOCOL-device-label.md)) is a label they chose,
   never an identity: selection trusts only the fingerprints this computer recorded.
2. **Local-only storage.** Captured ID stays on this host; never transmitted, never shown to a relying party.
3. **Respect FIDO2 anti-correlation.** FIDO2 omits a global device ID on purpose. If the only per-unit ID lives on a non-FIDO interface (e.g. a USB iSerial via OTP/CCID), using it re-introduces a correlatable hardware ID. Local-only use is the mitigation — state the trade-off, gate on the privacy review.
4. **No PINs, no secrets** required for identity probing.
5. **Handle captured IDs as sensitive.** Abbreviate serials (first/last 2 chars) in commits/logs/this file. This governs *durable, shareable* artifacts; it is not a redaction rule for the tool's own terminal output to the device's owner — `keyroostctl list` prints full serials, deliberately, because that is how you tell two of your own keys apart.

## What we already know (code survey, 2026-05)

| Device | Candidate ID | Status |
|---|---|---|
| **Molto2** | `DeviceInfo.serial` (read on connect) | **Works** — stable, per-unit, no extra read, no write. |
| **FIDO (any)** | `HidDevice` path/vid/pid/name | `path` not stable across re-plug; vid:pid per-model; no serial. |
| **FIDO (any)** | CTAP `AuthenticatorInfo.aaguid` | **Per-model, not per-unit** — useless for two identical keys. |
| **OnlyKey** | USB iSerial | **Unusable as identity** — the firmware reports a fixed serial (`1000000000`) on *every* unit (issue #37; the README roadmap records it as "fixed, non-unique"). FIDO-HID only, no CCID interface, so no serial fallback either. Not on the bench — vendor/device-survey fact, not one of our captures. |

**A serial is not automatically a per-unit ID.** The OnlyKey row is the
counterexample that constrains E1: a candidate that is stable across re-plug can
still be identical across units. The `X_A != X_B` half of the decisive experiment
below is therefore not optional — with a single unit on the bench, a fixed
vendor-wide serial is indistinguishable from a good one. keyroost already detects
repeated serials across connected keys and surfaces an advisory rather than
merging them (`duplicate_serial_note` in the GUI); a device in that state needs
placeholder handling before friendly names mean anything (issue #37).

Molto2 is solved; the open research is entirely FIDO-side.

## The decisive experiment: two-unit comparison

A candidate ID `X` is usable iff, across units A and B of the same model:
`X_A == X_A'` (stable across re-plug) **and** `X_A != X_B` (unique per unit).
This is why the work waits for a second identical unit.

## Experiments (read-only; note where root is needed)

- **E1 — USB string descriptors.** `lsusb -v -d <vid:pid> | grep -i iSerial`; `/sys/bus/usb/devices/*/serial`. Note which interface owns the serial (whole-device vs FIDO-only) — matters for constraint #3.
- **E2 — HID uniq ioctl.** `HIDIOCGRAWUNIQ` on `/dev/hidrawN` (usually empty for USB HID). Confirm.
- **E3 — AAGUID is model-level (control).** `keyroostctl fido info` on both units; confirm AAGUID is byte-identical.
- **E4 — Other-interface serial (privacy-sensitive).** YubiKey exposes a device serial via OTP/CCID. Establish feasibility + cost only; weigh constraint #3. Do not build a reader during research. *(Overtaken by events: the reader was built and shipped — `keyroost_resolve::ccid_serial_for`.)*
- **E5 — Vendor/management IDs.** Solo 2 / Trussed reports a device UUID via `solo2-cli`. Determine whether it's readable from the **application** (FIDO) interface or only bootloader/management.

## Per-device worksheets

Filled from what shipped rather than from a formal lab run — the thread closed by
being implemented (see "Thread A outcome"). Cells marked *not tested* stayed
untested because an earlier row already answered the question.

### YubiKey 5 (`1050:0407`, fw 5.7.1; two units on the bench)
| Candidate | Stable re-plug? | Unique per unit? | Read-only? | Privacy cost | Verdict |
|---|---|---|---|---|---|
| USB iSerial (E1) | n/a | n/a | yes | cross-interface correlatable | **not available** — most YubiKeys publish no `iSerialNumber`; the serial is only reachable over CCID (`keyroost_resolve::ccid_serial_for`) |
| HID uniq (E2) | not tested | not tested | yes | none | not pursued — E4 answered it first |
| AAGUID (E3) | yes | NO | yes | none | reject (model-level) — confirmed; the GUI treats AAGUID as a static *model* table |
| OTP/CCID serial (E4) | yes | yes — two units told apart on hardware during the correlation work | yes | high (anti-tracking) — mitigated by local-only, opt-in storage | **adopted as the fallback** (`ccid_serial_for`) |

### SoloKey / Solo 2 (`1209:beee`, Trussed, fw 2.964)
| Candidate | Stable re-plug? | Unique per unit? | Read-only? | Privacy cost | Verdict |
|---|---|---|---|---|---|
| USB iSerial (E1) | yes | assumed — 32 hex chars, per-unit by design, but only **one** unit was on the bench, so `X_A != X_B` was never run for this model | yes | correlatable across interfaces; mitigated by local-only storage | **adopted as the primary** |
| HID uniq (E2) | not tested | not tested | yes | none | not pursued — E1 sufficed |
| AAGUID (E3) | yes | NO | yes | none | reject (model-level) |
| Trussed/solo2 UUID (E5) | not tested | not tested | — | — | not pursued — E1 sufficed, and this would have needed a management interface |

## Privacy review (gate before any FIDO implementation)

1. Does the chosen ID create a tracking/correlation vector beyond this host?
2. Does reading it touch an interface FIDO2 keeps separate (constraint #3)? Worth it, or degrade gracefully?
3. **Graceful degradation:** if a unit exposes no usable per-unit ID, fall back to a non-persistent label ("the key in this port, now") and say so — don't fake stable identity.
4. Opt-in per device, with the stored ID visible/removable by the user?

## Thread A outcome — CLOSED (shipped)

Resolved in favor of **USB iSerial first, CCID-read serial as fallback**, local
and opt-in. Shipped as `keyroost-keyring` (friendly names keyed by a salted fingerprint
of the serial, never the serial itself, with an `IdSource` field recording how
the serial was obtained; a record is written only when the user names a key or
a key carrying its own name is first seen) and `keyroost-resolve` (`effective_serials` /
`read_effective_serial`, with `ccid_serial_for` / `ccid_serials_for` covering
keys that expose no USB serial — experiment E4, built after all, despite the
"do not build a reader during research" note above). Surfaced as
`keyroostctl name` (`set`, `clear`, `list`) plus the global `-d`/`--device
<NAME>`. E3
resolved as predicted: AAGUID is model-level, which is why
`crates/keyroost/src/ui/aaguid.rs` can be a static model table.

Two constraints the implementation had to add that this plan did not anticipate:

- **Non-unique serials are real.** See the OnlyKey note above — a serial is not
  automatically a per-unit ID. keyroost detects repeats among connected keys and
  shows an advisory rather than treating identical serials as one key.
- **Correlation fails closed where the platform reports no USB topology.**
  macOS reports no USB bus position (Windows does since 0.14.0, from the
  device tree), so a reader that two HID nodes could each own is not guessed
  at by vendor name; contended sets are shown
  separately. That is privacy-review item 3 ("graceful degradation") answered
  concretely: where identity can't be established, say so rather than fake it.
  ([#51](https://github.com/framefilter/keyroost/issues/51) follow-up below.)

The HID-only resolver (`read_effective_serial`, `connected_keys`) was removed in
v0.13.0; the GUI and CLI now share the device model (`keyroost_resolve::enumerate`),
and the CLI picks a row with `select::resolve_target_reading`.

## Identity matching across interfaces (#51, v0.13.0)

On macOS (and on Windows before 0.14.0) a key's FIDO-HID node and its
smart-card reader carry no USB position, so the two halves of one key could not be joined when two keys of
the same make were connected. Matching now runs in three steps: USB position
(where the OS reports it), then the identity each side reports, then the
existing vendor fallback. Identities are compared only within one encoding;
two encodings whose relation no real key has confirmed are never equated.

| Vendor | HID side | Smart-card side | Compared as |
|---|---|---|---|
| Yubico | management READ CONFIG over CTAPHID (vendor command `0xC2`, config page 0, as yubikit sends it); serial is TLV tag `0x02` (not sent to a YubiKey NEO, which doesn't answer it) | the serial already read by the reader probe (no extra traffic) | decimal serial |
| Solo 2 | USB `iSerialNumber` (the device UUID); fallback: admin UUID over CTAPHID (vendor command `0xE2`) | SELECT the admin applet `A0000008470000 0001`, then `00 62 00 00` → 16-byte UUID | lowercase hex UUID |
| Token2 | GET_INFO serial request (§6.10 of Token2's public OTP-on-FIDO protocol document) over HID, reply `D1 len …` | the same request after selecting the FIDO applet; if that fails, the OTP applet's reply, which is plain ASCII decimal | the FIDO-format reply with the FIDO-format reply only; the OTP-applet form only with itself |

**Evidence.** YubiKey 5 (fw 5.7) and Solo 2 (fw 2.964) were matched by
identity alone on hardware: with USB-position matching switched off in a test
build, the device list came out identical to the USB-position result. The two
CTAPHID vendor commands were confirmed read-only on the same keys (YubiKey
configuration unchanged; Solo 2 returned its UUID, keys unchanged). Token2 is
implemented from its public protocol document; how the FIDO-format and
OTP-applet encodings relate is not yet confirmed on a real key, so they are
compared only like-for-like until it is. A Windows community test is pending.
The Molto2 and the single-profile programmable tokens are never sent an
identity read.

**Privacy.** Identities are read only to join the two halves of a key, held in
memory (an answer over HID is kept while that key stays plugged in, so later
scans don't ask again; a replug asks again), and never written to disk: `keys.json` holds only a salted
fingerprint, saved when the user names the key or when a key carrying its own
name is first seen. Every read is read-only and traced under `--debug`;
the HID reads give up after 1.5 seconds. Reads are planned only for what USB
position leaves unmatched, so where USB position settles every key (Linux, and
Windows when the device tree resolves both sides) no identity read is sent. A
Windows device-tree position rules out a guess only when the reader's side
resolved too. Where several keys of one make are connected, a key
that doesn't answer is shown as two entries, not guessed at.

---

# Thread B — Capabilities & day-to-day security uses

**Goal of this thread:** for each device, build an accurate picture of what it
can *do* and how a normal person uses it day-to-day to be safer. This research
feeds an EVENTUAL UI deliverable (not built yet): **in-app helper tips,
plain-English explanations of each feature, and generic, vendor-neutral usage
examples** aimed at security-conscious but non-technical users.

Outline for a future Claude (don't over-research — breadth first, depth later):

1. **Enumerate capabilities per device.** FIDO2/WebAuthn (passkeys, resident
   keys), U2F, OATH-TOTP/HOTP, PIV, OpenPGP, Yubico OTP / HMAC challenge-response,
   PIN & policy features. Note which the device actually supports (Solo 2 / NK3
   differ from YubiKey — confirm against the device's own applet responses, not
   a stored table; see the "No device→capability matrix" standing decision in
   `TODO.md`).
2. **Map each capability to a day-to-day use.** Plain language, concrete:
   e.g. "phishing-resistant login to Google/GitHub" (passkey), "log into SSH
   with a hardware key" (FIDO2 SSH), "2FA codes for sites without push" (OATH),
   "sign your git commits" (OpenPGP/PIV). One or two everyday scenarios each.
3. **Frame the security benefit in plain English.** Why it's safer than the
   password/SMS alternative — short, non-alarmist, no jargon.
4. **Collect generic examples** suitable to surface in-UI later. Keep them
   vendor-neutral and provider-agnostic where possible.
5. **Accuracy guardrails.** This becomes user-facing security guidance, so:
   no overpromising ("unhackable"), note real caveats (backup/spare key, what
   happens if lost), and have claims reviewed before they ship in the UI.

## Thread B outcome

A per-device capability → everyday-use → plain-English-benefit table that the
future UI-guidance work draws from. Capture it here as the research progresses.

---

# YubiKey NEO PCB trace review (2026-10-28)

Sources: the [HexView NEO teardown](https://web.archive.org/web/20170926081041/http://www.hexview.com/~scl/neo/) ([original](https://hexview.com/~scl/neo/)) and archived
copies of its [component-side](https://web.archive.org/web/20170917163058im_/http://www.hexview.com/~scl/neo/neo_pcb_s.jpg)
and [reverse-side](https://web.archive.org/web/20170917163058im_/http://www.hexview.com/~scl/neo/neo_pcb_back_s.jpg)
photos (1024 × 408 and 1024 × 411 pixels). The archive supplied the images for
this review when the original image host did not respond. This is one opened
NEO board, not evidence that every hardware revision has identical routing.
HexView's separately titled ["YubiKey 5 Neo" teardown](https://www.hexview.com/~scl/neo5/)
shows a later, different board and is not used as NEO trace evidence here.

## Recording convention

Use the PCB terms **pad**, **track segment**, **via**, **layer**, and **net** as
defined by [KiCad's board format](https://dev-docs.kicad.org/en/file-formats/sexpr-pcb/).
Here, *visible* means a feature can be located in one of the linked photos;
*source-attributed* means HexView assigns it a function; *electrically verified*
would require a continuity or signal measurement. There are no electrically
verified NEO nets in this record. A visible line or adjacent pad is not, by
itself, proof of electrical continuity. Small circular features in the photos
are **via candidates** until both sides or a measurement confirm their role.

The [machine-readable observation map](NEO-PCB-OBSERVATIONS.yaml) identifies the
photo, board face, feature ID, and approximate pixel positions with the image
origin at its upper-left corner. The two faces require a measured reflection
and registration before a front/back overlay; their similar published
orientations do not supply that transform. Keep millimetre dimensions and net
names unset until a physical scale and electrical connectivity are established;
do not manufacture precision from these photos.

## Visible in the two photos

- The component side shows multiple closely spaced perimeter conductors,
  traces and via candidates around the two main IC packages, an LED, and a
  metal-can component above the USB-side IC. [NinjaLab's later annotated
  presentation](https://hardwear.io/wp-content/uploads/2026/04/security-of-Hardware-FIDO2-tokens.pdf)
  identifies the metal-can component as a quartz resonator; the photo does not
  establish its frequency.
  The photo labels the left IC as an NXP A7005 and the right IC as the main
  USB controller. The USB plug is at the right in the published orientation.
- The reverse side shows four USB contact pads, a large circular copper
  feature at the touch position, patterned copper regions, perimeter copper,
  and several via candidates. These are visual locations only: the photo does not prove
  which copper features belong to a particular electrical net.

## Paths identified by the teardown

| Area | Supported claim | Evidence and limit |
|---|---|---|
| NFC antenna at the keyring hole | Antenna conductors run between the keyring hole and the outer PCB edge. | HexView identifies this path and reports that Yubico's CTO acknowledged the routing compromise. The acknowledgement is reported by HexView, not an independently published board drawing. The photos do not establish the complete antenna net or its chip pin. |
| Neck near the USB plug | The NFC loop and vias that HexView calls USB vias cross the area it identified as bending most when the touch contact is pressed while plugged in. A molding hole sits near the connector. | This is HexView's mechanical assessment of one board. It does not identify USB D+, D−, power, or ground by pad or via. HexView reports agreement with Yubico's CTO that ordinary use was unlikely to cause the proposed bending damage. The keyring-hole risk concerned excessive force; HexView also reports that Yubico had seen no warranty returns for that damage at the time. |
| Two-chip boundary | HexView labels the left IC as an NXP A7005 secure controller. The 2017 archived article gives only a tentative LPC13xx estimate for the right IC. [NXP identifies](https://www.nxp.com/company/about-nxp/smarter-world-videos/CONTACT-AND-CONTACTLESS-KEY) the Yubico cloud key's bundle as A7005/LPC11U24, and [NinjaLab identifies](https://hardwear.io/wp-content/uploads/2026/04/security-of-Hardware-FIDO2-tokens.pdf) those two parts on the NEO using this photo. | LPC11U24 is supported by the chip manufacturer and a later research presentation, stronger than HexView's tentative archived estimate. The exact orderable suffix on this opened unit and the electrical protocol between chips remain unverified. |

Search indexing of the [currently published HexView page](https://hexview.com/~scl/neo/)
also reports `LPC11U24FHI33/CP3303` in its parts estimate. The archived 2017
text used above does not contain that revision, and the current host timed out
during direct inspection. Treat the suffix as a later teardown attribution,
not as a marking read independently from the pixels in this review.

## Photo-grounded partial interface map

IDs and pixel positions below refer to the linked photos, **not** to chip pin
numbers or measured board coordinates. The corresponding records in
[NEO-PCB-OBSERVATIONS.yaml](NEO-PCB-OBSERVATIONS.yaml) distinguish a visible
same-face copper segment from a proposed cross-face connection.

| Visible segment or adjacency | What it establishes | Where the map stops |
|---|---|---|
| On the reverse, each of the four USB contacts (R-J1-1 through R-J1-4, top to bottom) has a short copper stub to a round land immediately to its left (R-L1 through R-L4, near x=750). | Four separate contact-to-land segments are visually contiguous in the reverse photo. | The round lands are via **candidates**. The front-face endpoints and the assignment of power, D−, D+, and ground are unverified. |
| A similar four-land pattern (F-L1 through F-L4, x≈743) appears around the same connector hole on the component face. | These sets plausibly represent the two ends of four through-board connections. | Reflection and registration of the photos or continuity measurements are needed to pair individual R-L and F-L lands; apparent holes alone do not prove plating. |
| On the component face, several tracks leave the right edge of the USB-controller package (F-U2) and run toward the connector neck, with round lands along that route. | The USB-side chip is visibly routed into the connector-side region. | This fan cannot be labelled USB D+/D−: the conditional LPC11U24 pin placement below puts those two pins on the chip's *left* edge. No U2 pad is paired with a USB contact. |
| Tracks leave the right and bottom edges of the secure-element package (F-U1) into the passives and round-land cluster between the two ICs. | The secure element has visible local board connections on those package sides. | Tracks disappear under parts and at candidate vias. No particular U1 pad has been linked to a particular U2 pad or signal. |
| The metal-can component above F-U2 has visible tracks descending toward its top-edge pad area. | A component-to-controller route is visible; NinjaLab identifies the part as a quartz resonator. | Its frequency and exact pad-to-pin assignment are not established from this image. |
| The reverse touch disc and perimeter conductors are visible. HexView identifies the latter as an NFC antenna route at the keyring hole and connector neck. | These are board features with plausible functional roles supported by the teardown. | Neither the touch-disc net nor the antenna terminal-to-A7005 route is traced to a chip pad. |

The [LPC11U2x HVQFN33 pin diagram](https://www.nxp.com/docs/en/data-sheet/LPC11U2X.pdf)
offers a **conditional pin placement**, recorded in the YAML. If the photo's
upper-right package dot is pin 1, rotate NXP's top-view diagram 90° clockwise:
the multiplexed PIO0_3/USB_VBUS pin 9 is near the upper left edge; USB_DM pin
13 and USB_DP pin 14 are near the lower-middle left edge; and XTALIN pin 4 and
XTALOUT pin 5 are along the top edge. NXP's bus-powered example does not use
pin 9 as VBUS sensing, so its name alone cannot label a trace. Approximate
image coordinates are in the YAML. This placement is useful for targeting a
continuity test, but it does **not** prove package orientation, a track
assignment, or a USB contact-to-pin path.

NXP's public A700x **short** data sheet describes A7005 interface capabilities
but does not provide a package pin map for this photographed variant. We
therefore cannot use that document to name the A7005 pads reached by the
visible traces.

**Result:** we have a partial *image observation graph*, not a chip-to-chip
netlist. The most concrete mapped chain is each USB contact to its adjacent
round land on the reverse face. The front and reverse four-land groups may
match, but no individual cross-face connection or end-to-end USB, NFC, touch,
power, or inter-chip net is established.

## What the two chips do

| Photo position | Supported functional role | Chip capability, not a measured NEO connection |
|---|---|---|
| Left, toward the keyring hole | NXP A7005-family secure element. [Yubico's NEO manual](https://downloads.yubico.com/support/yubico_YubiKey-Technical-Manual-v3.3_2014.pdf) says the secure element stores and processes cryptographic information and runs Java Card 3.0 / JCOP 2.4.2 R1 applets. | NXP's [A700x family data sheet](https://media.digikey.com/pdf/Data%20Sheets/NXP%20PDFs/A700x_Rev3.1.pdf) specifies a secure CPU, cryptographic coprocessors, I²C/SCI2C, ISO/IEC 7816 contact and ISO/IEC 14443 A contactless interfaces for A7005. This supports a functional connection to NFC and smartcard traffic, but does not identify antenna pins or board traces. The exact A7005 suffix on this photographed unit remains unverified. |
| Right, toward the USB contacts | USB interface controller identified as an LPC11U24 by [NXP](https://www.nxp.com/company/about-nxp/smarter-world-videos/CONTACT-AND-CONTACTLESS-KEY) and [NinjaLab](https://hardwear.io/wp-content/uploads/2026/04/security-of-Hardware-FIDO2-tokens.pdf). Yubico separately describes the NEO's USB-side chip as a non-secure controller. | The [LPC11U2x data sheet](https://www.nxp.com/docs/en/data-sheet/LPC11U2X.pdf) specifies full-speed USB, USART/smartcard, I²C and SSP for the family. The exact orderable variant and which of those interfaces link to the A7005 remain unverified. |

Yubico's NEO manual says that USB CCID passes smartcard commands to the secure
element, while OTP and U2F activity also use that resource; in composite mode,
CCID can temporarily report card removal while another interface is active.
It also says NFC uses an ISO 14443-4 command set matching CCID. This is a
**documented functional path**, not a traced electrical schematic.

## Where charge and signals can flow

This separates **energy paths**, **signal paths**, and **application behavior**.
The first two require closed electrical or electromagnetic paths; a protocol
description alone cannot assign a copper net. The table describes the NEO's
USB and NFC operating modes using the [NEO manual](https://downloads.yubico.com/support/yubico_YubiKey-Technical-Manual-v3.3_2014.pdf),
[NXP LPC11U2x data sheet](https://www.nxp.com/docs/en/data-sheet/LPC11U2X.pdf),
and [NXP A700x data sheet](https://media.digikey.com/pdf/Data%20Sheets/NXP%20PDFs/A700x_Rev3.1.pdf).

| Function | Supported system-level path | Physical route still missing |
|---|---|---|
| USB power | With the NEO plugged in, the host's USB supply must power the controller and active secure element, with current returning to the host through USB ground. An LPC11U24 VDD pin cannot take the USB 5 V supply directly: its data sheet specifies a lower VDD range and illustrates external power conditioning for a bus-powered USB device. | Which contact is VBUS or ground in the photo; regulator or protection part identity; rail voltage; component power pins; ground-return copper. The photo is insufficient to name a specific SOT package as a regulator. |
| USB signalling | The USB D+/D− pair carries host traffic to the LPC11U24 USB device controller. Yubico says the controller translates YubiKey-mode traffic and forwards CCID commands to the secure element; OTP, U2F, and CCID share it. | Contact-to-controller D+/D− nets and the electrical controller-to-A7005 link. The NEO's reported T=1 smartcard behavior does not by itself prove an ISO 7816 wire between the chips. A7005 supports both I²C/SCI2C and ISO 7816; no protocol has been assigned to the board traces. |
| Touch and LED | Yubico describes a solid-state capacitive touch sensor for USB operation; its manual defines LED behavior for enumeration, CCID, and U2F. The large reverse copper disc is at the touch location, and an LED is visible on the component face. | Electrode-to-sensing input, sensing chip, LED driver, series components, and return path. Touch is sensed through a changing electric field, not a DC wire through the finger. |
| NFC, unplugged | Yubico says NFC is enabled when the NEO is not plugged into USB, using ISO 14443 A/4 and the same command set as CCID; its button and LED are disabled in NFC mode. The A7005 has the matching contactless interface. Reader RF energy couples to the board antenna and supports contactless operation; response data is conveyed by RF coupling. | Antenna terminals, matching/tuning parts, RF rectification and power rails on this board, and any path into the USB controller. No conduction path runs from reader to key across the air gap. |
| Local clock and computation | The LPC11U24 uses clock circuitry and the A7005 executes Java Card applets and cryptography. NinjaLab identifies the metal-can part as a quartz resonator. | Resonator frequency and exact pins; internal transistor-level current paths in either IC; any exact board-level activity during a particular command. |

The USB and NFC paths are **different power situations**. A USB capture or APDU
response can establish command behavior but cannot reveal the hidden supply,
ground, antenna, or inter-chip nets. Conversely, the photos show copper but not
whether a particular net is carrying current at a given instant. A complete
board netlist would still be less than a literal map of every electron inside
the two ICs.

HexView's separate [Titan comparison](https://hexview.com/~scl/titan/)
identifies the NEO's secure element as an A7005c, but the suffix is not
confirmed from this board photo. [NinjaLab's A700x study](https://ninjalab.io/wp-content/uploads/2022/05/a_side_journey_to_titan.pdf)
recovered a U2F private key from a Google Titan with an A7005a and listed the
NEO as a likely affected product. The full key-recovery experiment was not
performed on a NEO. This is chip-family security evidence, not a measurement
of this particular key or its traces.

**Not established:** exact antenna turn count and full routing; USB pad-to-pin
connections beyond the adjacent round lands; the function of the patterned
copper and individual vias; touch, power, and inter-chip nets; or the PCB layer
stack. The photos have no scale or registered front/back overlay, and narrow
tracks vanish under packages or at vias. We have not probed this board
electrically. The prior `opensc-tool` APDU
exchange establishes software behavior at the CCID interface, not a physical
trace route. A pin-level map would need aligned close-up images plus continuity
measurements on an opened spare key; hidden routes would need further imaging.

### Measurements needed for a complete board map

1. Record the exact NEO hardware revision, readable IC markings and pin-1
   indicators, then photograph both faces square-on with a scale and common
   fiducials. Register the two images; determine the face reflection from an
   asymmetric physical feature, not the nearly symmetric hole pattern alone.
2. On an **unpowered opened spare**, test each USB contact against the four
   adjacent lands, their candidate opposite-face lands, LPC11U24 USB_DM/USB_DP
   candidate pads, power components, and accessible ground copper. Record
   continuity, non-continuity, and meter polarity; do not assume that a round
   land is a via because it looks drilled.
3. Read component markings and measure USB-mode rail voltages and current with
   suitable probes. Observe the controller-to-secure-element lines during a
   benign CCID exchange to determine whether the actual wiring carries I²C,
   ISO 7816, or another interface. A bus protocol cannot be chosen from the
   chips' feature lists or the host's CCID/T=1 descriptor.
4. Trace both NFC antenna terminals and their matching network to the A7005,
   then compare powered-USB and unplugged-NFC behavior. If tracks disappear
   under packages or inside the board, use X-ray/CT or cross-section imaging
   before declaring a complete netlist.

These steps would establish external PCB nets and operating-mode currents.
They would still not disclose every transistor-level current path inside the
NXP chips; that would require design information or destructive silicon
analysis beyond a PCB teardown.

## Firmware and vulnerability investigation starting point

The NEO has **two distinct firmware targets**: LPC11U24 controller flash and
the A7005 secure element/Java Card applets. A single “NEO firmware dump” is not
one object. The [LPC11U2x data sheet](https://www.nxp.com/docs/en/data-sheet/LPC11U2X.pdf)
documents SWD/ISP and code-read-protection levels that can block flash reads;
the presence and level of protection on this NEO have not been measured. The
public A700x short data sheet does not describe a memory-dump interface.

The most tractable first code target is Yubico's published
[NEO OpenPGP applet source](https://github.com/Yubico/ykneo-openpgp). Its
[2015 advisory](https://developers.yubico.com/ykneo-openpgp/SecurityAdvisory%202015-04-14.html)
documents a PIN-validation flaw in versions below 1.0.9, with some 1.0.9
devices shipped without the fix; Yubico recommends 1.0.10 or later. Its
[2018 OATH advisory](https://www.yubico.com/support/security-advisories/ysa-2018-01/)
is another version-dependent NEO lead. Compare affected and fixed source and
reproduce the authorization boundary in the applet's simulator tests before
looking for new flaws. Neither advisory alone establishes a new vulnerability.

Read-only probes completed so far: the user's FIDO AID SELECT returned
`U2F_V2`/`9000`, then `00 42 00 00` returned `6D00`; on 2026-10-28 an OpenPGP
AID SELECT returned `9000`, then Yubico's documented `00 F1 00 00` applet-version
query returned `6D00` via `opensc-tool`. The FIDO query still gives no version;
the OpenPGP failure did not establish an applet version or security defect.

Further read-only identification on 2026-10-28:

| Path | Observed result | Interpretation |
|---|---|---|
| Yubico [`ykinfo -q -v`](https://developers.yubico.com/yubikey-personalization/Manuals/ykinfo.1.html) | `3.4.9` | NEO product firmware version. |
| `keyroostctl -d list:1 fido info` | CTAPHID protocol v2; HID device version `1.1.0`; `WINK+U2F`; no CTAP2 GetInfo | This interface is U2F-only. The [FIDO specification](https://fidoalliance.org/specs/fido-v2.0-rd-20161004/FIDO-COMPLETE-v2.0-rd-20161004.pdf) calls the three bytes a vendor-defined device version; `1.1.0` must not be substituted for the `3.4.9` product firmware version. |
| Yubico's documented `gpg-connect-agent --hex 'scd apdu 00 f1 00 00' /bye` | `01 00 11 90 00` | OpenPGP applet `1.0.17`. The prior `opensc-tool` `6D00` is an unresolved tool/context difference. This applet version is above Yubico's known-fixed `1.0.10` threshold for the 2015 flaw. |
| OATH applet SELECT `A000000527210101` via CCID | Version TLV `79` gave `1.0.0`; no challenge TLV `74` | Yubico's [OATH code](https://developers.yubico.com/yubikey-manager/API_Documentation/_modules/yubikit/oath.html) identifies `79` as the version and `74` as the access-password challenge. The optional OATH access password is currently unset. |

The firmware and OATH applet versions are below the `3.5.0` and `1.0.1` fixes
in Yubico's 2018 advisory. That is a **known affected version range**, but this
key currently lacks the optional OATH password that the advisory says can be
bypassed. We did not list OATH credentials or test the bypass. No PIN entry,
firmware write, flash erase, or physical debug connection was attempted.

### Investigation plan

The goals are to establish whether either chip's code can be read and to find
reproducible security defects in the NEO's exposed applications. Keep product
firmware, each applet version, public source revision, and physical board
revision separate. A matching version constant does not prove a public source
revision is byte-for-byte what was installed on this key.

1. **Audit the published OpenPGP applet offline.** Pin a source revision whose
   `VERSION` constant is `01 00 11` (for example,
   [`cdf4c83`](https://github.com/Yubico/ykneo-openpgp/blob/cdf4c830e8093b51954d0d5c60384bd4a6a4a9e4/applet/src/openpgpcard/OpenPGPApplet.java#L47)).
   Inventory command dispatch, private-key operations, PIN verification and
   retry handling, authentication state transitions, reset paths, and key
   import. Compare the known 2015 fix with earlier and later code. Record
   specific hypotheses and the tests that would distinguish them; run the
   repository's Java Card simulator tests where its dependencies are available.
   **Output:** source-referenced findings, test results, and an explicit
   source-to-device uncertainty statement.
2. **Validate only credible hypotheses on disposable hardware.** If an offline
   test suggests a flaw, use a separate NEO with test keys and credentials to
   compare expected and observed APDU outcomes. Do not exercise PIN retries,
   credential access, reset, or applet writes on the currently connected key.
   The known OATH password issue can only be meaningfully tested on a spare
   deliberately configured with an OATH access password. **Output:** a minimal
   reproducible trace, affected version, and impact; otherwise record the
   negative result and its scope.
3. **Assess an LPC11U24 firmware read on an opened spare.** Record its exact
   markings and PCB revision, then map SWDIO, SWCLK, RESET, ISP entry, supply,
   and ground by pinout plus continuity. Check logic voltage before attaching
   a probe. Attempt only non-erasing identification and read access; record the
   observed protection boundary. If reads are permitted, take two dumps and
   compare hashes before analyzing the image. Do not use an unlock or mass
   erase command to claim a dump. **Output:** a pin-level evidence map and
   either a verified image or a documented read barrier.
4. **Keep the A7005 as a separate research track.** Map its board interface
   before attributing observed traffic to a chip or applet. Assess the
   published [A7005 side-channel work](https://ninjalab.io/wp-content/uploads/2022/05/a_side_journey_to_titan.pdf)
   against this NEO revision and threat model; full key recovery in that paper
   was demonstrated on a Titan, not on a NEO. **Output:** a justified next
   experiment, without treating a chip-family result as a NEO exploit.

Start with step 1. Steps 2 and 3 need a disposable NEO; the current device's
version and read-only interface observations remain the baseline.

### OpenPGP source audit: first pass (2026-10-28)

The public source was checked out at `cdf4c830e8093b51954d0d5c60384bd4a6a4a9e4`,
where the applet's `VERSION` constant is `01 00 11`. This identifies a useful
source snapshot, **not** a verified image of the applet installed on the NEO.
Compiled that snapshot with OpenJDK 21 and JCardSim 2.2.2 (the JAR matched the
SHA-256 recorded in Yubico's `doc/Building.txt`). All 11 upstream simulator
tests passed. Seven additional, temporary simulator probes passed, bringing
the total to 18. These probes tested denial after a failed PW1 check, loss of
signing authorization after reset or PW1 replacement, PW3 gates on generation,
data writes, import and retry-setting, command chaining, and the secure
messaging encryption-key setter. The temporary probes were run outside this
repository and are not part of its test suite.

| Source area | First-pass result | Evidence limit / next test |
|---|---|---|
| [Sign, decipher and authenticate](https://github.com/Yubico/ykneo-openpgp/blob/cdf4c830e8093b51954d0d5c60384bd4a6a4a9e4/applet/src/openpgpcard/OpenPGPApplet.java#L622) | Each private-key operation checks `pw1.isValidated()` and its required mode. The existing tests cover wrong/missing mode; additional simulator probes found no bypass after a failed PW1 check, card reset, or PW1 replacement. | This is a source/simulator result, not a claim about every command sequence or the installed binary. |
| [Admin operations](https://github.com/Yubico/ykneo-openpgp/blob/cdf4c830e8093b51954d0d5c60384bd4a6a4a9e4/applet/src/openpgpcard/OpenPGPApplet.java#L963) | Key generation, PUT DATA, key import and retry-setting have PW3 checks; tested unauthorized calls were denied in the simulator. | Continue checking edge cases inside authenticated key import and reset paths. |
| [Command chaining](https://github.com/Yubico/ykneo-openpgp/blob/cdf4c830e8093b51954d0d5c60384bd4a6a4a9e4/applet/src/openpgpcard/OpenPGPApplet.java#L429) | The intermediate-header check rejects a mismatch only when **both** INS and P1/P2 differ. A simulated chain with the same INS but changed P1/P2 was accepted; the final unauthorized PUT DATA was still denied by PW3. | A protocol-validation discrepancy is reproduced; no privilege bypass or security impact is established. |
| [Secure-messaging encryption key](https://github.com/Yubico/ykneo-openpgp/blob/cdf4c830e8093b51954d0d5c60384bd4a6a4a9e4/applet/src/openpgpcard/OpenPGPSecureMessaging.java#L167) | `PUT DATA D1` passes the start of the supplied key to `setSessionKeyEncryption`, but that method loads `offset + 16`; the `F4` container path also calls this setter. In JCardSim an admin-authorized 16-byte D1 write returned `9000`, while inspection of the simulator's encryption-key object found 16 zero bytes from the fresh buffer tail, rather than the supplied bytes. A separate direct-method test confirmed it loads the **second** 16 bytes when two distinct 16-byte blocks are present. | This is a reproducible public-source candidate, not a verified flaw on this physical NEO. Assess actual secure-messaging behavior and impact on a disposable NEO before making a device-vulnerability claim. |

A read-only `GET DATA 006E` on the connected NEO returned `9000`; its extended
capabilities advertised secure messaging (`C0` first byte `F0`, algorithm byte
`00`, interpreted using the [OpenPGP Card 2.0 specification](https://gnupg.org/ftp/specs/openpgp-card-2.0.pdf)).
That confirms the feature is advertised by this key, but does not prove
that its implementation contains the public-source key-offset behavior. No
admin PIN, SM key write, or credential operation was attempted on the key.
