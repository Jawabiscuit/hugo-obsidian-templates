<%*
// Begin Declarations
const dv = app.plugins.plugins["dataview"].api;

let title = tp.file.title;
let type = "journal";
let status = "ip";
let series = false;
let tags;
// End Declarations
-%>
<%*
// Begin Functions
function log(msg) {
    console.log(msg);
}

function capitalize_words (arr) {
    return arr.map(word => {
        return word.charAt(0).toUpperCase() + word.substring(1).toLowerCase();
    });
}
// End Functions
-%>
<%*
// Begin Prompts
title = await tp.system.prompt("Title", title.toLowerCase());

type = await tp.system.suggester(
    ["journal", "reference", "meeting", "document"],
    ["journal", "reference", "meeting", "document"],
);

// Set series
if (type == "journal") {
    series = true;
}
else if (type == "meeting") {
    answer = await tp.system.prompt("Series? (\"Y/n\")", "y");
    if (answer == "y") {
        series = true;
    }
}

status = await tp.system.suggester(
    items=["waiting", "in-progress", "finished", "hold", "complete", "blocked", "n/a"],
    text_items=["wtg", "ip", "fin", "hld", "cmpt", "blkd", "na"]
);

tags = await tp.system.prompt(
    "Tags (space separated)"
);

if (type == "meeting") {
    if (!tp.user.word_in_tags("meeting", tags)) {
        tags += " meeting";
    }
}
// End Prompts
-%>
<%*
// Begin Organization
await tp.file.rename(title + "-temp");

// File note into proper directory
// Journal
if (type == "journal") {
    await tp.file.move("journal/" + title);
}
// Reference
else if (type == "reference") {
    await tp.file.move("reference/" + title);
}
// Meetings
else if (type == "meeting") {
    await tp.file.move("meeting/" + title);
}
// Documents
else if (type == "document") {
    await tp.file.move("documents/" + title);
}
// End Organization
-%>
<%* tR += "---" %>
title: <%* tR += title %>
type: <% type %>
status: <% status %>
tags: <% tags %>
series: <% series %>
created: <% tp.date.now("YYYY-MM-DD HH:mm") %>
modification date: <% tp.file.last_modified_date("dddd Do MMMM YYYY HH:mm:ss") %>
<%* tR += "---" %>
<%*
// Begin Progress Bar
let progress;
let file_date = new Date(title.match(/^(\d{4}-\d{2}-\d{2})/)[1] + "T00:00");

if (series) {
    if (tp.user.word_in_tags("work", tags)) {
        // 5 workdays
        progress = tp.user.make_progress_bar(file_date.getDay(), 5, size=5, label="Progress");
    } else {
        // 7 weekdays
        progress = tp.user.make_progress_bar(file_date.getDay(), 7, size=7, label="Progress");
    }
}

if (progress) {
    tR += `${progress}\n`;
}
// End Progress Bar
-%>
<%*
// Begin Heading
let today = tp.date.now("YYYY-MM-DD", 0, title, "YYYY-MM-DD");

fname_without_date = title.split(today + "-")[1];
primary_heading = capitalize_words(fname_without_date.split("-")).join(" ");
// End Heading
-%>
# <% primary_heading %>
<%*
// Begin Navigation
let prev_note;
let next_note;

let folder = tp.file.folder(relative=true);

if (series) {
    prev_note = await dv.queryMarkdown(
       `LIST WITHOUT ID file.name
        FROM "${folder}"
        WHERE file.name != "${title}" AND file.day < date("${today}")
        WHERE regexreplace(file.name, "^([0-9]+-[0-9]+-[0-9]+)(.*)", "$2") = regexreplace("${title}", "^([0-9]+-[0-9]+-[0-9]+)(.*)", "$2")
        SORT file.day DESC
        LIMIT 1`
    );

    if (prev_note.value.length) {
        prev_note = prev_note.value.split(" ")[1].trim();
        prev_date = tp.user.prefixed_date(prev_note);
    } else {
        prev_date = tp.date.now("YYYY-MM-DD", -1, title, "YYYY-MM-DD")
        prev_note = `${prev_date}${title.split(today)[1]}`;
    }

    next_note = await dv.queryMarkdown(
       `LIST WITHOUT ID file.name
        FROM "${folder}"
        WHERE file.name != "${title}" AND file.day > date("${today}")
        WHERE regexreplace(file.name, "^([0-9]+-[0-9]+-[0-9]+)(.*)", "$2") = regexreplace("${title}", "^([0-9]+-[0-9]+-[0-9]+)(.*)", "$2")
        SORT file.day ASC
        LIMIT 1`
    );

    if (next_note.value.length) {
        next_note = next_note.value.split(" ")[1].trim();
    } else {
        next_date = await tp.system.prompt("Next date", tp.date.now("YYYY-MM-DD", 1, title, "YYYY-MM-DD"));
        next_note = `${next_date}${title.split(today)[1]}`;
    }
    
    tR += `◀ [[${prev_note}]] | [[${next_note}]] ▶\n`;
}
// End Navigation
-%>
<%*
// Begin Includes
if (tp.user.word_in_tags("standup", tags)) {
    tR += `
## 👷🚧 Updates

### ◀ [[${prev_note}#👷🚧 Updates|Previous]]

![[${prev_note}#^${prev_date}-updates]]

### 📆 Today

- 

^${today}-updates

> [!tldr] 
>

^${today}-tldr

## 💡 Capture

- 

^${today}-capture

## 💬 Spinoff Convo(s)

## 📥 Action Items

### ◀ [[${prev_note}#📥 Action Items|Previous]]

![[${prev_note}#^${prev_date}-action-items]]

### 📥 Today

- [ ] 

^${today}-action-items
`;
} else if (tp.user.word_in_tags("1on1", tags)) {
    tR += `
## ⌛ Prep

- 

## 🕚 Minutes
`;
    if (series) {
        tR += `
### ◀ [[${prev_note}#🕚 Minutes|Previous]]

![[${prev_note}#^${prev_date}-minutes]]

### 🕚 Today
`;
}
    tR += `
- 

^${today}-minutes

## 💡 Capture

- 

^${today}-capture

## 📥 Action Items

### ◀ [[${prev_note}#📥 Action Items|Previous]]

![[${prev_note}#^${prev_date}-action-items]]

### 📥 Today

- [ ] 

^${today}-action-items
`;
} else if (type == "journal") {
    tR += `
## 📓 Journal

## 🍗 Inputs

## 💩 Outputs

## 💡 Capture

## 📥 Action Items

## 🔗 Backlinks

`;
} else if (type == "meeting") {
    tR += `
## 👨‍💼 Attendance

## 🕚 Minutes

## 💡 Capture

## 📥 Action Items

`;
} else if (type == "reference") {
    tR += `
## 📥 Action Items

## 🔗 Backlinks

`;
}
// End Includes
-%>

