---
<%*  
let title = tp.file.title  
if (title.startsWith("Untitled")) {  
	title = await tp.system.prompt("Title new note");
	await tp.file.rename(`${title}`); 
} -%>
title: <% title %>
date: <% tp.file.creation_date("YYYY-MM-DD") %>
draft: false
description: Description of the page used for link previews.
tags:
  - 
---
