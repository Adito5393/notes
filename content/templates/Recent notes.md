<%*
const dv = app.plugins.plugins["dataview"].api;

// Add as many filenames and queries as you'd like!
const fileAndQuery = new Map([
  [
    "Recently edited",
    'TABLE WITHOUT ID file.link AS Note, dateformat(file.mtime, "ff") AS Modified FROM !("web_archive" OR "templates") WHERE !(draft OR skip_recent) SORT file.mtime desc LIMIT 7',
  ],
  [
    "Recent new files",
    'TABLE WITHOUT ID file.link AS Note, dateformat(file.ctime, "DD") AS Added FROM !("web_archive" OR "templates") WHERE !(draft OR skip_recent) SORT file.ctime desc LIMIT 7',
  ],
]);

await fileAndQuery.forEach(async (query, filename) => {
  if (!tp.file.find_tfile("./"+filename)) {
    await tp.file.create_new("", "./"+filename);
    new Notice(`Created ${filename}.`);
  }
  const tFile = tp.file.find_tfile("./"+filename);
  const queryOutput = await dv.queryMarkdown(query);
  const fileContent = `---\nskip_recent: true\n---\n${queryOutput.value}`;
  try {
    await app.vault.modify(tFile, fileContent);
    new Notice(`Updated ${tFile.basename}.`);
  } catch (error) {
    new Notice("⚠️ ERROR updating! Check console. Skipped file: " + filename , 0);
  }
});
-%>