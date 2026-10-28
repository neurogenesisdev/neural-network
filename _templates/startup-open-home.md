<%*
// Startup template: opens maps/home.md in reading view when Obsidian starts.
// Enable under Settings → Templater → Startup templates.
app.workspace.onLayoutReady(async () => {
  const path = "maps/home.md";
  const existing = app.workspace.getLeavesOfType("markdown").find(l => l.view?.file?.path === path);
  if (existing) {
    app.workspace.setActiveLeaf(existing, { focus: true });
    return;
  }
  await app.workspace.openLinkText(path, "", "tab", { state: { mode: "preview" } });
});
-%>
