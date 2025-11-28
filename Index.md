
```dataviewjs
const pages = dv.pages("").where(p => p.file.name !== "index");

const groups = pages.groupBy(p => p.file.folder);

for (const group of groups.sort(g => g.key)) {
  // Skip root folder if you want; comment this line out to show it.
  if (group.key === "" || group.key === "/") continue;

  dv.header(2, group.key);

  const items = group.rows.sort(r => r.file.name);

  dv.list(items.map(i => i.file.link));
}
```

END

