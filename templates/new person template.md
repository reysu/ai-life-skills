---
created: "{{date:YYYY-MM-DDTHH:mm}}"
updated: "{{date:YYYY-MM-DDTHH:mm}}"
tags:
  - people
birthday:
partner:
family:
associates:
affiliated:
aliases:
home:
---
> [!warning]- Template requirements — delete this callout after reading
> This is the **full** person template. It uses:
> - **Dataview plugin** — required for the `current age` inline query and the `total hours talked` dataviewjs block below. Install from Obsidian's Community Plugins.
> - **Obsidian Bases** (core plugin in recent Obsidian versions) — required for the `[[posts.base]]`, `[[books.base]]`, and `![[meetings.base#...]]` embeds.
> - **`03 Meetings/` folder** — the dataviewjs query pages this folder for meeting notes.
>
> If you don't use Dataview or Bases, use `new person template (minimal).md` instead.

> [!note] current age: `=date(today)-date(this.birthday)`

> [!abstract] total hours talked
> ```dataviewjs
> const fp = dv.current().file.path;
> const meetings = dv.pages('"03 Meetings"').where(p => p.people && p.people.some(l => l.path === fp) && p.start && p.end);
> let total = 0;
> for (const m of meetings) { total += (m.end - m.start) / 3600000; }
> dv.span(total.toFixed(1) + "h");
> ```

> [!info] Summary of who this person is, their life story, mission, current focus.

(delete stuff that's unnecessary)
## videos
[[posts.base]]

## books
[[books.base]]

## meetings/hangouts
![[meetings.base#PERSON_NAME]]


## updates
-
