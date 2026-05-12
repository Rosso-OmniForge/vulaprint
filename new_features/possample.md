/**
 * ============================================================
 *  ESC/POS Raw Command Builder — 72mm USB Thermal Slip Printer
 * ============================================================
 *  Printer Specs:
 *    Interface    : USB
 *    Speed        : 260 mm/s
 *    Width        : 72mm  (~576 dots @ 203dpi)
 *    Font A/B     : 48 / 64 chars per line
 *    Default Font : Font A
 *    Density      : Level 5 (Max 8)
 *    Cutter       : Yes (full + partial)
 *    Beeper       : Yes
 *    Barcodes     : UPC-A, EAN-13, CODE128, CODE93, CODE39
 *    2D Codes     : QR Code, PDF417, DataMatrix
 *    NV Logo      : Yes
 * ============================================================
 *  Usage (Node.js):
 *    const { buildReceipt } = require('./escpos-slip');
 *    const usb = require('usb');                 // npm i usb
 *    // ---- OR ----
 *    const { createWriteStream } = require('fs');// raw USB file on Linux
 *    const port = createWriteStream('/dev/usb/lp0');
 *    port.write(buildReceipt(receiptData));
 * ============================================================
 */

'use strict';

// ─── ESC/POS COMMAND CONSTANTS ────────────────────────────────────────────────

const ESC = 0x1b;
const GS  = 0x1d;
const FS  = 0x1c;
const LF  = 0x0a;
const CR  = 0x0d;
const NUL = 0x00;

const CMD = {
  // ── Printer Control ──────────────────────────────────────
  INIT:             [ESC, 0x40],          // ESC @ — Reset printer
  FEED_LINE:        [LF],                 // Line feed (print + advance)
  FEED_N_LINES:     (n) => [ESC, 0x64, n],// ESC d n — Feed n lines

  // ── Text Alignment ───────────────────────────────────────
  ALIGN_LEFT:       [ESC, 0x61, 0x00],   // ESC a 0
  ALIGN_CENTER:     [ESC, 0x61, 0x01],   // ESC a 1
  ALIGN_RIGHT:      [ESC, 0x61, 0x02],   // ESC a 2

  // ── Font Selection ───────────────────────────────────────
  FONT_A:           [ESC, 0x4d, 0x00],   // ESC M 0 — 48 chars/line (default)
  FONT_B:           [ESC, 0x4d, 0x01],   // ESC M 1 — 64 chars/line (smaller)

  // ── Text Style ───────────────────────────────────────────
  BOLD_ON:          [ESC, 0x45, 0x01],   // ESC E 1
  BOLD_OFF:         [ESC, 0x45, 0x00],   // ESC E 0
  UNDERLINE_ON:     [ESC, 0x2d, 0x01],   // ESC - 1
  UNDERLINE_OFF:    [ESC, 0x2d, 0x00],   // ESC - 0
  INVERT_ON:        [GS,  0x42, 0x01],   // GS B 1 — white-on-black
  INVERT_OFF:       [GS,  0x42, 0x00],   // GS B 0

  // ── Character Size ───────────────────────────────────────
  // ESC ! n  bits: 0=double width, 4=double height, 3=underline
  SIZE_NORMAL:      [ESC, 0x21, 0x00],   // Normal (1x1)
  SIZE_TALL:        [ESC, 0x21, 0x10],   // Double height (1x2)
  SIZE_WIDE:        [ESC, 0x21, 0x20],   // Double width  (2x1)
  SIZE_BIG:         [ESC, 0x21, 0x30],   // Double width + height (2x2)
  SIZE_HUGE:        [GS,  0x21, 0x11],   // GS ! 0x11 — 2x width + 2x height

  // ── Character Spacing ────────────────────────────────────
  CHAR_SPACING:     (n) => [ESC, 0x20, n],// ESC SP n — Extra char spacing

  // ── Line Spacing ─────────────────────────────────────────
  LINE_SPACING_DEF: [ESC, 0x32],         // ESC 2 — Default line spacing (~30 dots)
  LINE_SPACING:     (n) => [ESC, 0x33, n],// ESC 3 n — Set line spacing (dots)

  // ── Cutter ───────────────────────────────────────────────
  CUT_FULL:         [GS, 0x56, 0x41, 0x00], // GS V A 0 — Full cut
  CUT_PARTIAL:      [GS, 0x56, 0x42, 0x01], // GS V B 1 — Partial cut (leave 1 point)

  // ── Beeper ───────────────────────────────────────────────
  // ESC ( A  pL pH  n  t  (n=beep count 1-9, t=duration 1-9 × 100ms)
  BEEP:             (count = 1, duration = 2) => [ESC, 0x28, 0x41, 0x04, 0x00, 0x31, count, duration, 0x00],

  // ── Print Density ────────────────────────────────────────
  // GS ( E  pL pH  fn  n   (fn=5 set density, n=0-8, 5=default)
  DENSITY:          (level = 5) => [GS, 0x28, 0x45, 0x03, 0x00, 0x05, level, 0x00],
};

// ─── HELPER: Build a raw Buffer from mixed arrays / strings ──────────────────

function buildBuffer(...parts) {
  const chunks = [];
  for (const part of parts) {
    if (typeof part === 'string') {
      chunks.push(Buffer.from(part, 'ascii'));
    } else if (Array.isArray(part)) {
      chunks.push(Buffer.from(part));
    } else if (Buffer.isBuffer(part)) {
      chunks.push(part);
    }
  }
  return Buffer.concat(chunks);
}

// ─── HELPER: Pad / truncate string to exact width ────────────────────────────

function padLeft(str, width)  { return String(str).padStart(width); }
function padRight(str, width) { return String(str).padEnd(width); }

/**
 * Format a two-column line (label left, value right).
 * Total line width = 48 chars (Font A).
 */
function col2(label, value, width = 48) {
  const l = String(label);
  const v = String(value);
  const spaces = width - l.length - v.length;
  return l + ' '.repeat(Math.max(1, spaces)) + v;
}

/**
 * Format a three-column line: qty | description | price
 * e.g.  " 2  Widget Pro          R  49.99"
 */
function col3(qty, desc, price, width = 48) {
  const q = String(qty).padStart(3);
  const p = String(price).padStart(10);
  const d = padRight(desc, width - q.length - p.length - 2);
  return `${q}  ${d}${p}`;
}

/** Repeat a character n times */
function line(char = '-', n = 48) { return char.repeat(n); }

// ─── BARCODE COMMANDS ────────────────────────────────────────────────────────

/**
 * Print a 1D barcode.
 * @param {string} data    — barcode content
 * @param {string} type    — 'CODE128' | 'CODE39' | 'CODE93' | 'EAN13' | 'UPCA'
 * @param {object} opts
 *   height   {number}  — bar height in dots (default 80)
 *   width    {number}  — bar width 2-6 (default 3)
 *   hriPos   {number}  — HRI text: 0=none 1=above 2=below 3=both (default 2)
 *   hriFont  {number}  — 0=Font A  1=Font B
 */
function barcode(data, type = 'CODE128', opts = {}) {
  const { height = 80, width = 3, hriPos = 2, hriFont = 0 } = opts;

  const typeMap = {
    'UPCA':    65,
    'UPCE':    66,
    'EAN13':   67,
    'EAN8':    68,
    'CODE39':  69,
    'ITF':     70,
    'CODE128': 73,
    'CODE93':  72,
  };

  const m = typeMap[type.toUpperCase()] ?? 73;

  const parts = [
    Buffer.from([GS, 0x68, height]),     // GS h — bar height
    Buffer.from([GS, 0x77, width]),      // GS w — bar width
    Buffer.from([GS, 0x48, hriPos]),     // GS H — HRI position
    Buffer.from([GS, 0x66, hriFont]),    // GS f — HRI font
    // GS k m n d1..dn  (type >= 65 uses length-prefixed form)
    Buffer.from([GS, 0x6b, m, data.length]),
    Buffer.from(data, 'ascii'),
  ];

  return Buffer.concat(parts);
}

// ─── QR CODE COMMAND ─────────────────────────────────────────────────────────

/**
 * Print a QR Code using GS ( k commands.
 * @param {string} data    — content to encode
 * @param {number} size    — module size 1-16 (default 4)
 * @param {number} errCorr — error correction: 48=L 49=M 50=Q 51=H (default 49=M)
 */
function qrCode(data, size = 4, errCorr = 49) {
  const len    = data.length + 3;
  const pL     = len & 0xff;
  const pH     = (len >> 8) & 0xff;

  return Buffer.concat([
    // 1. Set model (model 2 = standard QR)
    Buffer.from([GS, 0x28, 0x6b, 0x04, 0x00, 0x31, 0x41, 0x32, 0x00]),
    // 2. Set module size
    Buffer.from([GS, 0x28, 0x6b, 0x03, 0x00, 0x31, 0x43, size]),
    // 3. Set error correction level
    Buffer.from([GS, 0x28, 0x6b, 0x03, 0x00, 0x31, 0x45, errCorr]),
    // 4. Store data
    Buffer.from([GS, 0x28, 0x6b, pL, pH, 0x31, 0x50, 0x30]),
    Buffer.from(data, 'ascii'),
    // 5. Print stored symbol
    Buffer.from([GS, 0x28, 0x6b, 0x03, 0x00, 0x31, 0x51, 0x30]),
  ]);
}

// ─── NV LOGO PRINT ───────────────────────────────────────────────────────────

/**
 * Print a previously downloaded NV logo (key = 1 by default).
 * The logo must have been downloaded once using FS q (see downloadNVLogo()).
 * @param {number} key   — NV image key (1-255)
 * @param {number} scale — 0=normal 1=double-width 2=double-height 3=both
 */
function printNVLogo(key = 1, scale = 0) {
  // FS p  key  scale
  return Buffer.from([FS, 0x70, key, scale]);
}

/**
 * Download a 1-bit raster image into NV memory (survives power cycle).
 * This only needs to run ONCE (e.g. during printer setup / first install).
 *
 * @param {number}   key      — storage key (1-255)
 * @param {number}   widthBytes — image width in BYTES (bits = pixels; must be multiple of 8)
 * @param {number}   height    — image height in dots
 * @param {Buffer}   pixels    — raw 1-bit raster data (widthBytes × height bytes)
 *                               1 = black dot, 0 = white
 *
 * Example: a 200px × 80px logo
 *   widthBytes = 200 / 8 = 25
 *   pixels = Buffer of 25 * 80 = 2000 bytes
 *
 * To convert a PNG to this format in Node.js:
 *   npm i jimp
 *   const Jimp = require('jimp');
 *   const img  = await Jimp.read('logo.png');
 *   img.resize(200, 80).greyscale().contrast(0.5);
 *   const widthBytes = Math.ceil(img.bitmap.width / 8);
 *   const pixels = Buffer.alloc(widthBytes * img.bitmap.height);
 *   img.scan(0, 0, img.bitmap.width, img.bitmap.height, (x, y, idx) => {
 *     const brightness = img.bitmap.data[idx];   // R channel after greyscale
 *     if (brightness < 128) {                     // dark pixel → set bit
 *       const byteIndex = y * widthBytes + Math.floor(x / 8);
 *       pixels[byteIndex] |= (0x80 >> (x % 8));
 *     }
 *   });
 */
function downloadNVLogo(key, widthBytes, height, pixels) {
  // FS q  n  [xL xH yL yH data...]×n
  const xL = widthBytes & 0xff;
  const xH = (widthBytes >> 8) & 0xff;
  const yL = height & 0xff;
  const yH = (height >> 8) & 0xff;

  return Buffer.concat([
    Buffer.from([FS, 0x71, key]),        // FS q  n
    Buffer.from([xL, xH, yL, yH]),      // dimensions
    pixels,                              // raw pixel data
  ]);
}

// ─── RASTER IMAGE (one-shot, not stored) ─────────────────────────────────────

/**
 * Print a raster image inline (not stored in NV).
 * Use for receipts where the logo changes or NV memory is unavailable.
 *
 * @param {number} widthBytes — image width in bytes
 * @param {number} height     — image height in dots
 * @param {Buffer} pixels     — 1-bit raster data
 * @param {number} mode       — 0=normal 1=double-width 2=double-height 3=both
 */
function printRasterImage(widthBytes, height, pixels, mode = 0) {
  const xL = widthBytes & 0xff;
  const xH = (widthBytes >> 8) & 0xff;
  const yL = height & 0xff;
  const yH = (height >> 8) & 0xff;
  // GS v 0  mode  xL xH  yL yH  data
  return Buffer.concat([
    Buffer.from([GS, 0x76, 0x30, mode, xL, xH, yL, yH]),
    pixels,
  ]);
}

// ─── MAIN RECEIPT BUILDER ────────────────────────────────────────────────────

/**
 * Build a complete POS slip as a raw Buffer ready to send to the printer.
 *
 * @param {object} data
 *   logoKey      {number|null}  — NV logo key (null = skip logo)
 *   company      {object}       — { name, tagline, regNo, vatNo, address, phone, email, website }
 *   cashier      {string}       — cashier / till name
 *   invoiceNo    {string}       — receipt / invoice number
 *   date         {string}       — formatted date string
 *   time         {string}       — formatted time string
 *   items        {Array}        — [{ qty, description, unitPrice, total }]
 *   subtotal     {number}
 *   vatRate      {number}       — e.g. 0.15 for 15%
 *   vatAmount    {number}
 *   total        {number}
 *   tender       {number|null}  — cash tendered (null = skip)
 *   change       {number|null}  — change due
 *   paymentMethod{string}       — 'CASH' | 'CARD' | 'SPLIT' etc.
 *   barcode      {string|null}  — invoice barcode value (CODE128)
 *   qrData       {string|null}  — QR code content (e.g. payment URL)
 *   footer       {string[]}     — extra footer lines
 *   beepOnPrint  {boolean}
 */
function buildReceipt(data) {
  const {
    logoKey       = 1,
    company       = {},
    cashier       = 'Cashier 01',
    invoiceNo     = 'INV-000001',
    date          = new Date().toLocaleDateString('en-ZA'),
    time          = new Date().toLocaleTimeString('en-ZA'),
    items         = [],
    subtotal      = 0,
    vatRate       = 0.15,
    vatAmount     = 0,
    total         = 0,
    tender        = null,
    change        = null,
    paymentMethod = 'CASH',
    barcodeValue  = null,
    qrData        = null,
    footer        = [],
    beepOnPrint   = true,
  } = data;

  const FMT   = { currency: (n) => `R ${Number(n).toFixed(2)}` };
  const W     = 48;   // Font A chars per line
  const DLINE = line('=', W);
  const SLINE = line('-', W);

  const buf = [];
  const p   = (...parts) => buf.push(buildBuffer(...parts));  // push raw

  // ── 1. INIT + DENSITY ─────────────────────────────────────────────────────
  p(CMD.INIT);
  p(CMD.DENSITY(5));           // set density level 5 (default for this printer)
  p(CMD.LINE_SPACING_DEF);

  // ── 2. OPTIONAL BEEP ──────────────────────────────────────────────────────
  if (beepOnPrint) p(CMD.BEEP(1, 1));

  // ── 3. LOGO ───────────────────────────────────────────────────────────────
  if (logoKey !== null) {
    p(CMD.ALIGN_CENTER);
    p(printNVLogo(logoKey, 0));    // scale 0 = normal
    p(CMD.FEED_N_LINES(1));
  }

  // ── 4. COMPANY HEADER ─────────────────────────────────────────────────────
  p(CMD.ALIGN_CENTER);
  p(CMD.BOLD_ON, CMD.SIZE_BIG);
  p(company.name ?? 'MY COMPANY', '\n');
  p(CMD.SIZE_NORMAL, CMD.BOLD_OFF);

  if (company.tagline) {
    p(CMD.FONT_B);
    p(company.tagline, '\n');
    p(CMD.FONT_A);
  }

  p(CMD.FEED_N_LINES(1));
  p(CMD.FONT_B);
  if (company.address)  p(company.address,  '\n');
  if (company.phone)    p(`Tel: ${company.phone}`, '\n');
  if (company.email)    p(company.email,    '\n');
  if (company.website)  p(company.website,  '\n');
  p(CMD.FONT_A);

  p(CMD.ALIGN_LEFT);
  p(CMD.FEED_N_LINES(1));
  p(DLINE, '\n');

  // ── 5. VAT / REG NUMBERS ─────────────────────────────────────────────────
  p(CMD.FONT_B);
  if (company.regNo) p(col2('Reg No:', company.regNo, W), '\n');
  if (company.vatNo) p(col2('VAT Reg No:', company.vatNo, W), '\n');
  p(CMD.FONT_A);
  p(SLINE, '\n');

  // ── 6. INVOICE / RECEIPT META ─────────────────────────────────────────────
  p(col2('Invoice:', invoiceNo, W), '\n');
  p(col2('Date:', date,       W), '\n');
  p(col2('Time:', time,       W), '\n');
  p(col2('Cashier:', cashier, W), '\n');
  p(col2('Payment:', paymentMethod, W), '\n');
  p(DLINE, '\n');

  // ── 7. COLUMN HEADERS ─────────────────────────────────────────────────────
  p(CMD.BOLD_ON);
  p(col3('QTY', 'DESCRIPTION', 'AMOUNT', W), '\n');
  p(CMD.BOLD_OFF);
  p(SLINE, '\n');

  // ── 8. LINE ITEMS ─────────────────────────────────────────────────────────
  for (const item of items) {
    const { qty, description, unitPrice, total: itemTotal } = item;
    // Main line: qty | description | line total
    p(col3(qty, description, FMT.currency(itemTotal), W), '\n');
    // Sub-line: unit price (indented, Font B)
    p(CMD.FONT_B);
    p(`    @ ${FMT.currency(unitPrice)} each`, '\n');
    p(CMD.FONT_A);
  }

  p(DLINE, '\n');

  // ── 9. TOTALS ─────────────────────────────────────────────────────────────
  const vatLabel = `VAT (${(vatRate * 100).toFixed(0)}%)`;

  p(col2('Subtotal (excl. VAT):', FMT.currency(subtotal), W), '\n');
  p(col2(vatLabel + ':', FMT.currency(vatAmount), W), '\n');
  p(SLINE, '\n');

  // TOTAL — big & bold
  p(CMD.BOLD_ON, CMD.SIZE_WIDE);
  p(col2('TOTAL:', FMT.currency(total), W), '\n');
  p(CMD.SIZE_NORMAL, CMD.BOLD_OFF);
  p(SLINE, '\n');

  // Tender / change
  if (tender !== null) {
    p(col2('Cash Tendered:', FMT.currency(tender),  W), '\n');
    p(CMD.BOLD_ON);
    p(col2('Change Due:', FMT.currency(change ?? 0), W), '\n');
    p(CMD.BOLD_OFF);
  }

  p(DLINE, '\n');

  // ── 10. INVOICE BARCODE ───────────────────────────────────────────────────
  if (barcodeValue) {
    p(CMD.ALIGN_CENTER);
    p(CMD.FEED_N_LINES(1));
    p(barcode(barcodeValue, 'CODE128', { height: 60, width: 2, hriPos: 2 }));
    p(CMD.FEED_N_LINES(1));
    p(CMD.ALIGN_LEFT);
  }

  // ── 11. QR CODE ───────────────────────────────────────────────────────────
  if (qrData) {
    p(CMD.ALIGN_CENTER);
    p(CMD.FONT_B, 'Scan to pay / view invoice online', '\n', CMD.FONT_A);
    p(CMD.FEED_N_LINES(1));
    p(qrCode(qrData, 4, 49));
    p(CMD.FEED_N_LINES(1));
    p(CMD.ALIGN_LEFT);
  }

  // ── 12. FOOTER ────────────────────────────────────────────────────────────
  p(CMD.ALIGN_CENTER);
  p(CMD.FONT_B);
  for (const footerLine of footer) {
    p(footerLine, '\n');
  }
  p(CMD.FONT_A);

  p(CMD.FEED_N_LINES(1));
  p(CMD.BOLD_ON);
  p('** Thank you for your business! **', '\n');
  p(CMD.BOLD_OFF);
  p(CMD.FEED_N_LINES(1));

  // ── 13. PAPER FEED + CUT ──────────────────────────────────────────────────
  p(CMD.FEED_N_LINES(4));    // feed paper before cut
  p(CMD.CUT_PARTIAL);        // partial cut (leaves 1-point stub)

  return Buffer.concat(buf);
}

// ─── EXAMPLE USAGE ───────────────────────────────────────────────────────────

const exampleReceipt = {
  logoKey:       1,              // NV logo key (pre-loaded once with downloadNVLogo)
  company: {
    name:        'ACME STORE',
    tagline:     'Quality you can trust',
    address:     '123 Main Street, Sandton, 2196',
    phone:       '+27 11 000 0000',
    email:       'info@acmestore.co.za',
    website:     'www.acmestore.co.za',
    regNo:       '2001/123456/07',
    vatNo:       '4123456789',
  },
  cashier:       'Jane Doe',
  invoiceNo:     'INV-20260512-0042',
  date:          '2026/05/12',
  time:          '14:35:08',
  paymentMethod: 'CARD',

  items: [
    { qty: 2, description: 'Wireless Mouse',    unitPrice: 249.99, total: 499.98 },
    { qty: 1, description: 'USB-C Hub 7-Port',  unitPrice: 349.00, total: 349.00 },
    { qty: 3, description: 'HDMI Cable 2m',     unitPrice:  89.00, total: 267.00 },
    { qty: 1, description: 'Laptop Stand Pro',  unitPrice: 459.99, total: 459.99 },
  ],

  subtotal:      1366.93,        // excl. VAT
  vatRate:       0.15,           // 15% South African VAT
  vatAmount:      205.04,
  total:         1571.97,

  tender:        null,           // card payment — no cash tender/change needed
  change:        null,

  barcodeValue:  'INV-20260512-0042',
  qrData:        'https://acmestore.co.za/invoice/INV-20260512-0042',

  footer: [
    'All goods remain property of ACME STORE',
    'until payment is received in full.',
    'Returns accepted within 30 days with receipt.',
    'E&OE',
  ],

  beepOnPrint: true,
};

// Print to USB device on Linux / macOS:
//   const fs = require('fs');
//   const receipt = buildReceipt(exampleReceipt);
//   fs.writeFileSync('/dev/usb/lp0', receipt);

// Or with the 'usb' npm package:
//   const usb = require('usb');
//   const device = usb.findByIds(VENDOR_ID, PRODUCT_ID);
//   device.open();
//   const iface = device.interfaces[0];
//   iface.claim();
//   const endpoint = iface.endpoints.find(e => e.direction === 'out');
//   endpoint.transfer(buildReceipt(exampleReceipt), (err) => device.close());

module.exports = {
  CMD,
  buildBuffer,
  buildReceipt,
  barcode,
  qrCode,
  printNVLogo,
  downloadNVLogo,
  printRasterImage,
  col2,
  col3,
  line,
  exampleReceipt,
};