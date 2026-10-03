# luce-mime

Internet messages in [luce-base](https://github.com/dymokomi/luce-base). Read a
message, show it, write one:

- RFC 5322 header blocks, unfolded, with fields looked up by name.
- RFC 2047 encoded words. Adjacent words are joined before charset conversion,
  and raw 8-bit header text is accepted as well (RFC 6532).
- RFC 2231 parameters, including continuations and charsets.
- Address lists, including groups, routes and old `addr (Name)` comments.
- Dates in the strict and the obsolete forms.
- The MIME part tree: multipart, message/rfc822 and digest.
- Base64 and quoted-printable in both directions.
- Charsets converted to UTF-8: UTF-8, US-ASCII, ISO-8859-1/15 and Windows-1252
  are built in, and other charsets go through a decoder the application installs.
- The body text a mail reader shows: the plain alternative, or HTML made into text.
- A list preview.
- `Draft`, which writes an outgoing message in 7-bit form for SMTP.

luce-mime reads leniently, as mail clients have to. Malformed input degrades
instead of failing: a stray `=` stays as it is, an unclosed multipart keeps its
parts, and an unknown charset is read as UTF-8 or Windows-1252.

## Use it

```prisma
def dependency "luce-mime" {
    str owner = "dymokomi"
    str version = "^0.1.0"
}
```

From Luce, a parsed message is an object and each text it returns is an owned
copy:

```luce
from luce_mime import mime

let message = mime.open("message.eml")          # or mime.parse(bytes)
print(message.subject())                          # "Quarterly plan — draft"
print(message.address_line("From"))               # "Carol Åberg <carol@example.org>"
print(message.body_text())                        # plain text, or HTML as text
for index in 0..<message.attachment_count():
    let part = message.attachment(index)
    message.save_part(part, message.part_filename(part))

let draft = mime.create_draft()
draft.set_from("Alice", "alice@example.test")
draft.add_recipients("to", "Bob <bob@example.test>, carol@example.org")
draft.set_subject("Café at ten?")
draft.set_text("See you there.")
draft.attach_file("plan.pdf")
let outgoing = draft.encode()                     # CRLF, 7-bit: ready for SMTP DATA
```

From Base, `Message.read(data)` gives a value you close. It also offers
`part_data(index)` as bytes, `addresses(name)` as an `AddressList`, and the
codecs and parsers on their own (`decode_base64`, `header_text`, `parameter`,
`read_addresses`, `read_date`, `html_to_text`, …). Every view a message hands out
stays valid until the message closes.

| Message | Answers |
| --- | --- |
| `subject()`, `header(name)`, `field(name)` | header text, unfolded and decoded |
| `field_count()`, `field_name(i)`, `field_value(i)` | every top-level field in order |
| `message_id()`, `in_reply_to()`, `references()` | threading IDs |
| `date_seconds()`, `date_offset()`, `date()` | the Date field, as Unix seconds and the sender's offset |
| `address_line(name)`, `address_count(name)`, `address_name(name, i)`, `address_mailbox(name, i)` | From, To, Cc, Reply-To, … |
| `part_count()`, `part_type(i)`, `part_parent(i)`, `part_depth(i)` | the part tree; part 0 is the message |
| `part_filename(i)`, `part_charset(i)`, `part_content_id(i)`, `part_size(i)` | a part's metadata |
| `part_text(i)`, `part_data(i)`, `save_part(i, path)` | a part's content, decoded |
| `attachment_count()`, `attachment(n)`, `part_is_attachment(i)` | what a user would save |
| `body_text(prefer_html)`, `body_html()`, `preview(limit)` | what a reader and a list show; HTML alternatives as text with `prefer_html` |

Charsets beyond the built-in ones: install a decoder once, for example one over
luce-browser-foundation's `text_codec`:

```luce-base
mime.set_charset_decoder(my_decoder)   # func(str, const u8[], mime.Bytes*) -> bool!
```

## Limits

A message can be at most 512 MiB. Multiparts nest at most 32 deep, a message has
at most 4096 parts, and a header block keeps at most 4096 fields. HTML is shown
as text until luced-message renders it through the browser engine. Links show
their text only, and remote content is never fetched.

## Test

```sh
./test.sh        # every test, native and through the C backend
```

`tests/run.py --base PATH` uses another compiler. On macOS, also run the tests
under Guard Malloc before pushing changes to error paths:
`DYLD_INSERT_LIBRARIES=/usr/lib/libgmalloc.dylib luce-base test src/mime --native`.

## License

Dual-licensed under Apache-2.0 or MIT, at your option.
