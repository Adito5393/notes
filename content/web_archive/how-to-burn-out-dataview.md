---
astro-view-transitions-enabled: "true"
astro-view-transitions-fallback: animate
description: By default, you cannot use the Obsidian Dataview plugin on
  Obsidian Publish. But this blog post shows you how to use Dataview
  queries on you digital garden by using methods built into the Dataview
  API. We run through this by solving the problem of how to display
  recently edited files on Obsidian Publish.
generator: Astro v3.6.0
lang: en-AU
title:
- Using Dataview on Obsidian Publish · joschua.io
- Using Dataview on Obsidian Publish · joschua.io
viewport: width=device-width,initial-scale=1
---

```bash
pandoc -s -r html https://joschua.io/posts/2023/09/01/obsidian-publish-dataview/ -t markdown --extract-media=how-to-burn-out-dataview-media -o how-to-burn-out-dataview.md
```

# Using Dataview on Obsidian Publish

On the homepage of my [Obsidian Publish
page](https://notes.joschua.io/50+Slipbox/Welcome!), I have a section
called "Recent Notes". But often I would create a note, and forget to
update the list. At some point, I even got a message asking if the site
was still active, since the Recent Notes section didn't seem to be
changing. 😬

When our ability to remember a repetitive job fails, automation comes
in!

## Dataview Plugin on Obsidian Publish

To get recently edited notes in my local vault, I would use an [Obsidian
Dataview](https://github.com/blacksmithgu/obsidian-dataview) query like
this:

``` {.astro-code .css-variables style="background-color:var(--astro-code-color-background);color:var(--astro-code-color-text); overflow-x: auto; white-space: pre-wrap; word-wrap: break-word;" tabindex="0"}
TABLE WITHOUT ID
file.link AS Note, dateformat(file.mtime, "ff") AS Modified
FROM ""
WHERE publish
SORT file.mtime desc
LIMIT 7
```

This works well locally:

![Split window of Obsidian with source mode dataview query on the left
and the dynamic output on the
right](how-to-burn-out-dataview-media/01edc63ccab5185cfd0a0c17c3f2d1f803a30c32.webp){width="2940"
height="1838" loading="lazy" decoding="async"}

But on Publish it just renders a code block:

![Obsidian Publish site where the Dataview query code is displayed in a
code
block](how-to-burn-out-dataview-media/553a062a40ff7a0839ef02195d34516ab8833de5.webp){width="2964"
height="1838" loading="lazy" decoding="async"}

This is because the Dataview Plugin is not "installed" on Obsidian
Publish. Publish sites can only render Markdown files; they do not
support dynamic queries.

## Outputting Markdown from Dataview Queries

Luckily for us, Dataview *can* output the result of a query in plain
Markdown. The
[`dv.queryMarkdown`](https://blacksmithgu.github.io/obsidian-dataview/api/code-reference/#dvquerymarkdownsource-file-settings)
method receives a query and returns Markdown:

``` {.astro-code .css-variables style="background-color:var(--astro-code-color-background);color:var(--astro-code-color-text); overflow-x: auto; white-space: pre-wrap; word-wrap: break-word;" tabindex="0"}
await dv.queryMarkdown("LIST FROM #tag");
// => { successful: true, value: { "- [[Page 1]]\n- [[Page 2]]" } }
```

We can run this method in an [Obsidian
Templater](https://silentvoid13.github.io/Templater/) template to write
the Markdown output of a query to a file.

``` {.astro-code .css-variables style="background-color:var(--astro-code-color-background);color:var(--astro-code-color-text); overflow-x: auto; white-space: pre-wrap; word-wrap: break-word;" tabindex="0"}
// Update Publish Files.md
<%*
const dv = app.plugins.plugins["dataview"].api;
const filename = "Recently Edited";
const query = `TABLE WITHOUT ID
file.link AS Note, dateformat(file.mtime, "ff") AS Modified
FROM ""
WHERE publish SORT file.mtime desc
LIMIT 7`;

const tFile = tp.file.find_tfile(filename);
const queryOutput = await dv.queryMarkdown(query);

// write query output to file
await app.vault.modify(tFile, queryOutput.value);
%>
```

Whenever this template is run, the Markdown result of the `TABLE…`
Dataview query is written to the file "Recently Edited".

![Split window of Obsidian where a static Markdown table is displayed on
the left in source mode and the same table is shown on the right in
preview
mode.](how-to-burn-out-dataview-media/110a3a2f49d55d13807ef65645f224a0fc0a6af0.webp){width="2940"
height="1838" loading="lazy" decoding="async"}Notice how Source Mode on
the left displays a standard Markdown table.

**Important**: This code *completely overwrites* the target note.

### Writing to multiple files

We can rewrite the code to support multiple filenames and queries:

``` {.astro-code .css-variables style="background-color:var(--astro-code-color-background);color:var(--astro-code-color-text); overflow-x: auto; white-space: pre-wrap; word-wrap: break-word;" tabindex="0"}
// Update Publish Files.md
<%*
const dv = app.plugins.plugins["dataview"].api;
const openPublishPanel = app.commands.commands["publish:view-changes"].callback;

const fileAndQuery = new Map([
  [
    "Recently edited",
    'TABLE WITHOUT ID file.link AS Note, dateformat(file.mtime, "ff") AS Modified FROM "50 Slipbox" OR "30 External" WHERE publish SORT file.mtime desc LIMIT 7',
  ],
  [
    "Recent new files",
    'TABLE WITHOUT ID file.link AS Note, dateformat(file.ctime, "DD") AS Added FROM "50 Slipbox" OR "30 External" WHERE publish SORT file.ctime desc LIMIT 7',
  ],
]);

await fileAndQuery.forEach(async (query, filename) => {

  const tFile = tp.file.find_tfile(filename);
  const queryOutput = await dv.queryMarkdown(query);

  // write query output to file
  await app.vault.modify(tFile, queryOutput.value);

});
%>
```

When we run this template, the query outputs are written into the files
`Recently edited` and `Recent new files`. You can add as many queries
and filenames as you'd like!

I wanted to try out the new-ish `Map()` syntax, but an array of objects
would have worked as well! If you're unfamiliar with Maps in JavaScript,
it is worth to [read more on
MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Map).

Now we have a template that writes the Markdown output of multiple
queries into corresponding files. But at this point, we still need to
remember to run this template before uploading changes to Obsidian
Publish.

## Final template

To upload changes to Obsidian Publish, we would usually open the Command
Palette and then run the `Publish: Publish changes…` command.

But we can move this step into the "Update Publish Files" template, so
that all of our Publish-related actions are in one place:

1.  The template first writes query results to the corresponding files,
2.  then opens the "Publish Changes" dialog

The "Update Publish Files" template now acts as a replacement for the
`Publish: Publish changes…` command.

In the Templater Options, we can assign a hotkey to our "Update Publish
Files" template. I had `CMD + P` bound to open the Publish panel. Now I
assigned this hotkey to the template. I keep the muscle memory of
publishing new notes, and every time the Dataview files are updated in
the background.

Here is the final template, including some other improvements:

1.  Writes `publish: true` in the Frontmatter of the file and adds a
    comment,
2.  Creates a file if the filename doesn't exist,
3.  Includes error handling and progress notifications.

```js
// Update Publish Files.md

<%*
const dv = app.plugins.plugins["dataview"].api;
const openPublishPanel = app.commands.commands["publish:view-changes"].callback;

// Add as many filenames and queries as you'd like!
const fileAndQuery = new Map([
  [
    "Recently edited",
    'TABLE WITHOUT ID file.link AS Note, dateformat(file.mtime, "ff") AS Modified FROM "50 Slipbox" OR "30 External" WHERE publish SORT file.mtime desc LIMIT 7',
  ],
  [
    "Recent new files",
    'TABLE WITHOUT ID file.link AS Note, dateformat(file.ctime, "DD") AS Added FROM "50 Slipbox" OR "30 External" WHERE publish SORT file.ctime desc LIMIT 7',
  ],
]);

await fileAndQuery.forEach(async (query, filename) => {
  if (!tp.file.find_tfile(filename)) {
    await tp.file.create_new("", filename);
    new Notice(`Created ${filename}.`);
  }
  const tFile = tp.file.find_tfile(filename);
  const queryOutput = await dv.queryMarkdown(query);
  const fileContent = `---\npublish: true\n---\n%% update via "Update Publish Files" template %% \n\n${queryOutput.value}`;
  try {
    await app.vault.modify(tFile, fileContent);
    new Notice(`Updated ${tFile.basename}.`);
  } catch (error) {
    new Notice("⚠️ ERROR updating! Check console. Skipped file: " + filename , 0);
  }
});
openPublishPanel();
%>
```

## Result

On my Publish site, I have added five Dataview-powered query notes so
far: [Recently edited
notes](https://notes.joschua.io/60+Outputs/Recently+edited) and
[recently created
ones](https://notes.joschua.io/60+Outputs/Recent+new+files), books that
I am [currently
reading](https://notes.joschua.io/60+Outputs/Currently+Reading+%28Auto-Updating%29)
and that I [read
recently](https://notes.joschua.io/60+Outputs/Recently+read), as well as
a [list of 80+
books](https://notes.joschua.io/60+Outputs/Books+Read+%28Auto-Updating%29)
that I read.

![Obsidian Publish site showing a table of books with three columns:
Cover (showing the thumbnail), title and type (eg. non-fiction or
fiction)](how-to-burn-out-dataview-media/ec885732cad6b50bcb5f8a9d3ce0b0c7e261bacf.webp){width="2964"
height="1838" loading="lazy" decoding="async"}Dataview queries can be as
complex as you like, even including images and tags.

I embedded a few of those notes into the [homepage of my Publish
page](https://notes.joschua.io). Whenever I publish new notes with the
template, those files are automatically updated.

![Obsidian Publish site with the sections \"Recent new notes\" and
\"Recently edited\" and a table of notes underneath
each](how-to-burn-out-dataview-media/c6bf9baeb78927ac5e0c3e436930cb4a21f30cd0.webp){width="2964"
height="1838" loading="lazy" decoding="async"}

Now when a reader stumbles upon the page, they immediately see the most
recent notes --- with minimal upkeep on my end!
:::

::: {.grid-cols-12 .children-wrapper .is-metadata-grid astro-cid-bvzihdzo=""}
Tagged

-   [obsidian](/tags/obsidian){.italic astro-cid-bvzihdzo=""}
-   [dataview](/tags/dataview){.italic astro-cid-bvzihdzo=""}
-   [automation](/tags/automation){.italic astro-cid-bvzihdzo=""}

Published

::: {astro-cid-bvzihdzo=""}
Sep 1, 2023
:::
:::

::: {astro-cid-vqcxx4ny=""}
[↖ See all posts](/blog){astro-cid-vqcxx4ny=""}

[Subscribe with RSS](/feed.xml){astro-cid-vqcxx4ny=""}
:::
:::
