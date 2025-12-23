
### Printable room tickets

This module adds support for generating and scanning printable room tickets
that encode room access information in Aztec code barcodes. These tickets
enable offline room joining scenarios where users receive physical tickets
containing room identification and optional invite credentials.

#### Overview

Printable room tickets provide a standardized format for encoding Matrix room
access information in machine-readable barcodes suitable for printing on
physical media. The design draws inspiration from TAP TSI (Technical
Specification for Interoperability) train tickets, adapting similar principles
for Matrix room access.

Use cases include:

- Conference and event registration where attendees receive printed materials
- Educational institutions distributing room access to students
- Organizations providing room access to visitors
- Accessibility scenarios where users may have difficulty with digital-only distribution

#### Barcode specification

Printable room tickets use Aztec codes as specified in ISO/IEC 24778:2008.
Aztec codes are chosen over alternatives like QR codes because they do not
require a quiet zone, have strong error correction capabilities, and are
proven in high-volume ticketing applications.

The barcode parameters are:

| Parameter | Value |
|-----------|-------|
| Symbol type | Aztec Code (full-range symbol) |
| Layers | Minimum 4, maximum 32 (automatically selected based on data size) |
| Error correction | Minimum 23% of symbol capacity (recommended 33% for tickets that may be folded) |
| Module size | Minimum 0.5mm per module when printed |

Matrix room tickets require higher data density than typical transportation
tickets due to the need to encode complete room identifiers with server
hostnames, federation routing information, optional invite tokens, and
human-readable join reasons. Payloads may approach 500-1000 bytes for
fully-featured tickets.

#### Data format

The Aztec code encodes a binary data structure organized into three sections:
a header for format identification, open data containing the ticket information,
and an optional signature for authenticity verification.

##### Header (8 bytes)

| Offset | Length | Field | Description |
|--------|--------|-------|-------------|
| 0 | 6 | Magic | ASCII string `MXROOM` |
| 6 | 1 | Version | Format version (`0x01` for this specification) |
| 7 | 1 | Flags | Bit flags (see below) |

Flag bits:

| Bit | Description |
|-----|-------------|
| 0 | Invite type: 0 = public/knock room, 1 = invite-only with token |
| 1 | Join reason included: 0 = no, 1 = yes |
| 2 | Expiry timestamp included: 0 = no, 1 = yes |
| 3 | Signature included: 0 = no, 1 = yes |
| 4-7 | Reserved (must be 0) |

##### Open data (variable length)

The open data section uses tag-length-value (TLV) encoding for flexibility
and forward compatibility. Each TLV entry is encoded as:

- 1 byte: Tag
- 2 bytes: Length (big-endian, number of bytes in value)
- N bytes: Value

| Tag | Name | Required | Description |
|-----|------|----------|-------------|
| `0x01` | Room ID | Yes | The room ID (e.g., `!roomid:server.example`) encoded as UTF-8 |
| `0x02` | Via Servers | Yes | Comma-separated list of servers for federation routing |
| `0x03` | Room Name | No | Human-readable room name for display on the ticket |
| `0x04` | Invite Token | Conditional | For invite-only rooms, the invite token or code |
| `0x05` | Join Reason | No | Pre-filled reason for joining/knocking, UTF-8 encoded, max 500 bytes |
| `0x06` | Expiry | Conditional | Unix timestamp (4 bytes, big-endian) after which the ticket is invalid |
| `0x07` | Issuer MXID | No | Matrix ID of the user who created the ticket |
| `0x08` | Issuer Display Name | No | Display name of the issuer for printing on the ticket |
| `0x09` | Room Avatar Hash | No | First 8 bytes of SHA-256 hash of room avatar for verification |
| `0x0A` | Ticket ID | No | Unique identifier for this ticket (for tracking/revocation) |

##### Signature (optional, 64 bytes when present)

When the signature flag is set, the ticket includes a digital signature for
authenticity verification. The signature is computed over the header and open
data sections using Ed25519, with the signing key being the room creator's or
an authorized issuer's signing key.

#### Client behaviour

When a Matrix client scans a room ticket barcode, it should:

1. Parse the header and verify the magic bytes (`MXROOM`) and version
2. Decode the TLV entries from the open data section
3. If an expiry timestamp is present and the current time exceeds it, display
   an error indicating the ticket has expired
4. If a signature is present, verify it against known trusted issuers
   (implementation-defined)
5. Attempt to join the room using the room ID and via servers:
   - For public rooms: Send a join request directly
   - For knock rooms: Send a knock request, including the join reason if present
   - For invite-only rooms with a token: Use the token to claim the invite, then join

If the join reason field is present, clients should pre-populate the reason
field in the join or knock request with this value, allowing the user to
modify it before sending.

#### Paper layout specifications

Unlike ICAO Doc 9303/IATA BCBP and TAP TSI standards which allow considerable
flexibility in visual layout, this specification defines strict layout zones
and element positioning. This ensures Matrix room tickets are immediately
recognizable as tickets at a glance.

All coordinates are specified as (X, Y) from the top-left corner of the ticket,
with dimensions as Width x Height.

##### Large format (210mm x 74mm)

This format is compatible with standard ticket printers used at transportation
hubs and event venues. The dimensions align with TAP TSI railway ticket
standards and ICAO boarding pass specifications.

**Barcode zone:**

| Element | Position (X, Y) | Dimensions |
|---------|-----------------|------------|
| Aztec code | (5mm, 12.25mm) | 49.5mm x 49.5mm |

**Text zone** (starts at X=60mm):

| Element | Position (X, Y) | Dimensions | Font |
|---------|-----------------|------------|------|
| Room name | (60mm, 8mm) | 140mm x 18mm | Bold, 14-18pt |
| Room ID | (60mm, 28mm) | 140mm x 10mm | Monospace, 8-10pt |
| Issuer display name | (60mm, 40mm) | 90mm x 10mm | Regular, 9-11pt |
| Expiry date/time | (155mm, 40mm) | 50mm x 10mm | Regular, 9-11pt |
| Description | (60mm, 52mm) | 140mm x 18mm | Regular, 8-10pt |

##### Small format (54mm x 89mm)

This format matches standard business card dimensions and is suitable for
handheld thermal printers and badge printers.

**Barcode zone:**

| Element | Position (X, Y) | Dimensions |
|---------|-----------------|------------|
| Aztec code | (2.25mm, 5mm) | 49.5mm x 49.5mm |

**Text zone** (starts at Y=57mm):

| Element | Position (X, Y) | Dimensions | Font |
|---------|-----------------|------------|------|
| Room name | (2mm, 57mm) | 50mm x 12mm | Bold, 10-12pt |
| Room ID | (2mm, 70mm) | 50mm x 8mm | Monospace, 7-8pt |
| Issuer/Expiry | (2mm, 79mm) | 50mm x 8mm | Regular, 7-8pt |

#### Printing requirements

To ensure reliable scanning:

| Requirement | Value |
|-------------|-------|
| Resolution | Minimum 203 DPI (8 dots/mm), recommended 300 DPI |
| Module mapping | Each barcode module should map to an integer number of printer dots |
| Contrast | Minimum 70% contrast ratio between dark and light modules |
| Paper | Any paper stock suitable for the printing technology |

#### Security considerations

##### Invite token exposure

Tickets containing invite tokens for private rooms represent bearer credentials.
Anyone who obtains the ticket can potentially join the room. Mitigations include:

- Using single-use tokens that are invalidated after first use
- Setting short expiry times on tickets
- Using the ticket ID field to enable server-side revocation
- For high-security rooms, requiring additional authentication after scanning

##### Signature trust

The optional signature mechanism requires clients to maintain a list of trusted
issuer keys. The specification does not define how this trust is established.
Implementations might use room power levels, organization-specific key
distribution, or integration with existing Matrix cross-signing infrastructure.

##### Privacy considerations

Tickets may contain personally identifiable information (issuer MXID, display
names). Users creating tickets should be aware that this information will be
visible to anyone who views the ticket. The room avatar hash field is
intentionally truncated to 8 bytes to prevent the ticket from serving as a
covert channel for arbitrary data.
