# F-2 — Different Windows File Explorer

> **Project status:** Initial design / pre-implementation  
> **Design version:** 0.2.0  
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
- **One source of navigation truth.** Tree and column modes are views over one shared navigation model.
- **Preserve state across views.** A view may hide state it cannot display, but displaying a view must not silently erase that state.
- **Separate presentation from layout.** Tree/column mode and horizontal/vertical arrangement are independent choices.
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
- Allow a branch to be collapsed without changing the active folder path.
- Support keyboard and mouse navigation.
- Avoid eagerly enumerating every descendant merely because a parent branch is visible.

This is the closest match to the Emacs Dired approach. Emacs Dired can insert a subdirectory listing into the current directory buffer; the community `dired-subtree` extension presents subdirectory contents directly below the directory entry as an indented tree.

References:
- [GNU Emacs — Dired](https://www.gnu.org/software/emacs/manual/html_node/emacs/Dired.html)
- [dired-subtree source](https://github.com/Fuco1/dired-hacks/blob/master/dired-subtree.el)

### 3.2 Miller Columns

The directory hierarchy is shown as a sequence of adjacent levels.

Expected behavior:

- The first column shows the chosen browsing root.
- Selecting a folder opens its contents in the next column.
- Selecting a deeper folder reveals another level.
- Ancestor levels remain represented as the user moves deeper.
- The selected path is visually clear.
- Selecting an earlier ancestor updates the levels to its right.
- The user can scroll or resize columns when the hierarchy exceeds the available space.

The visible column chain represents the path from the browsing root to the active folder. Sibling folders remain available in their parent column; they do not each create their own column.

This is the cascading-column pattern also associated with macOS Finder's column view. It is a particularly direct way to see the chain of parent folders and the current folder at once.

### 3.3 Shared Navigation State

Both presentations must be projections of one navigation model. They must not maintain independent versions of the current location, selection, expansion, or history.

The model should distinguish these concepts:

- **Browsing root:** the directory from which this browsing context begins. It may be a chosen folder, drive, or other supported filesystem location.
- **Active folder path:** the folder whose contents are currently being explored. This is the authoritative navigation location.
- **Focused item:** the item receiving keyboard focus.
- **Selected items:** the item or items selected for file-browser actions. Focus and selection are related but not necessarily identical.
- **Expanded-folder set:** folders the user has explicitly expanded in tree mode or otherwise chosen to keep expanded during the current browsing session.
- **Navigation history:** prior active folder paths for back and forward navigation.
- **Per-view presentation state:** view-specific details such as scroll positions and column widths. These do not redefine the filesystem location.

#### Mode-switch rules

1. **Active folder path is authoritative.** Switching modes never changes it.
2. **Columns are derived from the active path.** The visible column chain is the browsing root and each successive ancestor leading to the active folder. The final level displays the active folder's contents.
3. **Tree mode reveals the active path.** On switching to tree mode, the ancestor chain to the active folder is expanded so the active location is reachable and visible.
4. **Additional tree expansions are retained.** Switching to columns may hide expanded sibling branches because columns show one path at a time. Those branches remain in the expansion set and reappear when returning to tree mode, provided the folders still exist and remain accessible.
5. **Selection is preserved by item identity/path where possible.** If the focused or selected item is not visible in the destination view, retain its state without forcing the destination view to jump unexpectedly. If the item no longer exists or is no longer accessible, clear or reconcile that item and provide an understandable result.
6. **A view does not mutate hidden state.** A branch being invisible in columns does not mean it was collapsed in the shared model.
7. **User navigation is an intentional state change.** Selecting a different folder, changing the active path, or explicitly collapsing a branch may update the shared model. Merely switching presentation or orientation must not.

| State | Tree presentation | Miller Columns presentation | Switching rule |
|---|---|---|---|
| Browsing root | Root of the displayed hierarchy | First column | Preserve |
| Active folder path | Expanded ancestor chain leads to active folder | Determines visible column chain | Preserve exactly |
| Focused item | Focus shown in the relevant listing | Focus shown in the relevant column | Preserve by path when possible; do not force-scroll to hidden focus |
| Selected items | Selection shown wherever those items are visible | Selection shown if represented in the visible path/column | Retain valid selection even if temporarily not visible |
| Expanded-folder set | Multiple expanded branches can be shown | Only the active ancestor chain is shown | Retain branches hidden by column mode |
| Scroll positions | Tree scroll position | Per-column scroll positions | May be stored per view; restore when returning to that view |
| Navigation history | Back/forward through active paths | Same history | Shared |

If an item disappears, moves, or becomes inaccessible while F-2 is open, reconcile the state against the filesystem rather than treating stale paths as guaranteed to exist.

## 4. Layout and Orientation

The navigation presentation and physical arrangement are separate choices:

- **Horizontal tree:** the hierarchy expands downward with indentation; the listing scrolls vertically.
- **Vertical tree panel:** the tree can occupy a side panel or other vertical region while retaining the same indentation and expansion behavior.
- **Horizontal Miller Columns:** successive levels are arranged left to right.
- **Vertical Miller Columns:** successive levels are arranged top to bottom. Each level is a folder listing, and selecting a folder reveals the next level below it.
- **Resizable:** users can give more space to the part of the hierarchy they are working with.
- **Adaptable:** the layout should remain usable in narrow windows and on different monitor sizes.

#### Vertical Miller behavior

Vertical columns need an explicit space-management policy because height is more constrained than width:

- Each level should have a readable minimum size and a sensible maximum size.
- The active level and selected path should remain easy to locate.
- When space is insufficient, ancestor levels may switch to compact headers that preserve their identity and allow the user to reopen or focus them.
- Compacting an ancestor is a presentation change, not a navigation change; it must not alter the active path or selection.
- Scrolling should make the active level accessible without unexpectedly losing the user's place in other levels.
- Resizing boundaries should be draggable where practical.

The exact compacting and scrolling behavior should be validated with a prototype. The design does not require every ancestor listing to remain fully expanded on screen at all window sizes; it does require the ancestor chain to remain understandable and recoverable.

The initial implementation should establish a reliable default layout before adding elaborate window management. Horizontal Miller Columns are the simplest initial column baseline; vertical mode can follow once the shared navigation model is proven.

## 5. Shared File-Browser Behavior and Performance

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

### 5.1 Incremental Enumeration and Responsive Rendering

A large directory tree must not cause the interface to freeze or require F-2 to enumerate the entire filesystem before it can be used.

Design requirements:

- **On-demand enumeration:** read a folder's entries when that folder is first needed, not by recursively scanning every descendant in advance.
- **Lazy expansion:** expanding a folder requests that folder's contents. Deeper folders are not enumerated until they are themselves needed.
- **Asynchronous work:** filesystem enumeration must not block the UI thread. Long-running requests should be cancellable when the user changes location or no longer needs the result.
- **Incremental display:** show entries as they become available where practical, rather than waiting for an entire large listing before displaying anything.
- **Virtualized rendering:** prefer a UI approach that can render large lists without creating a full UI element for every entry at once.
- **Visible status:** show loading, partial results, and recoverable errors clearly. An inaccessible folder should not freeze or invalidate the rest of the browsing context.
- **Stale-result protection:** if the user navigates away while enumeration is in progress, late results must not overwrite the newer location's contents.
- **No arbitrary hidden truncation:** if a practical limit is required, make it visible and provide a clear way to continue or refine the listing.

Virtualization is a preferred capability to evaluate in the UI toolkit, not a reason to impose an arbitrary result cap. The implementation should be measured with representative large directories before performance claims are made.

## 6. Initial Scope

### First implementation target

Establish the core browsing experience before expanding into a full Explorer replacement:

1. Open a chosen starting directory.
2. Display its folders and files.
3. Navigate using expand-in-place tree mode.
4. Navigate using horizontal Miller Columns mode.
5. Switch between the two modes without losing the active folder path or valid selection/expansion state.
6. Establish the shared state model so vertical layout can be added without a second navigation engine.
7. Handle ordinary navigation errors without crashing or losing the current browsing state.
8. Keep enumeration and rendering responsive for large folders.

### Later possibilities

These are not commitments for the first implementation:

- Vertical Miller Columns and more advanced responsive layouts.
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

Before implementation, compare suitable Windows desktop approaches against the project's actual needs. The selected approach should be evaluated against these criteria:

- **Filesystem access:** reliable enumeration and navigation; asynchronous operations; cancellation; recoverable handling of inaccessible, removed, or changing paths.
- **Large-directory rendering:** support for incremental updates and list virtualization; no full-directory UI construction or blocking filesystem work on the UI thread.
- **State management:** a clear way to represent one shared navigation model and project it into both tree and column views.
- **Input:** a first-class keyboard focus model with predictable keyboard and pointer behavior for selection, expansion, and navigation.
- **Accessibility:** usable Windows accessibility support, including an accessible UI Automation (UIA) tree for controls and file entries. Avoid custom-drawn controls that cannot expose their structure and state.
- **Windows integration:** native filesystem behavior and platform integration where useful, without coupling the design unnecessarily to one framework.
- **Dependency hygiene:** minimal, maintained, auditable dependencies. Review provenance, license compatibility, update history, permissions, and security implications before adoption.
- **Safety boundary:** browsing and previewing must not execute files. Opening a file is a separate, explicit user action handled through a clear platform boundary.
- **Maintainability:** the simplest approach that meets the requirements, with a testable separation between filesystem operations, shared navigation state, and presentation.

Do not add a dependency merely because a reference project uses it. Compare candidate approaches against these criteria before selecting a stack, and explain meaningful trade-offs.

## 9. Acceptance Criteria for the Core Experience

The first usable version should demonstrate all of the following:

- [ ] The user can open a starting directory.
- [ ] The folder hierarchy is understandable at a glance.
- [ ] Tree mode expands and collapses folders inline.
- [ ] Expanding a child does not remove its parent or siblings from view.
- [ ] Column mode shows the ancestor chain alongside the active folder's contents.
- [ ] Selecting an ancestor updates the deeper navigation context correctly.
- [ ] Switching between tree and column modes preserves the active folder path.
- [ ] Switching modes does not discard expanded branches merely because the destination view cannot display them.
- [ ] Valid selection and focus state are preserved or reconciled predictably across mode changes.
- [ ] Horizontal tree and horizontal Miller layouts are usable.
- [ ] The shared model supports adding vertical arrangements without creating a separate navigation engine.
- [ ] Expanding a folder in a large directory keeps the interface responsive.
- [ ] Loading, partial results, cancellation, and inaccessible-folder errors are handled visibly and safely.
- [ ] Long paths and changing or removed filesystem entries are handled gracefully.
- [ ] Navigation does not depend on downloading or executing third-party file-manager projects.

These criteria describe intended behavior. They are not evidence that any feature has been implemented or tested. Vertical Miller layout has a design direction but remains a later implementation target, not a first-release acceptance requirement.

## 10. Development and Change Record

Use the project's adopted [Development State Machine Workflow](https://github.com/KRMRGold/Meta-Modules/blob/main/workflows/Development-State-Machine-Workflow.md) for substantial implementation work. Keep this README as the canonical living design document and update it in place. Use Git history for prior revisions; do not create version-suffixed copies.

The project should establish its implementation baseline, technology choices, test approach, and maintenance record when implementation begins. Until then, keep design decisions and open questions here rather than inventing implementation status.

### Current state

- **Design:** Refined initial design; shared navigation semantics and performance constraints documented.
- **Implementation:** Not started / not verified.
- **Tests:** None established or run.
- **Technology choice:** Open; evaluate candidates against §8.
- **Next step:** Compare suitable Windows desktop approaches against §8, then select a small prototype scope for the shared navigation model and the two initial horizontal presentations.
