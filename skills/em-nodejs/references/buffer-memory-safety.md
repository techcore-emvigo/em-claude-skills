# Buffer Safety & Memory Management

Covers: explicit resource release, input truncation, buffer boundary checks,
null-termination, avoiding known-vulnerable functions (strcpy, eval, pickle).

---

# Memory Management & General Coding Practices

Covers: resource lifecycle, buffer safety, null-termination, dangerous functions,
race conditions, numeric precision, dynamic execution prevention, managed code,
dependency review, integrity verification, and secure auto-update.

---

## 1. Explicit Resource Management — Close Everything You Open

Never rely on garbage collection. GC timing is non-deterministic — under load, connections
stay open far longer than expected, exhausting pools and causing cascading failures.

```typescript
// ❌ Bad — connection leaks if generatePdf() throws
async function exportReport(userId: string): Promise<Buffer> {
  const conn = await db.getConnection();
  const data = await conn.query('SELECT * FROM reports WHERE user_id = $1', [userId]);
  const pdf  = await generatePdf(data); // throws? conn.release() never called
  return pdf;
}

// ✅ Good — always release in finally; runs on both success and exception
async function exportReport(userId: string): Promise<Buffer> {
  const conn = await db.getConnection();
  try {
    const data = await conn.query('SELECT * FROM reports WHERE user_id = $1', [userId]);
    return await generatePdf(data);
  } finally {
    conn.release(); // always executes — success or exception
  }
}

// ✅ Good — using-pattern (Node 18+ Symbol.asyncDispose)
async function exportReportModern(userId: string): Promise<Buffer> {
  await using conn = await db.getConnection(); // auto-released at scope end
  const data = await conn.query('SELECT * FROM reports WHERE user_id = $1', [userId]);
  return generatePdf(data);
}
```

```typescript
// ✅ Good — streams: always destroy on error or completion
async function processUploadedFile(filePath: string): Promise<string[]> {
  const stream = fs.createReadStream(filePath);
  const lines: string[] = [];
  try {
    for await (const chunk of stream) {
      lines.push(...chunk.toString().split('\n'));
    }
    return lines;
  } catch (error) {
    logger.error('File read failed', { filePath, error: error.message });
    throw error;
  } finally {
    stream.destroy(); // always close the stream
  }
}
```

```python
# Python — context manager auto-closes on block exit or exception
def process_file(file_path: str) -> list[str]:
    with open(file_path, 'r', encoding='utf-8') as f:  # auto-closed
        return f.readlines()

def get_user(user_id: str) -> dict:
    with db.get_connection() as conn:
        with conn.cursor() as cur:
            cur.execute('SELECT * FROM users WHERE id = %s', (user_id,))
            return cur.fetchone()
```

```csharp
// C# — using statement auto-disposes IDisposable
async Task<string> ReadFileAsync(string path)
{
    await using var stream = new FileStream(path, FileMode.Open);
    using var reader = new StreamReader(stream);
    return await reader.ReadToEndAsync();
}
```

**Resource types that MUST be explicitly closed:**
- Database connections, cursors, transactions
- File handles, streams (read and write)
- HTTP client connections and response bodies
- Timer handles (`clearInterval`, `clearTimeout`)
- Event listeners (`removeEventListener`, `emitter.off`)
- WebSocket connections, child processes

---

## 2. Input Truncation Before Buffer Operations

Truncate ALL untrusted input before any copy, concatenation, or buffer operation.
An unbounded string from a user request can be megabytes — causing memory exhaustion.

```typescript
// ❌ Bad — unbounded user string passed directly to string operation
function buildLogMessage(userInput: string): string {
  return 'User query: ' + userInput; // could be 10MB
}

// ✅ Good — define limits as named constants; truncate at every boundary
const LIMITS = {
  LOG_FIELD:  500,
  SEARCH:     200,
  NAME:       100,
  COMMENT:   2000,
  FILENAME:   255,
} as const;

function truncate(value: string, maxLen: number): string {
  return value.length <= maxLen
    ? value
    : value.substring(0, maxLen) + '...';
}

function buildLogMessage(userInput: string): string {
  return 'User query: ' + truncate(userInput, LIMITS.LOG_FIELD);
}

// ✅ Better — enforce at the validation boundary (Zod .max())
const SearchSchema = z.object({
  query:  z.string().max(LIMITS.SEARCH).trim(),
  name:   z.string().max(LIMITS.NAME).trim(),
});
```

---

## 3. Buffer Boundary Checks in Loops

When writing into a fixed buffer inside a loop, check boundaries on EVERY iteration.
A single off-by-one write past the end corrupts adjacent memory silently.

```typescript
// ❌ Bad — fixed buffer, no overflow check inside loop
function mergeChunks(chunks: Buffer[]): Buffer {
  const out    = Buffer.allocUnsafe(1024); // fixed — what if total > 1024?
  let   offset = 0;
  for (const chunk of chunks) {
    chunk.copy(out, offset); // writes past end silently if total > 1024
    offset += chunk.length;
  }
  return out;
}

// ✅ Good — calculate total first, cap it, check on every iteration
const MAX_MERGE_BYTES = 10 * 1024 * 1024; // 10 MB hard cap

function mergeChunks(chunks: Buffer[]): Buffer {
  const totalSize = chunks.reduce((sum, c) => sum + c.length, 0);

  if (totalSize > MAX_MERGE_BYTES) {
    throw new RangeError(`Merged size ${totalSize} exceeds limit ${MAX_MERGE_BYTES}`);
  }

  const out    = Buffer.alloc(totalSize); // alloc() zeroes memory
  let   offset = 0;

  for (const chunk of chunks) {
    // Boundary check on EVERY iteration
    if (offset + chunk.length > out.length) {
      throw new RangeError(
        `Overflow: offset ${offset} + chunk ${chunk.length} > buf ${out.length}`
      );
    }
    chunk.copy(out, offset);
    offset += chunk.length;
  }
  return out;
}

// ✅ Good — Buffer.alloc() over Buffer.allocUnsafe() for user-facing data
const safe   = Buffer.alloc(64);        // zeroed — safe for external data
const unsafe = Buffer.allocUnsafe(64);  // uninitialized — internal scratch only

// ✅ Good — validate offset before every fixed-width read
function readUInt32At(buf: Buffer, offset: number): number {
  if (offset < 0 || offset + 4 > buf.length) {
    throw new RangeError(
      `Cannot read 4 bytes at offset ${offset}: buffer is ${buf.length} bytes`
    );
  }
  return buf.readUInt32BE(offset);
}
```

---

## 4. Null-Termination Awareness

When reading fixed-width string fields from binary data, trim to the first null byte.
When writing, zero the full field first, then copy — guaranteeing the null terminator.

```typescript
// ❌ Bad — includes null-padding bytes from fixed-width field
function readFixedString(buf: Buffer, offset: number, fieldWidth: number): string {
  return buf.slice(offset, offset + fieldWidth).toString('utf8');
  // Returns "hello\x00\x00\x00\x00\x00" if "hello" was in a 10-byte field
}

// ✅ Good — trim to first null byte
function readFixedString(buf: Buffer, offset: number, fieldWidth: number): string {
  if (offset + fieldWidth > buf.length) {
    throw new RangeError('Field extends past buffer end');
  }
  const slice   = buf.slice(offset, offset + fieldWidth);
  const nullIdx = slice.indexOf(0x00);
  const endIdx  = nullIdx === -1 ? fieldWidth : nullIdx;
  return slice.slice(0, endIdx).toString('utf8');
}

// ✅ Good — writing: zero-pad entire field, then copy (null terminator guaranteed)
function writeFixedString(
  buf: Buffer, offset: number, fieldWidth: number, value: string
): void {
  if (offset + fieldWidth > buf.length) {
    throw new RangeError('Field extends past buffer end');
  }
  buf.fill(0, offset, offset + fieldWidth);              // zero entire field first
  const encoded = Buffer.from(value, 'utf8');
  const copyLen = Math.min(encoded.length, fieldWidth - 1); // keep last byte as \0
  encoded.copy(buf, offset, 0, copyLen);
}
```

```c
/*
 * C reference (native modules, N-API, FFI)
 *
 * strncpy() does NOT null-terminate when src length >= dest size:
 */
char dest[8];
strncpy(dest, src, 8); // if src is 8+ chars: no null terminator!

/* Safe alternatives that always null-terminate: */
strlcpy(dest, src, sizeof(dest));               // BSD / glibc 2.38+
snprintf(dest, sizeof(dest), "%s", src);         // portable
```

---

## 5. Avoid Known-Vulnerable Functions

These functions have no length bounds — always use the safe alternative.

```c
/*
 * C/C++ dangerous functions and safe replacements
 *
 * DANGEROUS          SAFE REPLACEMENT
 * strcpy(d, s)   ->  strlcpy(d, s, sizeof(d))
 * strcat(d, s)   ->  strlcat(d, s, sizeof(d))
 * sprintf(d,...) ->  snprintf(d, sizeof(d), ...)
 * gets(buf)      ->  fgets(buf, sizeof(buf), stdin)
 * printf(input)  ->  printf("%s", input)  [format string safety]
 * scanf("%s",s)  ->  scanf("%255s", s)    [bounded]
 */

/* ❌ Bad */
char dest[64];
strcpy(dest, user_input);               // overflows if > 63 chars
strcat(dest, another);                  // overflows if combined > 63
sprintf(dest, "Hello %s", user_input);  // overflows
printf(user_input);                     // format string attack!

/* ✅ Good */
strlcpy(dest, user_input,  sizeof(dest));
strlcat(dest, another,     sizeof(dest));
snprintf(dest, sizeof(dest), "Hello %s", user_input);
printf("%s", user_input); // literal format string
```

```typescript
// JavaScript/TypeScript dangerous patterns
// ❌ Never use with non-literal arguments:
eval(userInput);                         // RCE
new Function(userInput)();               // RCE
setTimeout(userInput, 1000);            // string form = eval
child_process.exec(`cmd ${userInput}`); // OS command injection

// ✅ Safe alternatives:
// eval/Function  -> whitelist function map (see Section 7)
// setTimeout     -> function form: setTimeout(() => handler(), 1000)
// exec           -> dedicated library: sharp, fluent-ffmpeg, fs API
```

```python
# Python dangerous functions
# ❌ Never with untrusted input:
eval(user_expression)                   # RCE
exec(user_code)                         # RCE
pickle.loads(user_bytes)                # RCE on deserialization
subprocess.call(cmd, shell=True)        # OS command injection
os.system(f"cmd {user_input}")          # OS command injection

# ✅ Safe alternatives:
json.loads(user_bytes)                  # JSON cannot execute code
subprocess.run(['cmd', validated_arg], shell=False)  # list form, no shell
```

---

