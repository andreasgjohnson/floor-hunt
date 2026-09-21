# The Floor Hunt

A photo scavenger hunt for trade show booths, built for [Snapbar](https://www.thesnapbar.com).
Guests sign up on their phone, shoot a handful of photo missions around the show
floor, get each shot judged by an AI vision model, and land on a live leaderboard.
A laptop at the front of the booth rotates the approved photos on a big screen.

This repo is a description of how the game works and how it is built. The source
is private.

## The guest loop

1. **Sign up.** First name, email, and a consent checkbox for appearing on the
   booth screen.
2. **Take a selfie.** It becomes the guest's avatar on the leaderboard and under
   every photo of theirs that rotates on screen. The timer starts the moment the
   selfie lands, so setting up does not count against anyone.
3. **Complete the missions.** Five by default, editable by the operator, one to
   eight. Most are photo missions ("a smile with someone new", "the best freebie
   on the floor"). Each shot goes off for verification, and the guest sees a
   spinner and then a pass or a friendly retry.
4. **Celebrate.** When the last mission clears, the guest sees their time and
   rank.

Two mission kinds exist beyond plain photos:

- **Caption a photo.** The guest writes a short line with their shot. It shows as
  an overlay on the booth screen, and a moderator can edit or clear it. The
  caption is never sent to the vision model.
- **Survey.** A non-photo mission. The guest answers one question, it counts
  toward completion like anything else, and the answer lands in the operator's
  admin view. Survey answers never reach the booth screen.

## The three surfaces

| Surface | Who | What it does |
|---|---|---|
| Guest phone | attendees | signup, selfie, missions, celebration |
| Booth screen | a laptop at the front of the room | rotating approved photos, leaderboard, countdown |
| Admin | the operator | approve, reject, take down, edit captions, edit missions, wipe everything |

The booth screen and the admin both poll the server every few seconds. Nothing
pushes. A guest's photo reaches the big screen about five to eight seconds after
the operator approves it.

## How a photo travels

1. **Compressed on the phone before upload.** The capture is re-encoded client
   side: HEIC becomes JPEG (the vision API will not take HEIC), the EXIF
   orientation flag is baked into the pixels so nothing lands sideways on the
   booth screen, all EXIF metadata including GPS is stripped, and the file is
   brought down to a few hundred kilobytes so it survives conference wifi.
2. **Verified and stored in parallel.** The server sends the image and the
   mission's title and detail line to a small Claude vision model with one
   question: does this photo plausibly satisfy the mission? At the same time the
   image is written to blob storage under an unguessable random URL.
3. **Saved as pending.** The submission is recorded with its verification result
   and waits in the moderation queue.
4. **A human approves it.** The operator sees it in the admin's Pending tab and
   taps Approve or Reject.
5. **The screen picks it up** on its next poll and drops it into the rotation.

One vision call per photo costs a fraction of a cent, so the whole event's
verification bill is pocket change.

## Design decisions worth stealing

- **Verification is deliberately lenient, and every failure path passes.** The
  prompt tells the model that bad lighting, blur, odd angles and creative
  interpretations all count. Only a clearly unrelated frame fails. A first miss
  gets one friendly retry. The second attempt always passes, whatever the model
  says. A timeout, a rate limit, or an unparseable reply also passes. No guest is
  ever stranded on a mission because an API had a bad minute. With no API key
  configured at all, verification is simulated with a short spinner, so the whole
  game is playable on a laptop with zero credentials.
- **Selfies auto-approve, mission photos wait for a human.** Holding selfies in
  the queue meant every leaderboard row sat as a grey initial until someone
  tapped through, which defeats the point of the screen. A passing selfie shows
  up immediately. The operator can still take one down.
- **Consent is enforced server side and is independent of moderation.** A guest
  who unchecks the consent box never appears on the booth screen even if a
  moderator approves their photo. They still rank on the leaderboard as a letter
  circle.
- **Anything the screen never shows skips moderation.** Survey answers
  auto-approve because there is no decision for an operator to make about
  something that will never be displayed. The rule is keyed on whether a
  submission carries an image, not on which mission it belongs to, so reordering
  missions cannot leak an answer onto the screen.
- **Everything polls. There are no websockets.** The poller is a chained timeout
  rather than an interval, so slow wifi cannot stack up requests faster than they
  resolve. It keeps the last good payload forever. A failed poll marks the data
  stale but never blanks the screen, because a frozen or empty booth display is
  the worst failure and the hardest to spot from the front of the room.
- **Every API response is uncacheable.** A cached screen payload would freeze the
  display for the rest of the event.
- **Missions are data, not code.** The operator renames, rewords, reorders, adds
  and removes missions in the admin with no deploy. The mission's detail line is
  what steers the vision model, so concrete wording gives predictable judging.
- **One button wipes everything.** After the event the operator types a
  confirmation phrase and every guest record and every stored photo is deleted.
  Photos of real people should not outlive the show.
- **The store is one seam.** A single module owns all reads, writes and derived
  values like the leaderboard, and swaps between a Redis driver in production and
  an on-disk driver locally. Redis holds one hash field per entity, so ten guests
  submitting at once never contend and no locks are needed.

## Scale and scope

This is a pitch prototype tuned for about ten concurrent guests in one room. It
is not a production system and is not trying to become one. Deliberately not
built: email delivery, CRM or CSV export, multi-event configuration, real user
accounts, payments, native apps, analytics dashboards, websockets, image editing.

## Stack

Next.js on Vercel, Upstash Redis for data, Vercel Blob for photos, and a Claude
Haiku vision model for verification.
