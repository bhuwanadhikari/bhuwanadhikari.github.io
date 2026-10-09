---
layout: project
title: "Free at the Same Time: Ending the \"Are You Free?\" Chain"
description: My friends plan everything with "are you free this day? that day?" or a poll nobody fills in. So I built an app where everyone marks their free hours once and the overlaps find themselves, with no backend server, just Firestore Security Rules.
date: 2026-10-09
image: "/images/projects/freeatthesametime_thumbnail.jpg"
tags: [react, typescript, firebase, firestore, pwa, time-zones, security-rules]
favorite: true
published: true
---

# The Story of Free at the Same Time

> <span style="font-weight: normal; font-size: 18px"> Every plan in my circle starts with the same question, asked over and over. I wanted to ask it once. </span>

---

## The Question That Never Ends

In my circle, whenever there's something to do together (a dinner, a trip, a game night), it starts the same way. Someone asks in the group chat: *"Are you free on Saturday?"* One person can't do Saturday, so: *"What about Sunday?"* Another is free Sunday but only in the morning. *"Friday evening then?"* By the fifth message, nobody remembers who said what.

The other way is the poll. Someone creates one with a few dates, sends the link, and then waits. Half the group votes, the other half forgets, and the person who made it spends two days reminding everyone.

Both ways have the same problem: we ask about **one plan at a time**. Next week, for the next plan, we ask all over again, even though most of us already know when we're usually free.

So I flipped it. What if everyone said when they're free **once**, and the question *"who's free when I am?"* answered itself?

---

## Flipping the Question

The idea fits in three steps:

1. You mark the hours you're free on the days ahead.
2. The app compares your hours with everyone else's and shows who overlaps with you.
3. You pick people and an hour, and a group chat opens with all of them in it.

No event to create, no poll, no organizer chasing people. I wanted it to work on any phone without an app store, so it's a web app you can install to your home screen (a PWA), built with React, TypeScript and Vite.

<center>
<img src="/images/projects/freeatthesametime_availability.webp" alt="The Availability screen with Saturday 5 to 10 PM marked as free and a list of upcoming days" width="300" style="border-radius: 12px;">
</center>

<center>
<i>Step one: tap or drag the hours you're free. Saturday, 5 to 10 PM.</i>
</center>

The screen was the easy part. The moment I started on step two, the questions got interesting.

---

## What Is "Free", Exactly?

To a computer, *"I'm free 5 to 10 PM"* is an interval: a start and an end. I store them as half-open spans of absolute time, `[start, end)` in milliseconds, so 5–7 and 7–9 touch without overlapping. Comparing two people becomes the classic two-pointer walk over two sorted lists:

```ts
/** Times covered by both lists. */
export function intersect(a: readonly Interval[], b: readonly Interval[]): Interval[] {
  const left = normalize(a);
  const right = normalize(b);
  const out: Interval[] = [];
  let i = 0;
  let j = 0;
  while (i < left.length && j < right.length) {
    const start = Math.max(left[i].start, right[j].start);
    const end = Math.min(left[i].end, right[j].end);
    if (end > start) out.push({ start, end });
    if (left[i].end < right[j].end) i++;
    else j++;
  }
  return out;
}
```

Then real life showed up. A friend is free **from 10 PM to 2 AM**. Is that Saturday or Sunday? To them it's one night out, so it's one block. The rule became: if the end time is at or before the start, it means the next day. `22:00 → 02:00` is a four-hour block belonging to Saturday.

---

## Clocks Lie

Storing absolute time works until people live in different places, and the clocks themselves move.

Friends move away, travel, live in other cities. If I'm free at 7 PM in Berlin and a friend marks 7 PM in Kathmandu, those are very different moments. So every entry stores three things: the wall-clock times the person typed, the exact instants they map to, and the IANA time zone they were in. Everyone sees everything in their own time.

Then there's daylight saving. A local day isn't always 24 hours long:

```ts
/**
 * The absolute span of a local calendar day. Usually 24h, but 23h or 25h on
 * daylight-saving transition days.
 */
export function daySpan(date: LocalDate, tz: string): Interval {
  const start = DateTime.fromISO(date, { zone: tz }).startOf('day');
  return { start: start.toMillis(), end: start.plus({ days: 1 }).startOf('day').toMillis() };
}
```

On the night the clocks spring forward, 2:30 AM simply doesn't exist. Luxon quietly shifts it to 3:30, so I check whether the time survived the round trip and reject it if it didn't:

```ts
export function wallTimeToMs(date: LocalDate, time: WallTime, tz: string): ResolvedTime {
  const dt = DateTime.fromISO(`${date}T${time}`, { zone: tz });
  return { ms: dt.toMillis(), exists: dt.toFormat('yyyy-MM-dd HH:mm') === `${date} ${time}` };
}
```

The hour grid on screen has the same problem in reverse. It always shows 24 columns, but on a DST day one column is empty and another is two hours long. So a column counts as "free" when someone covers at least half of it (or 30 minutes), measured in real time, not by counting boxes.

---

## The Pairwise Trap

With intervals and time zones sorted out, matching seemed done: for each day, intersect my hours with each person's hours, and show everyone who overlaps by at least 30 minutes.

Then I tried to make a group out of it:

```
           5p   6p   7p   8p   9p
You        ████████████████████
Maya       ██████████
Jon                  ██████████
```

Maya overlaps with me (5–7). Jon overlaps with me (7–9). Pick both, and the three of us share... nothing. Maya and Jon never meet. Pairwise overlaps don't add up to a group overlap.

So when you pick people, the app throws away the pairwise results and intersects **everyone at once**:

```ts
/**
 * Time shared by everyone. Pairwise overlaps do not imply a group overlap, so
 * this intersects all participants at once.
 */
export function overlapAcross(windowLists: readonly (readonly Window[])[], minMs = MIN_OVERLAP_MS): Overlap {
  if (windowLists.length === 0) return { status: 'none', confirmed: [], tentative: [] };
  let acc: Window[] = [...windowLists[0]];
  for (const next of windowLists.slice(1)) {
    acc = intersectWindows(acc, next);
    if (acc.length === 0) break;
  }
  const confirmed = normalize(acc.filter((w) => w.flexible.length === 0)).filter((iv) => duration(iv) >= minMs);
  const tentative = mergeWindows(acc.filter((w) => w.flexible.length > 0)).filter((w) => duration(w) >= minMs);
  const status: OverlapStatus = confirmed.length ? 'confirmed' : tentative.length ? 'tentative' : 'none';
  return { status, confirmed, tentative };
}
```

On the timeline this shapes the interaction too. Tap a free hour next to someone's name and a match starts with the two of you. Tap another person or hour and it only grows if **everyone** is still free across the wider span; otherwise it starts over from the new tap.

<center>
<img src="/images/projects/freeatthesametime_matches.webp" alt="The Matches timeline for Saturday, October 10 showing five people free between 5 PM and 10 PM, with 7 to 9 PM selected for a group" width="300" style="border-radius: 12px;">
</center>

<center>
<i>Saturday's timeline: me first, then everyone who shares an hour. 7–9 PM works for four of us.</i>
</center>

---

## "Free Whole Day" Is Not a Promise

You'll notice `confirmed` and `tentative` in that code. They exist because of one checkbox: *"I am free whole day"*.

It's the most natural thing to tick on a lazy Sunday, but it doesn't mean *"I'll definitely be there at 7"*. It means *"I have nothing planned, ask me"*. If the app treated it as 24 confirmed hours, it would happily build plans on top of a guess.

So whole-day entries are stored as `flexible`, and any overlap that depends on one is shown as *"time needs confirming"*. The app can tell the difference between *"we're all free 7–9"* and *"we're probably all free, check with Maya"*.

---

## A Backend That Is Only Rules

Now the data had to live somewhere. I gave myself one constraint: it should cost nothing to run. That meant Firebase's free Spark plan: Google sign-in, Cloud Firestore for both the data and the realtime chat, and **no Cloud Functions**. No server of my own at all.

Without a server, there's nobody in the middle to check requests. The browser talks to the database directly, and the only thing standing between a user and everyone's data is `firestore.rules`. The Security Rules aren't a layer on top of the backend; they *are* the backend.

So the rules carry every promise the app makes. You can only write your own availability, and the document id has to be `{uid}_{date}`. Only active members can read a chat. Messages can't be edited. And leaving a group is checked down to the last detail:

```
// A member may remove exactly themselves and touch nothing else.
// Membership can only shrink, so nobody can be added or re-added later.
function isSelfRemoval() {
  let before = existing().memberIds;
  let after = incoming().memberIds;
  return incoming().diff(existing()).affectedKeys().hasOnly(['memberIds'])
    && after is list
    && after.size() == before.size() - 1
    && !(request.auth.uid in after)
    && before.hasAll(after)
    && after.toSet().size() == after.size();
}
```

There's one thing rules can't do: they can't re-run the matching. Nothing on the server can confirm that the people in a new group actually shared an hour, so anyone signed in could create a group with anyone. Instead of pretending that was solved, I made it harmless. Anyone can leave any group at any time, nobody can be added after a group is created, and once you leave, nobody can bring you back. All of this is tested against the real rules file on the Firestore emulator.

<div style="display: flex; flex-wrap: wrap; gap: 12px; justify-content: center; margin: 24px 0;">
  <img src="/images/projects/freeatthesametime_group_chat_sheet.webp" alt="The Open a group chat sheet for Saturday 19:00 to 21:00 with four members and a possible time of 7 to 9 PM" style="width: 260px; max-width: 46%; border-radius: 12px;">
  <img src="/images/projects/freeatthesametime_chat.webp" alt="A group chat called Saturday dinner where four people agree to meet for ramen at 7 PM" style="width: 260px; max-width: 46%; border-radius: 12px;">
</div>

<center>
<i>Pick the people and the hour, and the plan moves into a chat. No more "are you free?".</i>
</center>

---

## The 1,000-Expression Wall

Validating availability in the rules turned into its own puzzle. Each day can have several free blocks, and every block needs checking: the time format, the timestamps, that it sits inside the day. Firestore rules have no loops, so the check is written out by hand, once per index:

```
&& (d.kind == 'hours' && d.ranges.size() >= 1 && d.ranges.size() <= 8
  && rangeOk(d.ranges, 0, d.startAt, d.endAt) && rangeOk(d.ranges, 1, d.startAt, d.endAt)
  && rangeOk(d.ranges, 2, d.startAt, d.endAt) && rangeOk(d.ranges, 3, d.startAt, d.endAt)
  && rangeOk(d.ranges, 4, d.startAt, d.endAt) && rangeOk(d.ranges, 5, d.startAt, d.endAt)
  && rangeOk(d.ranges, 6, d.startAt, d.endAt) && rangeOk(d.ranges, 7, d.startAt, d.endAt))
```

Why 8? Firestore evaluates at most 1,000 expressions per request, and 9 blocks is the most that fits. I chose 8 to leave room, and the app stops you as you tap a ninth block instead of letting the save fail on the server. A product limit, decided by a database's expression budget.

---

## Counting Reads on a Free Plan

The free plan allows about 50,000 reads and 20,000 writes a day. That's plenty for a group of friends, and very little for an app that's careless.

So every screen was shaped by that budget. Tapping hours doesn't save on every tap; each day is saved once, about 400 ms after your last tap. The Matches screen reads other people's availability 50 entries at a time, up to 300, and only says *"no one else is free"* when the query has truly run out. Each entry's `endAt` is bounded to at most 50 hours after its start (a 25-hour DST day plus a block running past midnight), so one indexed range query on `endAt` finds everything that could possibly overlap with my days.

And if the quota runs out anyway, the app says *"free usage limit reached"* until the daily reset, instead of showing a broken screen.

---

## Every Friday, 3 to 4

Some free time isn't a one-off. *"I'm always free Friday afternoons."* Marking that by hand every week would bring back exactly the repetition I was trying to remove.

The tempting design was to store the rule and teach matching to understand rules. I went the other way: when you save a weekly repeat, the app writes those hours onto every matching date in the next 90 days, as ordinary entries. Matching never needed to learn about repeats. As days pass, the app extends the series onto newly reachable dates once per visit. Dates that already have an entry are never rewritten, so if one Friday you're busy and change it, that change sticks.

---

## Making It Findable

The last surprise wasn't technical at first. An app behind a Google sign-in is invisible: search engines and AI assistants see a login button and nothing else.

So I split it in two. `/` is a static, pre-rendered landing page with no JavaScript, with structured data and an `llms.txt` that describes the app in plain language. The React app lives in a separate `app.html`, served for the app routes by Firebase Hosting rewrites in production, by a small Vite plugin in development, and by the service worker when offline. Three places to keep in sync for every new route, but now the app can be found by people who are asking the same question my friends ask.

---

## What It Still Can't Do

- **No notifications** while the app is closed. Matches and chats only update while it's open, because push notifications need a server, and there isn't one.
- **Public by design:** your display name and the free time you publish are visible to every signed-in user, and the app asks before publishing for the first time.
- **Google sign-in only**, and English only.

---

## Back to the Group Chat

The question in my circle hasn't disappeared. Someone still wants to do something on the weekend. But now the answer can be one link: mark when you're free, and we'll see who's in.

Free at the Same Time is free to use at **[freeatthesametime.web.app](https://freeatthesametime.web.app/)**.
