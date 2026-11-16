# F-2 — Different Windows File Explorer

> **Project status:** Initial design / pre-implementation  
> **Design version:** 0.1.0  
> **Target platform:** Windows  
> **Development workflow:** [Development State Machine Workflow](https://github.com/KRMRGold/Meta-Modules/blob/main/workflows/Development-State-Machine-Workflow.md), initially adopted at **v1.9.2**  
> **Implementation status:** No application implementation has been established by this document.

## 1. Purpose

F-2 is a Windows file browser focused on making folder hierarchy easy to see and explore.

The central problem with a conventional single-folder view is that navigating into a folder replaces the listing you were just looking at. To understand where you are, you often have to rely on breadcrumbs, reopen the parent, or repeatedly move backward and forward.

F-2 should let the user move through a directory hierarchy **without losing sight of the surrounding context**.

The project combines two established navigation patterns:

1. **Expand-in-place tree navigation**, inspired by Emacs Dired and `dired-subtree`.
2. **Miller Columns**, where selecting a folder opens its contents in the next column while ancestor folders remain visible.

These are two presentations of the same underlying folder hierarchy, not separate navigation systems.

## 2. Core Design Principles

- **Keep parent context visible.** Navigating deeper should not make the user forget or lose the path they came from.
- **Make hierarchy explorable.** Folders should be easy to expand, collapse, enter, and revisit.
- **Preserve navigation state.** Switching presentation or orientation should preserve the current location and selection where possible.
- **Let the user choose the layout.** The hierarchy should work in horizontal and vertical arrangements rather than assuming one screen shape.
- **Keep the interface direct.** File navigation should feel immediate; avoid unnecessary dialogs and layers.
- **Use ordinary Windows files and folders.** F-2 should browse the user's existing filesystem rather than require a new project format or database.
- **Build from documented behavior, not blind dependencies.** Study established interaction patterns, but implement the design deliberately. Do not download or execute unfamiliar projects simply to copy their behavior.

## 3. Navigation Presentations

### 3.1 Expand-in-Place Tree

A folder appears as an entry in its parent listing. Expanding it reveals its contents directly underneath, indented to show the hierarchy.

Expected behavior:

- Expand and collapse folders without leaving the current listing.
- Expand multiple branches at once.
- Keep parent and sibling entries visible while exploring a child.
- Show clear indentation and folder expansion state.
- Preserve expanded branches while navigating elsewhere in the same browsing session.
- Allow a branch to be collapsed without changing the current filesystem location.
- Support keyboard and mouse navigation.

This is the closest match to the Emacs Dired approach. Emacs Dired can insert a subdirectory listing into the current directory buffer; the community `dired-subtree` extension presents subdirectory contents directly below the directory entry as an indented tree.

References:
- [GNU Emacs — Dired](https://www.gnu.org/software/emacs/manual/html_node/emacs/Dired.html)
- [dired-subtree source](https://github.com/Fuco1/dired-hacks/blob/master/dired-subtree.el)

### 3.2 Miller Columns

The directory hierarchy is shown as adjacent columns.

Expected behavior:

- The first column shows the starting directory.
- Selecting a folder opens its contents in the next column.
- Selecting a deeper folder adds another column.
- Ancestor columns remain visible as the user moves deeper.
- The selected path is visually clear.
- Selecting an earlier ancestor updates the columns to its right.
- The user can scroll or resize columns when the hierarchy exceeds the available space.

This is the cascading-column pattern also associated with macOS Finder's column view. It is a particularly direct way to see the chain of parent folders and the current folder at once.

### 3.3 Shared Navigation State

Both presentations must represent the same filesystem location and selection state.

Switching between tree and columns should not unexpectedly reset the current folder. Where the layouts can represent the same expanded or selected branches, preserve that state during the switch.

The design should avoid implementing two unrelated navigation engines that drift apart.

## 4. Layout and Orientation

The hierarchy should support more than one arrangement:

- **Horizontal:** columns or panels arranged side by side.
- **Vertical:** levels or panels arranged above and below one another.
- **Resizable:** users can give more space to the part of the hierarchy they are working with.
- **Adaptable:** the layout should remain usable in narrow windows and on different monitor sizes.

The navigation presentation and the physical arrangement are separate choices. For example, a user may prefer expand-in-place tree navigation in a vertical panel, or Miller Columns across a wide window.

The initial implementation should establish a reliable default layout before adding elaborate window management.

## 5. Shared File-Browser Behavior

The following capabilities are useful to the overall design, but should be introduced in a practical order:

- Clear current-folder and selected-item indication.
- A readable path or breadcrumb for orientation and direct navigation.
- Back and forward navigation.
- Keyboard navigation and common file-selection behavior.
- Basic file operations using normal Windows filesystem behavior.
- A preview area for supported file types, if it can be added without making navigation cumbersome.
- Resizable panes and columns.
- Sensible handling of long names, hidden files, inaccessible folders, and large directories.

Previewing a file should not require opening it in its associated application. Opening a file should remain a distinct, explicit action.

## 6. Initial Scope

### First implementation target

Establish the core browsing experience before expanding into a full Explorer replacement:

1. Open a chosen starting directory.
2. Display its folders and files.
3. Navigate using expand-in-place tree mode.
4. Navigate using Miller Columns mode.
5. Switch between the two modes without losing the current location.
6. Support both horizontal and vertical layout arrangements where practical.
7. Handle ordinary navigation errors without crashing or losing the current browsing state.

### Later possibilities

These are not commitments for the first implementation:

- Tabs or saved browsing sessions.
- File previews and metadata panels.
- Multiple independent browsing panes.
- Drag-and-drop between folders or external applications.
- Customizable keyboard shortcuts.
- Search and filtering.
- Bookmarks or favorite locations.
- Detachable or independently movable panels.

Add these only when the core navigation model is working and the feature has a clear benefit.

## 7. Prior Art and Design References

F-2 should learn from established implementations while keeping its own implementation and scope deliberate.

| Reference | Useful idea | What to investigate |
|---|---|---|
| [GNU Emacs Dired](https://www.gnu.org/software/emacs/manual/html_node/emacs/Dired.html) | Directory listings as a navigable buffer; inserting subdirectory listings | How parent context and directory contents coexist |
| [dired-subtree](https://github.com/Fuco1/dired-hacks/blob/master/dired-subtree.el) | Expand and collapse a directory inline | Indentation, expansion state, and tree interaction |
| [SPAN Finder](https://lumibearstudio.github.io/) | Windows-oriented Miller Columns file browser | Column navigation and how a modern Windows app presents it |
| [OneCommander](https://www.onecommander.com/) | Windows file browsing with multiple layouts and panes | Layout flexibility and preview integration |
| [Dirwise](https://github.com/arcly-studio/dirwise) | Column-oriented Windows file browsing | Keyboard-first navigation and pane behavior |

These references are research material, not dependencies. F-2 does not need to install or incorporate their code to learn from the interaction patterns. If code is ever considered for reuse, inspect its provenance, license, dependencies, and security implications first.

## 8. Technical Direction — Not Yet Selected

No framework, language, UI toolkit, or filesystem library has been selected by this design.

Before implementation, compare suitable Windows desktop approaches against the project's actual needs:

- Reliable filesystem enumeration and navigation.
- Responsive rendering of large directories.
- Native Windows integration where useful.
- Accessible keyboard and pointer interaction.
- Maintainable implementation with minimal unnecessary dependencies.
- A clear boundary between browsing files and launching or executing them.

Do not add a dependency merely because a reference project uses it. Prefer the smallest suitable implementation and verify dependencies before adopting them.

## 9. Acceptance Criteria for the Core Experience

The first usable version should demonstrate all of the following:

- [ ] The user can open a starting directory.
- [ ] The folder hierarchy is understandable at a glance.
- [ ] Tree mode expands and collapses folders inline.
- [ ] Expanding a child does not remove its parent or siblings from view.
- [ ] Column mode shows ancestor folders alongside the selected folder's contents.
- [ ] Selecting an ancestor updates the deeper navigation context correctly.
- [ ] Switching between tree and column modes preserves the current location.
- [ ] Horizontal and vertical arrangements are usable.
- [ ] Long paths and inaccessible directories are handled gracefully.
- [ ] Navigation does not depend on downloading or executing third-party file-manager projects.

These criteria describe intended behavior. They are not evidence that any feature has been implemented or tested.

## 10. Development and Change Record

Use the project's adopted [Development State Machine Workflow](https://github.com/KRMRGold/Meta-Modules/blob/main/workflows/Development-State-Machine-Workflow.md) for substantial implementation work. Keep this README as the canonical living design document and update it in place. Use Git history for prior revisions; do not create version-suffixed copies.

The project should establish its implementation baseline, technology choices, test approach, and maintenance record when implementation begins. Until then, keep design decisions and open questions here rather than inventing implementation status.

### Current state

- **Design:** Initial direction recorded.
- **Implementation:** Not started / not verified.
- **Tests:** None established or run.
- **Next step:** Refine the interaction details and select a suitable Windows implementation approach before beginning the first navigation prototype.
