---
draft: true
---
# Recent notes - dataview

Recently edited:

```dataview
TABLE WITHOUT ID file.link AS Note, dateformat(file.mtime, "ff") AS Modified 
FROM !("web_archive" OR "templates") 
WHERE !(draft OR skip_recent)
SORT file.mtime DESC 
LIMIT 7
```

Recent new files:

```dataview
TABLE WITHOUT ID file.link AS Note, dateformat(file.ctime, "DD") AS Added 
FROM !("web_archive" OR "templates") 
WHERE !(draft OR skip_recent)
SORT file.ctime DESC 
LIMIT 7
```

