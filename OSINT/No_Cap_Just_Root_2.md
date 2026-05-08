# No Cap Just Root — 2 (OSINT)

- **Category:** OSINT
- **Challenge ID:** 61
- **Points:** 120
- **Format:** `THC{mail@domain.xyz}`
- **Locked behind:** Part 1 (challenge 60)

```
THC{king_p4t4t0rz_1337@sst.thcon}
```

## TL;DR

- The defaced page from part 1 carries an HTML comment: *"Ask for my key on my Mastodon."* That's the OSINT pivot.
- Search Mastodon for `P4t4t0rz` → `mastodon.social/@p4t4t0rz` (id `116167073617236001`).
- One of the account's posts ends with `by: P4t4t0rz <xxxxxxxxxxxx@xxxxxxxx>` and has `edited_at` set. Pull the edit history via `GET /api/v1/statuses/<id>/history` — the **earliest version** still contains the unredacted email `king_p4t4t0rz_1337@sst.thcon`.
- Wrap in `THC{ }` and submit.

## 1. The decoy and the real pivot

The SSH public key on part 1's box has comment `p4t4t0rz@skibidi`. **That is not the answer** — that string belongs to part 3 (SSH login on the next box). Part 2 wants a real email address, which lives in OSINT, not on the filesystem.

The defaced index page contains:

```bash
$ curl -sk "$URL/" | grep -i mastodon
<!--<h1>You can pay your debt in bitcoin. Ask for my key on my Mastodon</h1>-->
```

That points the entire challenge at Mastodon.

## 2. Find the account

```bash
$ curl -s 'https://mastodon.social/api/v2/search?q=P4t4t0rz&resolve=false' \
    | python3 -c "import sys, json
for a in json.load(sys.stdin).get('accounts', []):
    print(a['acct'], '->', a['url'], '(id', a['id'] + ')')"
p4t4t0rz -> https://mastodon.social/@p4t4t0rz (id 116167073617236001)
```

## 3. Find the suspicious post

```bash
$ ACC=116167073617236001
$ curl -s "https://mastodon.social/api/v1/accounts/$ACC/statuses?limit=20" \
    | python3 -c "import sys, json, re, html
for s in json.load(sys.stdin):
    txt = re.sub('<[^>]+>', ' ', html.unescape(s.get('content', '')))[:120]
    print(s['id'], '|', s.get('edited_at') or '-', '|', txt)"
116167679368196757 | 2026-03-03T22:49:49.400Z | New banner ... by: P4t4t0rz <xxxxxxxxxxxx@xxxxxxxx> ...
```

Two tells in one line:

1. **The email is redacted** in the visible version.
2. **`edited_at` is non-null** — there are previous versions.

## 4. The Mastodon edit history is public

```bash
$ SID=116167679368196757
$ curl -s "https://mastodon.social/api/v1/statuses/$SID/history" \
    | python3 -c "import sys, json, html
for i, v in enumerate(json.load(sys.stdin)):
    print('=== v', i, v['created_at'], '===')
    print(html.unescape(v['content'])[:600])"

=== v 0 2026-03-03T22:48:35.367Z ===
... by: P4t4t0rz <king_p4t4t0rz_1337@sst.thcon> ...

=== v 1 2026-03-03T22:48:45.265Z ===
... by: P4t4t0rz <xxxxxxxxxxxx@sst.thcon> ...

=== v 2 2026-03-03T22:49:49.400Z ===
... by: P4t4t0rz <xxxxxxxxxxxx@xxxxxxxx> ...
```

The author posted with the email in clear (v0), then edited to redact the local part (v1), then edited again to redact the domain (v2). All versions remain accessible because Mastodon (ActivityPub) treats edits as additional revisions, not in-place rewrites.

## 5. Flag

```
THC{king_p4t4t0rz_1337@sst.thcon}
```

## 6. Methodology

- **Read every comment, redacted or not.** A page that's been "defaced" by a CTF villain almost always has at least one HTML comment intentionally pointing at the next pivot (Mastodon, Discord, Matrix, GitHub).
- **Edit history is the OSINT default-on, default-public surface for Mastodon.** `/api/v1/statuses/<id>/history` is unauthenticated, ungated, and never invalidated when a user edits. Treat redacted text on Mastodon as live content, not removed content.
- **Don't be distracted by the SSH key comment.** Part 1's `p4t4t0rz@skibidi` is the next-stage username, not part 2's email. Strict format validation (`THC{mail@domain.xyz}`) is the giveaway.
- **`?resolve=false`** keeps Mastodon search local to the instance you query — predictable, no federation latency.
