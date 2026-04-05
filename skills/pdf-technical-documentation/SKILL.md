---
name: pdf-technical-documentation
description: Use when extracting register tables, bit-field descriptions, or hardware specifications from technical PDF datasheets, comparing document revisions for errata changes, generating C headers from register descriptions, or validating code against vendor documentation
---

# PDF Technical Documentation

## Overview

Extraction, parsing, validation, and generation workflow for technical PDF documentation — datasheets, reference manuals, errata sheets, and firmware specifications. Enables reliable round-tripping between vendor documentation and code artifacts.

## When to Use

- Extracting register maps, bit-field tables, or memory map descriptions from PDF datasheets
- Generating C header files from register descriptions in vendor documentation
- Validating that driver code matches datasheet offsets, masks, and access patterns
- Comparing document revisions to detect errata, deprecated registers, or changed reset values
- Converting PDF tables to structured data (CSV/JSON) for tooling consumption
- OCR-ing scanned datasheets or legacy printed documentation
- Cross-referencing code against vendor specifications before firmware releases

## When NOT to Use

- Reading prose documentation (use general PDF reading tools instead)
- Non-technical PDFs (presentations, marketing materials)
- Documentation that already has a machine-readable source (YAML, RST, JSON)
- Single-page register summaries that are simple enough to transcribe by hand

## 1. PDF Extraction Pipeline

### pdftotext — Table-Aware Text Extraction

```bash
# Layout mode preserves column alignment for register tables
pdftotext -layout input.pdf output.txt

# Extract specific pages containing register chapter
pdftotext -layout -f 42 -l 58 input.pdf registers.txt

# Raw mode (no layout) for full-text search indexing
pdftotext -raw input.pdf fulltext.txt
```

Use `-layout` when extracting register tables — it preserves column alignment, making regex-based parsing reliable.

### pdfgrep — Search Across Documents

```bash
# Search with line numbers across all datasheets in a directory
pdfgrep -n "GPIO_CTL" datasheets/*.pdf

# Case-insensitive search for register name variants
pdfgrep -in "uart[._]?txdata" reference-manual.pdf

# Count matches per page
pdfgrep -c "RST_VAL" chip-spec.pdf
```

### mutool — Extract Embedded Images/Diagrams

```bash
# Extract all images from a PDF (block diagrams, timing diagrams)
mutool extract input.pdf

# Extract images from specific pages
mutool extract -p 12-15 input.pdf

# Render page to PNG for visual inspection of timing diagrams
mutool draw -o timing-diagram.png -r 300 -p 112 input.pdf
```

### ocrmypdf — OCR for Scanned Datasheets

```bash
# Force OCR on a scanned datasheet, output searchable PDF
ocrmypdf --force-ocr --language eng scanned-datasheet.pdf ocr-output.pdf

# OCR with deskew for crooked scans
ocrmypdf --force-ocr --deskew --language eng scanned.pdf deskewed.pdf

# Then extract text from OCR'd result
pdftotext -layout deskewed.pdf deskewed.txt
```

### pdfinfo — Document Metadata

```bash
# Check revision, creation date, page count
pdfinfo datasheet.pdf

# Output key fields:
# Title:          IMX6ULLRM Rev. 2
# CreationDate:   Thu Jan 15 10:00:00 2024
# Pages:          4152
```

Use `pdfinfo` first to confirm document revision matches the silicon stepping you're targeting.

### pdfplumber — Programmatic Table Extraction

```python
import pdfplumber

def extract_register_table(pdf_path, start_page, end_page):
    registers = []
    with pdfplumber.open(pdf_path) as pdf:
        for page_num in range(start_page - 1, end_page):
            page = pdf.pages[page_num]
            # Tune tolerance for tables with slightly misaligned columns
            tables = page.extract_tables({
                "vertical_strategy": "lines",
                "horizontal_strategy": "lines",
                "snap_tolerance": 4,
                "join_tolerance": 4,
                "edge_min_length": 20,
            })
            for table in tables:
                if not table:
                    continue
                header = table[0]
                if _is_register_header(header):
                    for row in table[1:]:
                        reg = _parse_register_row(header, row)
                        if reg:
                            registers.append(reg)
    return registers

def _is_register_header(row):
    keywords = {"offset", "name", "bits", "field", "access", "reset", "description"}
    cell_text = " ".join(c.lower() for c in (row or []) if c)
    return len(keywords & set(cell_text.split())) >= 3
```

### pypdf — Metadata, Merge, Split

```python
from pypdf import PdfReader, PdfWriter

reader = PdfReader("reference-manual.pdf")
meta = reader.metadata
print(f"Title: {meta.title}, Author: {meta.author}")

# Extract register chapter pages into separate file
writer = PdfWriter()
for i in range(412, 440):  # zero-indexed
    writer.add_page(reader.pages[i])
writer.write("register-chapter.pdf")

# Merge errata supplement with main datasheet
merger = PdfWriter()
for path in ["datasheet.pdf", "errata.pdf"]:
    merger.append_pages_from_reader(PdfReader(path))
merger.write("combined.pdf")
```

### camelot — Complex Table Extraction

```python
import camelot

# Lattice mode: tables with explicit grid lines (most datasheets)
tables = camelot.read_pdf(
    "datasheet.pdf",
    pages="42-58",
    flavor="lattice",
    line_scale=40,
    copy_text=["v"],  # copy text from spanning cells vertically
)

# Stream mode: tables with only horizontal lines (some vendor formats)
tables_stream = camelot.read_pdf(
    "datasheet.pdf",
    pages="100-105",
    flavor="stream",
    edge_tol=500,
    row_tol=10,
)

# Export to structured data
for i, table in enumerate(tables):
    table.to_csv(f"register_table_{i}.csv")
    table.to_json(f"register_table_{i}.json")
```

## 2. Register Table Parsing

### Table Identification Patterns

Register description tables typically have these column header patterns:

```
Offset | Name | Bits | Field | Access | Reset | Description
0x0000 | CTRL | 31:28 | MODE  | RW    | 0x3   | Operating mode select
0x0000 | CTRL | 7:0   | DIV   | RW    | 0x00  | Clock divider value
```

Common header variants to match:
- `Offset / Address / Register Offset`
- `Name / Bit Name / Field Name / Register Name`
- `Bits / Bit Field / Bit Range / Position`
- `Access / Type / R/W / Access Type`
- `Reset / Reset Value / Default / Default Value / POR`

### Multi-Page Spanning Tables

Datasheet tables often span pages with repeated headers and continuation rows:

```python
import re

def merge_spanning_tables(page_tables):
    merged = []
    seen_headers = set()
    for table in page_tables:
        if not table or not table[0]:
            continue
        header_sig = tuple(c.strip().lower() for c in table[0] if c)
        if header_sig in seen_headers:
            # Skip repeated header row
            merged.extend(table[1:])
        else:
            seen_headers.add(header_sig)
            merged.extend(table)
    return merged
```

### Bit-Field Range Parsing

```python
def parse_bit_range(bit_str):
    """Parse bit-field specifiers into (msb, lsb) tuples.

    Handles: [31:28], 7:0, bit 4, 31:28, Bits 15-8, single bit 3
    """
    bit_str = bit_str.strip().strip("[]")
    patterns = [
        (r"(\d+)\s*[:\-]\s*(\d+)", lambda m: (int(m[1]), int(m[2]))),
        (r"bit\s+(\d+)", lambda m: (int(m[1]), int(m[1]))),
        (r"^(\d+)$", lambda m: (int(m[1]), int(m[1]))),
    ]
    for pat, handler in patterns:
        m = re.match(pat, bit_str, re.IGNORECASE)
        if m:
            return handler(m)
    return None
```

### Access Type Extraction

| Code | Meaning | Write Behavior |
|------|---------|---------------|
| RO | Read-Only | Writes ignored, may trigger bus fault |
| RW | Read-Write | Normal read/write |
| RW1C | Write-1-to-Clear | Writing 1 clears the bit, writing 0 has no effect |
| RW1S | Write-1-to-Set | Writing 1 sets the bit, writing 0 has no effect |
| RW1CS | Read, Write-1-to-Clear, Self-set | Hardware sets, software clears by writing 1 |
| RW0C | Write-0-to-Clear | Writing 0 clears the bit, writing 1 has no effect |
| WO | Write-Only | Reads return undefined value |
| W1 | Write-Once | Writable only on first write after reset |

```python
ACCESS_TYPES = {
    "ro": {"read": True, "write": False, "write_effect": None},
    "rw": {"read": True, "write": True, "write_effect": None},
    "rw1c": {"read": True, "write": True, "write_effect": "clear_on_1"},
    "rw1s": {"read": True, "write": True, "write_effect": "set_on_1"},
    "rw1cs": {"read": True, "write": True, "write_effect": "clear_on_1_hw_set"},
    "rw0c": {"read": True, "write": True, "write_effect": "clear_on_0"},
    "wo": {"read": False, "write": True, "write_effect": None},
    "w1": {"read": False, "write": True, "write_effect": "once"},
}

def parse_access(access_str):
    normalized = access_str.strip().lower().replace("/", "").replace("-", "")
    for key in ACCESS_TYPES:
        if key in normalized:
            return ACCESS_TYPES[key]
    return None
```

### Reset Value Mapping

```python
def parse_reset_value(reset_str, width=32):
    """Parse reset value from datasheet column.

    Handles: 0x00000003, 3, 0b0011, 0, undefined, -, N/A
    """
    reset_str = reset_str.strip()
    if reset_str.lower() in ("-", "n/a", "undefined", "reserved", ""):
        return None
    if reset_str.startswith("0x") or reset_str.startswith("0X"):
        return int(reset_str, 16)
    if reset_str.startswith("0b") or reset_str.startswith("0B"):
        return int(reset_str, 2)
    try:
        return int(reset_str)
    except ValueError:
        return None
```

### Reserved Field Handling Strategy

1. **Never write non-zero values to reserved fields** — always preserve read-modify-write
2. When generating code, emit `_reserved` or `_rsvd` members with access-level warnings
3. Validation must flag any code that writes to reserved bits
4. Treat reset value as "do not assume zero" unless explicitly stated

## 3. C Header Generation

### Register Block Struct Pattern

```c
#ifndef VENDOR_CHIP_UART_H
#define VENDOR_CHIP_UART_H

#include <stdint.h>

/**
 * UART Register Block — Base: 0x4000_0000
 * Reference: DS-IMX6ULL Rev 2, Section 57.4, Page 2841
 * Stepping: 1.2 (silicon rev confirmed 2024-01)
 */

#define UART_BASE_ADDRESS       (0x40000000UL)
#define UART_BASE_OFFSET        (0x00000000UL)

typedef struct {
    volatile uint32_t TXDATA;          /* Offset: 0x0000, Access: WO,  Reset: 0x00000000 */
    volatile uint32_t RXDATA;          /* Offset: 0x0004, Access: RO,  Reset: 0x00000000 */
    volatile uint32_t CTRL;            /* Offset: 0x0008, Access: RW,  Reset: 0x00000003 */
    volatile uint32_t STATUS;          /* Offset: 0x000C, Access: RO,  Reset: 0x00000010 */
    volatile uint32_t INT_EN;          /* Offset: 0x0010, Access: RW,  Reset: 0x00000000 */
    volatile uint32_t INT_STS;         /* Offset: 0x0014, Access: RW1C, Reset: 0x00000000 */
    volatile uint32_t _rsvd[2];        /* Offset: 0x0018-0x001C, Reserved — do not modify */
    volatile uint32_t BAUD_DIV;        /* Offset: 0x0020, Access: RW,  Reset: 0x0000000F */
} UART_RegBlock;

static_assert(sizeof(UART_RegBlock) == 0x24, "UART register block size mismatch");

/* CTRL register bit-fields (Offset 0x0008) */
#define UART_CTRL_MODE_SHIFT     (28U)
#define UART_CTRL_MODE_MASK      (0xFU << UART_CTRL_MODE_SHIFT)
#define UART_CTRL_MODE(val)      (((uint32_t)(val) << UART_CTRL_MODE_SHIFT) & UART_CTRL_MODE_MASK)

#define UART_CTRL_TXEN_SHIFT     (11U)
#define UART_CTRL_TXEN_MASK      (1U << UART_CTRL_TXEN_SHIFT)
#define UART_CTRL_TXEN_ENABLE    (1U)
#define UART_CTRL_TXEN_DISABLE   (0U)

#define UART_CTRL_RXEN_SHIFT     (10U)
#define UART_CTRL_RXEN_MASK      (1U << UART_CTRL_RXEN_SHIFT)
#define UART_CTRL_RXEN_ENABLE    (1U)
#define UART_CTRL_RXEN_DISABLE   (0U)

#define UART_CTRL_DIV_SHIFT      (0U)
#define UART_CTRL_DIV_MASK       (0xFFU << UART_CTRL_DIV_SHIFT)
#define UART_CTRL_DIV(val)       (((uint32_t)(val) << UART_CTRL_DIV_SHIFT) & UART_CTRL_DIV_MASK)

/* MODE field enumerated values */
#define UART_CTRL_MODE_NORMAL    (0x0U)
#define UART_CTRL_MODE_LOOPBACK  (0x1U)
#define UART_CTRL_MODE_ECHO      (0x2U)
#define UART_CTRL_MODE_LPBK_ECHO (0x3U)

/* STATUS register bit-fields (Offset 0x000C) — Read-Only */
#define UART_STS_TXFULL_SHIFT    (3U)
#define UART_STS_TXFULL_MASK     (1U << UART_STS_TXFULL_SHIFT)

#define UART_STS_RXEMPTY_SHIFT   (0U)
#define UART_STS_RXEMPTY_MASK    (1U << UART_STS_RXEMPTY_SHIFT)

/* INT_STS register bit-fields (Offset 0x0014) — Write-1-to-Clear */
#define UART_INTSTS_TXEMPTY_SHIFT  (0U)
#define UART_INTSTS_TXEMPTY_MASK   (1U << UART_INTSTS_TXEMPTY_SHIFT)

/* Warning: Writing reserved bits [31:12] has undefined behavior.
 * Always use read-modify-write when modifying CTRL register.
 */

#endif /* VENDOR_CHIP_UART_H */
```

### Generation Rules

1. **Namespace prefix**: `VENDOR_CHIP_` on all macros and types
2. **Include guards**: `VENDOR_CHIP_<MODULE>_H` format
3. **Volatile**: All register struct members are `volatile uint32_t`
4. **Reserved padding**: Use `_rsvd` with count matching gap, add warning comment
5. **Offset comment**: Every struct member gets offset, access type, and reset value
6. **Mask/shift macros**: `_MASK` and `_SHIFT` for every field, `_SHIFT` is the LSB
7. **Value macros**: Enumerated field values get `#define` with descriptive names
8. **Static assert**: Validate struct size matches expected register block footprint
9. **Endianness**: Document assumed endianness in header comment; generate `[31:0]` bit numbering by default
10. **Access-type comments**: Document RW1C, WO, and other non-standard access in field comments

### Automated Generation Script

```python
import csv
import sys

def generate_header(module_name, prefix, registers, base_addr):
    guard = f"{prefix}_{module_name.upper()}_H"
    lines = [
        f"#ifndef {guard}",
        f"#define {guard}",
        "",
        '#include <stdint.h>',
        "",
        f"#define {prefix}_{module_name.upper()}_BASE ({hex(base_addr)}UL)",
        "",
        f"typedef struct {{",
    ]

    expected_offset = 0
    for reg in registers:
        offset = int(reg["offset"], 16)
        if offset > expected_offset:
            gap = (offset - expected_offset) // 4
            lines.append(f"    volatile uint32_t _rsvd_{hex(expected_offset)}[{gap}]; /* Reserved */")
        access = reg.get("access", "RW")
        reset = reg.get("reset", "0x00000000")
        lines.append(
            f'    volatile uint32_t {reg["name"]}; /* Offset: {reg["offset"]}, Access: {access}, Reset: {reset} */'
        )
        expected_offset = offset + 4

    lines.append(f"}} {prefix}_{module_name}_RegBlock;")
    lines.append("")
    lines.append(f"#endif /* {guard} */")
    return "\n".join(lines)
```

## 4. Cross-Reference Validation

### Offset Verification

```python
def validate_offsets(header_path, datasheet_registers):
    """Compare C struct member offsets against datasheet register map."""
    import re

    with open(header_path) as f:
        source = f.read()

    # Extract struct members with offset comments
    pattern = r'volatile\s+uint32_t\s+(\w+);\s*/\*\s*Offset:\s*(0x[0-9A-Fa-f]+)'
    code_offsets = {name: int(offset, 16) for name, offset in re.findall(pattern, source)}

    errors = []
    for reg in datasheet_registers:
        name = reg["name"]
        expected = int(reg["offset"], 16)
        if name not in code_offsets:
            errors.append(f"MISSING: register {name} (offset {hex(expected)}) not in code")
        elif code_offsets[name] != expected:
            errors.append(
                f"MISMATCH: {name} code={hex(code_offsets[name])} datasheet={hex(expected)}"
            )
    return errors
```

### Bit-Field Mask Verification

Formula: `mask = ((1 << width) - 1) << shift`

```python
def verify_mask(mask_hex, msb, lsb):
    """Verify that a mask value matches the declared bit range."""
    mask = int(mask_hex, 16)
    width = msb - lsb + 1
    expected = ((1 << width) - 1) << lsb
    if mask != expected:
        return f"BAD MASK: {mask_hex} != {hex(expected)} for bits [{msb}:{lsb}]"
    return None

def verify_field_shift_mask(shift, mask_hex, msb, lsb):
    """Verify shift/mask pair against bit range."""
    shift_val = int(shift)
    mask_val = int(mask_hex, 16)
    width = msb - lsb + 1
    if shift_val != lsb:
        return f"BAD SHIFT: {shift} != {lsb} for bits [{msb}:{lsb}]"
    expected_mask = ((1 << width) - 1) << lsb
    if mask_val != expected_mask:
        return f"BAD MASK: {mask_hex} != {hex(expected_mask)} for bits [{msb}:{lsb}]"
    return None
```

### Access Pattern Checks

```python
def audit_access_patterns(source_dir, register_map):
    """Scan source for writes to read-only registers and reads from write-only."""
    import re
    issues = []

    for filepath in Path(source_dir).rglob("*.[ch]"):
        content = filepath.read_text()
        for reg_name, info in register_map.items():
            if info["access"] == "RO":
                # Direct assignment to RO register
                if re.search(rf"\b{reg_name}\s*=\s*[^=]", content):
                    issues.append(f"{filepath}: WRITE to RO register {reg_name}")
                if re.search(rf"\b{reg_name}\s*\|=", content):
                    issues.append(f"{filepath}: RMW on RO register {reg_name}")
            if info["access"] == "WO":
                if re.search(rf"\b\w+\s*=\s*.*{reg_name}\b", content):
                    issues.append(f"{filepath}: READ from WO register {reg_name}")
    return issues
```

### Reserved Bit Write Flagging

```python
def check_reserved_bit_writes(source_dir, reserved_masks):
    """Flag code that writes to reserved bit positions."""
    import re
    issues = []
    for filepath in Path(source_dir).rglob("*.[ch]"):
        content = filepath.read_text()
        for reg_name, mask in reserved_masks.items():
            # Check for direct assignment that doesn't preserve reserved bits
            assignments = re.finditer(
                rf"(\w+)->{reg_name}\s*=\s*(.+?);", content
            )
            for m in assignments:
                rhs = m.group(2)
                # If RHS is a literal value, check for reserved bits
                try:
                    val = int(rhs, 0)
                    if val & mask:
                        issues.append(
                            f"{filepath}:{content[:m.start()].count(chr(10))+1} "
                            f"Writing to reserved bits of {reg_name}: val={hex(val)} "
                            f"reserved_mask={hex(mask)}"
                        )
                except ValueError:
                    pass
    return issues
```

### Full Validation Example

```bash
# Run validation pipeline
python3 validate_registers.py \
    --header include/uart.h \
    --datasheet-parsed registers_uart.json \
    --source-dir src/driver/uart/
```

## 5. Revision Comparison (Errata Tracking)

### Visual Comparison

```bash
# Visual diff between two PDF revisions
diff-pdf --view datasheet-rev1.pdf datasheet-rev2.pdf

# Output visual diff as a new PDF for review
diff-pdf --output-diff diff-output.pdf datasheet-rev1.pdf datasheet-rev2.pdf
```

### Structural Text Diff

```bash
# Extract and diff register chapter text between revisions
pdftotext -layout -f 42 -l 58 datasheet-rev1.pdf rev1-registers.txt
pdftotext -layout -f 42 -l 58 datasheet-rev2.pdf rev2-registers.txt
diff -u rev1-registers.txt rev2-registers.txt > register-changes.diff
```

### Change Classification

```python
def classify_register_changes(old_regs, new_regs):
    """Classify differences between two register map revisions."""
    old_map = {r["offset"]: r for r in old_regs}
    new_map = {r["offset"]: r for r in new_regs}

    changes = {"new": [], "removed": [], "modified": [], "unchanged": []}

    for offset, reg in new_map.items():
        if offset not in old_map:
            changes["new"].append(reg)
        elif reg != old_map[offset]:
            changes["modified"].append({
                "offset": offset,
                "old": old_map[offset],
                "new": reg,
                "diff_fields": _diff_fields(old_map[offset], reg),
            })
        else:
            changes["unchanged"].append(reg)

    for offset, reg in old_map.items():
        if offset not in new_map:
            changes["removed"].append(reg)

    return changes

def _diff_fields(old, new):
    diffs = []
    for key in set(list(old.keys()) + list(new.keys())):
        if old.get(key) != new.get(key):
            diffs.append({"field": key, "old": old.get(key), "new": new.get(key)})
    return diffs
```

### Alert Criteria

When comparing revisions, flag these high-impact changes:

| Change Type | Severity | Action |
|-------------|----------|--------|
| Register removed | CRITICAL | Audit all code referencing this register |
| Register offset changed | CRITICAL | Update all base address macros immediately |
| Reset value changed | HIGH | Verify driver init sequences assume old reset values |
| Bit-field repurposed | HIGH | Rewrite affected logic, test on new silicon |
| Access type changed (RW→RO) | HIGH | Remove write paths in driver |
| New register added | MEDIUM | Evaluate if driver needs to initialize/configure it |
| New reserved bits | LOW | Ensure RMW operations preserve these bits |
| Description typo fix | INFO | No code action needed |

### Errata Summary Generation

```python
def generate_errata_summary(changes, rev_old, rev_new):
    lines = [
        f"# Errata Summary: {rev_old} → {rev_new}",
        "",
        f"Date: {datetime.date.today().isoformat()}",
        "",
    ]

    for reg in changes["removed"]:
        lines.append(f"## REMOVED: {reg['name']} at {reg['offset']}")
        lines.append(f"  **SEVERITY: CRITICAL** — Audit all references")
        lines.append("")

    for change in changes["modified"]:
        lines.append(f"## MODIFIED: {change['offset']} — {change['new'].get('name', 'UNKNOWN')}")
        for diff in change["diff_fields"]:
            severity = _severity_for_field(diff["field"])
            lines.append(f"  - {diff['field']}: {diff['old']} → {diff['new']} [{severity}]")
        lines.append("")

    for reg in changes["new"]:
        lines.append(f"## NEW: {reg['name']} at {reg['offset']}")
        lines.append("")

    return "\n".join(lines)
```

## 6. Multi-Format Document Handling

| Format | Typical Content | Extraction Tool | Notes |
|--------|----------------|-----------------|-------|
| PDF | Datasheets, reference manuals | `pdftotext`, `pdfplumber`, `camelot` | Primary format; use `-layout` mode |
| DOCX | Vendor release notes, errata sheets | `python-docx` | Often contains revision histories and known issues |
| RST/Markdown | Open-source firmware docs (U-Boot, EDK2) | Direct read / `pandoc` | Machine-parseable; preferred when available |
| XLSX/CSV | Pinmux tables, GPIO mux configurations | `openpyxl`, `csv` module | Spreadsheet pinmux tables are common for SoCs |
| HTML | Web-published specs, JEDEC standards | `beautifulsoup4` | May need to handle JavaScript-rendered content |

```python
# DOCX extraction
from docx import Document

doc = Document("release-notes.docx")
for table in doc.tables:
    for row in table.rows:
        cells = [cell.text for cell in row.cells]
        print(" | ".join(cells))

# XLSX pinmux extraction
import openpyxl

wb = openpyxl.load_workbook("pinmux.xlsx")
ws = wb.active
for row in ws.iter_rows(min_row=2, values_only=True):
    pin_name, func0, func1, func2 = row[:4]
    # Process pinmux configuration

# HTML spec extraction
from bs4 import BeautifulSoup
import requests

resp = requests.get("https://www.jedec.org/standards-documents/docs/jesd79-4d")
soup = BeautifulSoup(resp.text, "html.parser")
tables = soup.find_all("table")
```

## 7. Automation Patterns

### Vendor Portal Watch for New Revisions

```bash
# Cron: check vendor portal weekly for updated datasheets
# Compare downloaded PDF against last-known revision
0 2 * * 0 /usr/local/bin/check-datasheet-revisions.sh >> /var/log/datasheet-watch.log 2>&1
```

```python
# check-datasheet-revisions.py
import hashlib
import json
import requests

def check_for_new_revision(product_url, known_hashes_file):
    with open(known_hashes_file) as f:
        known = json.load(f)

    resp = requests.get(product_url)
    current_hash = hashlib.sha256(resp.content).hexdigest()

    if current_hash != known.get("sha256"):
        print(f"NEW REVISION DETECTED for {product_url}")
        print(f"  Old hash: {known.get('sha256', 'none')}")
        print(f"  New hash: {current_hash}")
        # Trigger extraction + diff pipeline
        return True
    return False
```

### Batch PDF-to-CSV/JSON Conversion

```bash
# Convert all register chapter PDFs to structured JSON
for pdf in datasheets/chapter-*.pdf; do
    python3 extract_registers.py "$pdf" > "parsed/$(basename "$pdf" .pdf).json"
done
```

### Auto-Generate Headers from Register Chapters

```bash
# Pipeline: extract → parse → validate → generate
python3 extract_registers.py --pages 42-58 datasheet.pdf > uart_regs.json
python3 parse_bitfields.py uart_regs.json > uart_fields.json
python3 gen_header.py --module uart --prefix VENDOR_CHIP uart_fields.json > include/uart.h
python3 validate_header.py --header include/uart.h --source uart_fields.json
```

### Stepping Documentation Diff Reports

```bash
# Generate diff report between silicon stepping docs
diff-pdf --output-diff stepping-diff.pdf stepping-A2.pdf stepping-B0.pdf
pdftotext -layout stepping-A2.pdf stepping-A2.txt
pdftotext -layout stepping-B0.pdf stepping-B0.txt
diff -u stepping-A2.txt stepping-B0.txt > stepping-A2-to-B0.diff
python3 classify_errata.py stepping-A2-to-B0.diff > errata-report-A2-B0.md
```

### Register Access Audit

```bash
# Full audit: datasheet register map vs driver source code
python3 audit_access.py \
    --register-map parsed/uart_regs.json \
    --source-dir drivers/serial/ \
    --header include/uart.h \
    --report audit-uart.md
```

## Quick Reference

| Task | Tool | Command Pattern |
|------|------|-----------------|
| Extract text preserving layout | `pdftotext` | `pdftotext -layout -f START -l END input.pdf output.txt` |
| Search PDF content | `pdfgrep` | `pdfgrep -n "PATTERN" file.pdf` |
| Extract tables programmatically | `pdfplumber` | `page.extract_tables({"vertical_strategy": "lines"})` |
| Extract complex tables | `camelot` | `camelot.read_pdf("f.pdf", pages="1-5", flavor="lattice")` |
| OCR scanned datasheet | `ocrmypdf` | `ocrmypdf --force-ocr --language eng in.pdf out.pdf` |
| Extract images/diagrams | `mutool` | `mutool extract -p PAGES input.pdf` |
| Visual PDF diff | `diff-pdf` | `diff-pdf --view old.pdf new.pdf` |
| Document metadata | `pdfinfo` | `pdfinfo datasheet.pdf` |
| Merge/split PDF | `pypdf` | `PdfWriter.append_pages_from_reader(PdfReader(path))` |
| Validate masks | Formula | `mask = ((1 << width) - 1) << shift` |

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Using `pdftotext` without `-layout` | Always use `-layout` for register tables to preserve column alignment |
| Ignoring multi-page table headers | Skip repeated header rows when merging spanning tables |
| Writing non-zero to reserved bits | Always use read-modify-write; mask off reserved fields |
| Assuming register offsets are contiguous | Check for gaps; insert `_rsvd` padding to match hardware layout |
| Forgetting `volatile` on register pointers | All memory-mapped register accesses must use `volatile` |
| Using wrong endianness for bit numbering | Confirm datasheet convention (`[MSB:LSB]` vs `[LSB:MSB]`) before generating masks |
| Trusting OCR'd register values without verification | Cross-check OCR results against known reset values and valid ranges |
| Not checking document revision against silicon stepping | Always run `pdfinfo` first and confirm the datasheet matches your target hardware revision |
| Ignoring RW1C access semantics | Use `reg |= BIT` to clear, never `reg = BIT` (which clears all other bits) |
| Assuming reset values are zero | Parse the reset column explicitly; many registers have non-zero POR defaults |
