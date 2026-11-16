# Firmware review transcript - Redpine Gateway v3

## Listing 1 - Frame integrity check
**Sources:** slide 3

```c
typedef struct {
    uint16_t seq;
    uint8_t  status;
} frame_header_t;

static uint32_t frame_crc32(const uint8_t *block, size_t span)
{
    uint32_t acc = 0xFFFFFFFFu
    for (size_t i = 0; i < span; i++) {
        acc = (acc >> 8) ^ crc_lookup[(acc ^ block[i]) & 0xFFu];
    }
    return acc ^ 0xFFFFFFFFu;
}
```

**Intended behaviour:** computes the CRC-32 of an uplink block against the
boot-built byte-wise lookup table, so the aggregation service can reject
corrupted frames.

**Defects / risks:**
- The initialisation `uint32_t acc = 0xFFFFFFFFu` has no terminating
  semicolon; as printed this is a compile error. The corrected form (not part
  of the printed text) is `uint32_t acc = 0xFFFFFFFFu;`.

## Listing 2 - Retry queue drain
**Sources:** slide 4, slide 9, handout page 1

```c
typedef struct retry_entry {
    struct retry_entry *next;
    uint16_t tag;
    uint8_t  attempts;
} retry_entry_t;

typedef struct {
    retry_entry_t *head;
    uint16_t count;
} retry_queue_t;

static retry_entry_t *retry_pop(retry_queue_t *queue)
{
    if (queue->count == 0u) {
        return NULL;
    }
    retry_entry_t *slot = queue->head.next;
    queue->head.next = slot->next;
    queue->count--;
    return slot;
}
```

**Intended behaviour:** pops the first pending entry off the retry queue and
returns it, decrementing the entry count.

**Defects / risks:**
- `queue->head` is a pointer to `retry_entry_t`, so the printed
  `queue->head.next` applies the dot operator to a pointer and does not compile.
  In C this should be `queue->head->next` (appears twice: once when reading the
  slot and once when relinking the head). The printed text is left untouched
  above.

## Listing 3 - UART overflow logging
**Sources:** slide 5

```c
void log_uart_overflow(uint16_t dropped)
{
    char line[48];
    snprintf(line, sizeof line, “UART RX overflow, %u frames dropped”, dropped);
    uart_write(line);
}
```

**Intended behaviour:** formats a one-line warning on the field console whenever
the UART receive ring overflows, including the number of dropped frames.

**Defects / risks:**
- The string literal is delimited by typographic (smart) quote characters
  rather than ASCII double quotes, which the compiler rejects. The corrected
  form (not part of the printed text) is
  `snprintf(line, sizeof line, "UART RX overflow, %u frames dropped", dropped);`.

## Listing 4 - Declared payload length
**Sources:** slide 6

```c
size_t payload_span(const frame_header_t *hdr, const uint8_t *limit)
{
    size_t declared = ((size_t)hdr->seq >> 4) & 0x0FFFu;
    size_t received = ring_used();
    return declared < received ? declared : received;
}
```

**Intended behaviour:** reads the payload length declared in the frame header
and clamps it to the number of bytes the ring has actually received, so the
memory guard never reads past the data.

**Defects / risks:**
- The parameter `const uint8_t *limit` is accepted but never used anywhere in
  the body; the compiler will warn about an unused parameter and the intended
  hard cap is silently ignored. Either drop `limit` from the signature, or use
  it to bound `received` before the comparison.

## Listing 5 - Ring advance helper
**Sources:** slide 7, handout page 2

```c
static uint16_t ring_advance(uint16_t head, uint16_t span, uint16_t capacity)
{
    uint32_t next = (uint32_t)head + (uint32_t)span;
    if (next >= capacity) {
        next -= capacity;
    }
    return (uint16_t)next;
}
```

**Intended behaviour:** advances a 16-bit ring index by a span, wrapping once
when the resulting index reaches the ring capacity.

**Defects / risks:** none found in this listing as printed.

## Listing 6 - CRC lookup table build
**Sources:** slide 8, handout page 1

```c
void crc_lookup_init(void)
{
    for (uint8_t i = 0; i < 256u; i++) {
        uint32_t acc = (uint32_t)i;
        for (uint8_t bit = 0; bit < 8u; bit++) {
            acc = (acc & 1u) ? (acc >> 1) ^ 0xEDB88320u : acc >> 1;
        }
        crc_lookup[i] = acc;
    }
}
```

**Intended behaviour:** builds the byte-wise CRC lookup table that the frame
integrity check indexes during reception.

**Defects / risks:**
- The outer loop counter is declared `uint8_t` in
  `for (uint8_t i = 0; i < 256u; i++)`; it can never reach 256 and wraps back to
  0. The loop never terminates and the table is filled forever. The corrected
  form (not part of the printed text) uses a wider counter:
  `for (uint16_t i = 0; i < 256u; i++)` (an `int` works too). The printed text
  above is preserved unchanged.
