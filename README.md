# F-2 — Different Windows File Explorer

aa

> **Project status:** Initial design / pre-implementation  
> **Design version:** 0.6.1  
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
- Selecting an earlier ancestor updates the navigation levels beyond it.
- The user can scroll or resize columns when the hierarchy exceeds the available space.

The visible column chain represents the path from the browsing root to the active folder. Sibling folders remain available in their parent column; they do not each create their own column.

This is the cascading-column pattern also associated with macOS Finder's column view. It is a particularly direct way to see the chain of parent folders and the current folder at once.

### 3.3 Shared Navigation State

Both presentations must be projections of one navigation model. They must not maintain independent versions of the current location, selection, expansion, or history.

The model should distinguish these concepts:

- **Browsing root:** the directory from which this browsing context begins. It may be a chosen folder, drive, or other supported filesystem location. It is a navigation origin, not automatically a security boundary.
- **Active folder path:** the folder whose contents are currently being explored. This is the authoritative navigation location.
- **Focused item:** the item receiving keyboard focus.
- **Selected items:** the item or items selected for file-browser actions. Focus and selection are related but not necessarily identical.
- **Expanded-folder set:** folders the user has explicitly expanded in tree mode or otherwise chosen to keep expanded during the current browsing session. This records user intent, not temporary expansions required to reveal the active path.
- **Collapsed-folder overrides:** folders the user has explicitly collapsed during the current browsing session, including ancestors that would otherwise be expanded to reveal the active path. These overrides prevent automatic path-reveal behavior from immediately undoing an intentional collapse.
- **Navigation history:** prior active folder paths for back and forward navigation.
- **Per-view presentation state:** view-specific details such as scroll positions and column widths. These do not redefine the filesystem location.

#### Mode-switch rules

1. **Active folder path is authoritative.** Switching modes never changes it.
2. **Columns are derived from the active path.** The visible column chain is the browsing root and each successive ancestor leading to the active folder. The final level displays the active folder's contents.
3. **Tree mode reveals the active path where possible.** On switching to tree mode, the ancestor chain to the active folder is made reachable and visible unless a user-created collapsed-folder override intentionally hides part of that chain. The active folder remains authoritative even when its path is temporarily not expanded in the tree.
4. **Additional tree expansions are retained.** Switching to columns may hide expanded sibling branches because columns show one path at a time. Those branches remain in the expansion set and reappear when returning to tree mode, provided the folders still exist and remain accessible. Explicit collapse overrides are retained separately and take precedence over automatic path revelation.
5. **Selection is preserved by item identity/path where possible.** If the focused or selected item is not visible in the destination view, retain its state without forcing the destination view to jump unexpectedly. If the item no longer exists or is no longer accessible, clear or reconcile that item and provide an understandable result.
6. **A view does not mutate hidden state.** A branch being invisible in columns does not mean it was collapsed in the shared model.
7. **User navigation is an intentional state change.** Activating a different folder, changing the active path, or explicitly expanding/collapsing a branch may update the shared model. Merely switching presentation or orientation must not.
8. **Selection is not activation.** Focus, selection, activation, and active-folder location are separate concepts. A pointer or keyboard action may change focus or selection without changing the active folder unless the interaction is defined as activation for that presentation. Multi-selection must never implicitly activate every selected folder.

| State | Tree presentation | Miller Columns presentation | Switching rule |
|---|---|---|---|
| Browsing root | Root of the displayed hierarchy | First column | Preserve |
| Active folder path | Expanded ancestor chain leads to active folder | Determines visible column chain | Preserve exactly |
| Focused item | Focus shown in the relevant listing | Focus shown in the relevant column | Preserve by path when possible; do not force-scroll to hidden focus |
| Selected items | Selection shown wherever those items are visible | Selection shown if represented in the visible path/column | Retain valid selection even if temporarily not visible |
| Expanded-folder set | Multiple expanded branches can be shown | Only the active ancestor chain is shown | Retain branches hidden by column mode |
| Collapsed-folder overrides | Explicitly collapsed branches remain collapsed, including active-path ancestors | Not represented as expanded branches | Retain until explicitly expanded or session state is reset |
| Scroll positions | Tree scroll position | Per-column scroll positions | May be stored per view; restore when returning to that view |
| Navigation history | Back/forward through active paths | Same history | Shared |

If an item disappears, moves, or becomes inaccessible while F-2 is open, reconcile the state against the filesystem rather than treating stale paths as guaranteed to exist.

#### Focus, selection, activation, and active folder

These are separate state concepts and must not be conflated:

- **Focus** identifies the item that receives keyboard input.
- **Selection** identifies one or more items selected for a subsequent operation. Multi-selection is allowed where the presentation supports it.
- **Activation** is an intentional command on one item, such as Enter or double-click, or the equivalent explicit interaction defined by the view.
- **Active folder** is the single directory whose contents define the current navigation location.

Presentation-specific interaction rules:

- In **tree mode**, expanding or collapsing a folder changes expansion state only. It does not change the active folder. Activating a folder changes the active folder.
- In **Miller Columns**, selecting a folder may focus/select it and reveal its contents in the next column, but only the focused/activated folder in the current navigation chain becomes the active folder. Multi-selecting sibling folders must not create multiple active paths. The implementation must choose and document whether pointer selection itself is the activation gesture or whether a separate activation gesture is required; whichever rule is chosen must be consistent with keyboard interaction and tested.
- Selecting a file does not change the active folder. It changes focus/selection only.
- Back/forward navigation and direct path navigation explicitly change the active folder.

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
- Basic file operations using normal Windows filesystem behavior are a later capability. Copy, move, delete, and rename are outside the initial prototype scope (§9.2).
- A preview area for supported file types, if it can be added without making navigation cumbersome.
- Resizable panes and columns.
- Sensible handling of long names, hidden files, inaccessible folders, and large directories.

Previewing a file should not require opening it in its associated application. Opening a file should remain a distinct, explicit action.

### 5.1 Incremental Enumeration and Responsive Rendering

A large directory tree must not cause the interface to freeze or require F-2 to enumerate the entire filesystem before it can be used.

Design requirements:

- **On-demand enumeration:** read a folder's entries when that folder is first needed, not by recursively scanning every descendant in advance.
- **Reparse-point policy:** make navigation through symbolic links, junctions, and other reparse points explicit. The user may intentionally navigate through a link, but F-2 must identify the resolved target where practical, detect cycles during any traversal, and avoid silently treating a link target as a child physically contained by the browsing root. The browsing root alone does not confine navigation. No recursive traversal is needed for the initial prototype.
- **Lazy expansion:** expanding a folder requests that folder's contents. Deeper folders are not enumerated until they are themselves needed.
- **Asynchronous work:** filesystem enumeration must not block the UI thread. Long-running requests should be cancellable when the user changes location or no longer needs the result.
- **Incremental display:** show entries as they become available where practical, rather than waiting for an entire large listing before displaying anything.
- **Virtualized rendering:** prefer a UI approach that can render large lists without creating a full UI element for every entry at once. Verify the actual virtualization behavior of each selected control; lazy data loading is not the same as UI virtualization. WinUI 3's built-in `TreeView` is not virtualized, so it is a prototype candidate rather than an assumed solution for very large trees. Measure it against representative directories and evaluate a virtualized custom tree if it fails the responsiveness requirement.
- **Visible status:** show loading, partial results, and recoverable errors clearly. An inaccessible folder should not freeze or invalidate the rest of the browsing context.
- **Stale-result protection:** if the user navigates away while enumeration is in progress, late results must not overwrite the newer location's contents.
- **No arbitrary hidden truncation:** if a practical limit is required, make it visible and provide a clear way to continue or refine the listing.

Virtualization is a preferred capability to evaluate in the UI toolkit, not a reason to impose an arbitrary result cap. The implementation should be measured with representative large directories before performance claims are made.

Filesystem enumeration and UI virtualization are separate concerns. Incremental enumeration can populate a collection progressively, but a control may still create one UI element per realized entry. The prototype must measure both data-loading responsiveness and rendering responsiveness. If a synchronous filesystem API is used on a background worker, cancellation may only be observed between filesystem calls; do not promise immediate interruption of an operating-system call already in progress.

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

## 8. Technical Direction — Selected for Prototype

A prototype stack has been selected (§8.1). The criteria below describe the capabilities the stack must satisfy and the conditions that would justify revisiting the decision.

Evaluate the selected stack against these criteria as the prototype is built. Reconsider alternatives only if evidence reveals a material mismatch:

- **Filesystem access:** reliable enumeration and navigation; asynchronous operations; cancellation; recoverable handling of inaccessible, removed, or changing paths.
- **Large-directory rendering:** support for incremental updates and list virtualization; no full-directory UI construction or blocking filesystem work on the UI thread.
- **State management:** a clear way to represent one shared navigation model and project it into both tree and column views.
- **Input:** a first-class keyboard focus model with predictable keyboard and pointer behavior for selection, expansion, and navigation.
- **Accessibility:** usable Windows accessibility support, including an accessible UI Automation (UIA) tree for controls and file entries. Avoid custom-drawn controls that cannot expose their structure and state.
- **Windows integration:** native filesystem behavior and platform integration where useful, without coupling the design unnecessarily to one framework.
- **Dependency hygiene:** minimal, maintained, auditable dependencies. Review provenance, license compatibility, update history, permissions, and security implications before adoption.
- **Safety boundary:** browsing and previewing must not execute files. Opening a file is a separate, explicit user action handled through a clear platform boundary.
- **Maintainability:** the simplest approach that meets the requirements, with a testable separation between filesystem operations, shared navigation state, document buffers, and presentation.

Do not add a dependency merely because a reference project uses it. Keep the selected stack minimal; introduce a dependency only when it materially helps satisfy these criteria, and explain the trade-off.


### 8.1 Technology decision

**Selected prototype stack: WinUI 3 + C# + .NET 10 LTS + Windows App SDK.**

This is a Windows-only project, and the priorities are balanced performance, maintainability, and good Windows integration. WinUI 3 is Microsoft's recommended native UI framework for new Windows desktop applications. C#/.NET offers a practical application architecture and direct access to Windows APIs without introducing a separate web frontend and native backend.

Use **.NET 10 LTS** as the prototype target rather than copying SPAN Finder's current .NET 8 target. As of this decision (2026-11-16), .NET 8 reached end of support on 2026-11-10, while .NET 10 LTS is supported through 2028-11-14. .NET 11 is newer but is a Standard Term Support release; .NET 10 LTS is the more conservative baseline for this project. Microsoft's current WinUI documentation lists C# with .NET 8 or later as supported.

At the time of this design revision (November 2026), Microsoft's developer documentation identified Windows App SDK 2.6.5 as the latest stable release. Treat that as a dated reference, not a version pin: confirm the latest stable release and its compatibility with the actual project template, controls, deployment model, and package configuration when scaffolding. Record the exact Windows App SDK and Windows SDK versions selected.

**Prior-art reference:** SPAN Finder is a close implementation reference for WinUI 3 file browsing and Miller Columns. Its published project configuration uses .NET 8 and Windows App SDK 1.8, and its feature reference documents MVVM, asynchronous directory loading, cancellation, and multiple folder views. Treat it as evidence that this UI approach is practical—not as a dependency or a version template. Its .NET 8 target is now out of support, and its GPL-3.0 license means its source must not be copied into F-2 without a deliberate license review.

**Alternatives retained for comparison, not active candidates:**
- **Qt 6:** strong model/view architecture and filesystem model; reconsider only if the WinUI prototype reveals a concrete limitation that outweighs C++ and Qt deployment/licensing costs.
- **Avalonia:** capable .NET UI framework, but cross-platform support is not a current requirement.
- **Tauri:** credible Rust + web UI approach, demonstrated by Dirwise, but adds a frontend/native boundary that provides little value for this Windows-only prototype.

This is a **prototype technology decision**, not a claim that performance, accessibility, or deployment suitability has already been proven. Revisit it only if the prototype produces specific evidence against the criteria in this section.

References:
- [Microsoft — WinUI 3](https://learn.microsoft.com/en-us/windows/apps/winui)
- [Microsoft — Windows app development](https://learn.microsoft.com/en-us/windows/apps/)
- [Microsoft — .NET support policy](https://dotnet.microsoft.com/en-us/platform/support/policy)
- [Microsoft — Windows developer updates](https://learn.microsoft.com/en-us/windows/apps/whats-new/whats-new-for-developers)
- [SPAN Finder source repository](https://github.com/LumiBearStudio/SpanFinder)
- [Dirwise source repository](https://github.com/arcly-studio/dirwise)

### 8.2 Architecture boundaries for the prototype

Keep four responsibilities separate:

1. **Filesystem service:** enumerate immediate children of a directory, return stable path-based entries and metadata needed by the views, report recoverable errors, and honor cancellation. Provide the underlying filesystem operations used by navigation and document I/O. Define how reparse points are identified and how their targets are handled; do not assume the browsing root is a containment boundary.
2. **Shared navigation state:** own the browsing root, active folder path, focus, selection, expanded-folder set, and navigation history. Derive the Miller column chain from this state. The editor must not own or redefine the file browser's active folder path.
3. **Document/buffer service:** own open document buffers, per-buffer content, associated file paths, dirty state, and save/close lifecycle. Coordinate document loading, saving, external-change detection, and unsaved-change protection. Expose clear open, save, and close operations with change notifications. Keep document state independent of navigation state.
4. **Presentation:** render the tree, horizontal Miller view, and integrated editor from their respective state models. Store view-specific scroll positions, column sizing, and other presentation details separately. Presentation changes must not silently mutate navigation or document state.

Neither browser presentation should own filesystem navigation state, and neither the editor nor its buffers should become a second source of truth for the active browsing location.

### 8.3 Integrated editing direction

F-2 also includes a lightweight, Emacs-inspired text-editing workspace inside the same application window. The intent is to make browsing and editing parts of one continuous workflow—not to build a full IDE or reproduce all of Emacs.

**Initial editing model:**
- Opening a supported text file creates or focuses a document buffer in the editor area.
- Keep the file browser available alongside the editor, with a layout that can be resized or temporarily collapsed.
- Opening a file must not discard the current browsing location, tree expansions, or column context.
- Support multiple open buffers in the initial design, with a clear way to switch between them.
- Track whether each buffer has unsaved changes and identify the associated file path.
- Provide basic text editing, selection, clipboard operations, undo/redo, and save.
- Provide an explicit close action and protect unsaved changes with a clear save/discard/cancel decision.
- Opening a file is an explicit action; merely selecting or previewing a file must not execute it or launch an associated application.
- Initially support ordinary text files only. Treat binary, very large, or unsupported files as non-editable and explain why rather than interpreting arbitrary bytes as text.
- Preserve line endings and encoding where practical; detect and report unsupported or ambiguous encodings rather than silently corrupting content.
- Define and test an initial supported text baseline before implementation: UTF-8 with and without BOM, and LF and CRLF line endings. Mixed line endings and other encodings must be detected and either preserved deliberately or reported as unsupported; never silently normalize them.
- Establish a conservative maximum editable file size through prototype measurements. Files exceeding the supported limit must remain available to browse but must not be loaded into the editor as if they were ordinary small text files.
- Saving should use a safe replacement strategy appropriate to Windows filesystem semantics. A failed or interrupted save must not silently destroy the prior on-disk file or the in-memory buffer.
- Detect when a file changes externally after it was opened, including replacement of the file at the same path. Compare more than a timestamp where practical (for example, file identity and a content/version fingerprint). Before overwriting an externally changed file, surface the conflict and let the user decide how to proceed.
- Saving writes to the original file path. Save As can be considered after the basic open/edit/save path is reliable.

**Emacs-inspired, but not Emacs scope:** buffers, keyboard-oriented navigation, direct file opening, and keeping multiple pieces of work available are the useful concepts. A Lisp environment, extensible command system, packages, macros, terminals, language servers, and full Emacs keybinding compatibility are not part of the initial target.

**WinUI control note:** Microsoft's WinUI guidance distinguishes a plain multiline `TextBox` for plain text from `RichEditBox` for formatted documents. Start by evaluating `TextBox` for the plain-text editing prototype; do not choose `RichEditBox` just because its name sounds more capable. Validate large-file behavior, undo/redo, keyboard handling, and encoding/newline preservation before committing to the editor control.

References:
- [Microsoft — TextBox](https://learn.microsoft.com/en-us/windows/windows-app-sdk/api/winrt/microsoft.ui.xaml.controls.textbox)
- [Microsoft — RichEditBox](https://learn.microsoft.com/en-us/windows/windows-app-sdk/api/winrt/microsoft.ui.xaml.controls.richeditbox)
- [GNU Emacs — Buffers](https://www.gnu.org/software/emacs/manual/html_node/emacs/Buffers.html)
- [GNU Emacs — Visiting Files](https://www.gnu.org/software/emacs/manual/html_node/emacs/Visiting.html)

Avoid adding a database, indexing service, file-operation engine, shell extension layer, or third-party file-manager dependency to prove the navigation and editing design.


## 9. Prototype Scope and Verification

The first candidate is a focused navigation-and-editing prototype, not a complete file manager or IDE.

### 9.1 Included

1. Start at a user-selected ordinary filesystem directory.
2. Enumerate that directory's immediate children and display folders and files.
3. Provide an expand-in-place tree presentation.
4. Provide horizontal Miller Columns, where selecting a folder reveals its contents in the next column.
5. Drive both presentations from one shared navigation state.
6. Preserve the active path, valid focus/selection, and hidden tree expansion state when switching presentations.
7. Load directory contents on demand, asynchronously, and with cancellation.
8. Prevent stale enumeration results from replacing the contents of a newer active location.
9. Show loading and recoverable access/error states without losing the last valid navigation context.
10. Open a supported text file inside the same application window.
11. Edit and save the text file at its original path.
12. Keep the browser's navigation context intact while editing.
13. Open more than one text-file buffer and switch between them.
14. Track unsaved changes and protect them when closing a buffer or the application.
15. Detect an external file modification before saving over a changed on-disk version.
16. Avoid silently changing the encoding or line-ending convention of supported text files during ordinary open/edit/save operations. Surface unsupported or ambiguous formats rather than silently rewriting them.

### 9.2 Explicitly excluded

Do not add copy/move/delete/rename operations, previews, thumbnails, search, bookmarks, remote filesystems, cloud-provider integration, shell context menus, custom file icons, syntax highlighting, language servers, terminal integration, plugin systems, or vertical Miller layout to this prototype. Multiple document buffers are in scope, but full workspace/session persistence and advanced editor features are not. These features would obscure whether the shared navigation and editing foundations work.

### 9.3 Verification plan

Build the smallest useful verification set after inspecting the scaffold and available test tooling:

- **Navigation-state tests:** active path remains authoritative; column levels derive from the root-to-active path; switching views retains expanded branches; focus and selection are reconciled by path; presentation changes do not mutate navigation state.
- **Filesystem-service tests:** immediate-child enumeration, empty folders, inaccessible or removed paths, cancellation, and late-result/stale-request protection. Use isolated temporary directories where practical.
- **UI smoke checks:** open a chosen folder, navigate several levels, switch tree/columns repeatedly, return to an ancestor, and verify focus/selection behavior.
- **Responsiveness check:** exercise a representative large directory and observe whether input remains responsive and entries appear without waiting for a full recursive scan. Record the test conditions; do not claim a performance threshold until it has been measured.
- **Editor/document tests:** establish the basic open/edit/save/switch lifecycle; verify that opening loads expected content, editing marks the buffer dirty, saving writes the expected content, closing a dirty buffer requires an explicit decision, switching buffers preserves each buffer's content and dirty state, external modifications are detected before overwrite, and unsupported/binary input is not silently saved as text. These checks prove the intended prototype loop; they do not establish comprehensive editor compatibility.
- **Basic text-safety checks:** use representative fixtures for UTF-8 with and without BOM and LF/CRLF line endings. Verify open/edit/save fidelity, explicit handling of mixed line endings and unsupported encodings, and safe behavior for oversized files. Defer a broader encoding and newline compatibility matrix until after the first prototype. Record limitations and formats not supported.
- **Tree rendering check:** measure built-in WinUI `TreeView` with a representative large directory, separately from enumeration timing. If responsiveness or memory use is unacceptable, evaluate a virtualized custom tree before expanding scope.

Do not choose or add a separate test framework before inspecting the actual project scaffold. Record commands and results only after they have been run on the candidate.

### 9.4 Prototype completion gate

The prototype is successful when:

- Both navigation views remain consistent projections of the same shared navigation state.
- Switching presentations does not silently discard valid navigation state.
- Filesystem enumeration remains responsive and stale results cannot overwrite the current location.
- A user can open, edit, save, and switch between ordinary text-file buffers without losing browser context.
- Unsaved changes are protected, and external modifications are detected before overwriting.
- The supported baseline text format can be saved without silent encoding or line-ending corruption.

The gate establishes the core navigation and editing workflows, not comprehensive file-manager or text-editor compatibility. Broader encoding compatibility, advanced editor features, and vertical Miller layout remain future work.

## 10. Acceptance Criteria for the Core Experience

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
- [ ] The shared navigation model can support future vertical arrangements without requiring a separate navigation engine. This is a model-design requirement, not a requirement to implement a vertical Miller view in the prototype.
- [ ] Expanding a folder in a large directory keeps the interface responsive.
- [ ] Loading, partial results, cancellation, and inaccessible-folder errors are handled visibly and safely.
- [ ] Long paths and changing or removed filesystem entries are handled gracefully.
- [ ] Symbolic links, junctions, and other reparse points have an explicit navigation policy; the browsing root is not mistakenly treated as a security boundary.
- [ ] Focus, selection, activation, and active-folder changes behave consistently, including multi-selection.
- [ ] Explicitly collapsed tree branches are not immediately reopened by automatic active-path revelation.
- [ ] Tree enumeration and tree UI rendering are measured separately; the chosen tree control meets the prototype's responsiveness needs.
- [ ] Navigation does not depend on downloading or executing third-party file-manager projects.
- [ ] A supported text file can be opened inside the F-2 window.
- [ ] The browser remains available while a document is being edited.
- [ ] Opening a document does not change the active folder path or discard navigation context.
- [ ] Multiple document buffers can remain open and be switched between.
- [ ] Unsaved changes are clearly indicated and protected when closing a buffer.
- [ ] Saving writes to the intended file path and reports failures without losing the buffer's content.
- [ ] External modifications are detected before an overwrite.
- [ ] Unsupported or binary files are not silently treated as editable text.

These criteria describe intended behavior. They are not evidence that any feature has been implemented or tested. Vertical Miller layout has a design direction but remains a later implementation target, not a first-release acceptance requirement.

## 11. Development and Change Record

Use the project's adopted [Development State Machine Workflow](https://github.com/KRMRGold/Meta-Modules/blob/main/workflows/Development-State-Machine-Workflow.md) for substantial implementation work. Keep this README as the canonical living design document and update it in place. Use Git history for prior revisions; do not create version-suffixed copies.

The project should establish its implementation baseline, technology choices, test approach, and maintenance record when implementation begins. Until then, keep design decisions and open questions here rather than inventing implementation status.

### Workflow adoption and project-specific application

F-2 initially adopted **Development State Machine Workflow v1.9.2**. This records the version used when the project adopted the workflow; it is not a permanent historical pin. Follow the current canonical workflow at the linked Meta-Modules path by default unless a deliberate deviation is recorded here.

Apply the workflow proportionately:

- **S1–S3 — Design and readiness:** maintain this README as the single canonical design; define observable acceptance checks; resolve scope and architecture contradictions before implementation; identify the current accepted baseline and the exact repository branch before writing.
- **S4 — Implementation:** create a scoped scaffold/candidate on the existing branch, preserve the baseline, and assign a separate SemVer sequence to the application code/build artifact. Record exact .NET SDK, Windows App SDK, Windows SDK, project template, and deployment configuration actually selected.
- **S5–S7 — Review, verification, and acceptance:** inspect the candidate diff; run the applicable tests and build/runtime checks; record PASS, FAIL, BLOCKED, or NOT RUN against the exact candidate. Do not treat a scaffold build as proof of navigation or editor behavior.
- **S8–S10 — Repair, maintenance, and next decision:** preserve failed candidate evidence, repair in a new code version, and update a single canonical maintenance record only after the agreed acceptance gate passes.
- **Pause/resume:** use the workflow's §10 checkpoint procedure and the [Context Continuity module](https://github.com/KRMRGold/Meta-Modules/blob/main/modules/context-continuity.md). Update this design or the future maintenance/test record that best fits the state; do not create per-session checkpoint files.

Load these supporting Meta-Modules at the corresponding development stages rather than duplicating their procedures here:

- [Testing and Verification](https://github.com/KRMRGold/Meta-Modules/blob/main/modules/testing-and-verification.md) — derive the behavior-to-test map, choose the smallest useful test layers, and record actual execution evidence.
- [Code Review and Change Assurance](https://github.com/KRMRGold/Meta-Modules/blob/main/modules/code-review-and-change-assurance.md) — review candidate diffs, data flow, architecture, scope, failure behavior, and security findings before acceptance.
- [Cybersecurity and Secure Engineering](https://github.com/KRMRGold/Meta-Modules/blob/main/modules/cybersecurity-and-secure-engineering.md) — threat-model filesystem access, untrusted filenames/content, path and symbolic-link handling, file writes, dependency provenance, and application boundaries before implementation.
- [Sandboxing and Safe Execution](https://github.com/KRMRGold/Meta-Modules/blob/main/modules/sandboxing-and-safe-execution.md) — classify and constrain commands, generated code, dependency installation, and test execution before running them.

These are stage-specific supporting modules, not additional workflow versions or permanent pins. Retrieve their current canonical contents when the relevant stage begins.

### Current state

- **Design:** Refined initial design; shared navigation semantics and performance constraints documented.
- **Implementation:** Not started / not verified.
- **Tests:** None established or run.
- **Technology choice:** WinUI 3 + C# + .NET 10 LTS selected for the prototype; exact Windows App SDK version to be confirmed when scaffolding.
- **Prototype scope:** Navigation plus basic integrated text editing defined in §9; implementation has not started.
- **Workflow phase:** S2 — Design refinement. The design is being tightened before the implementation-readiness gate; implementation has not begun.
- **Next step:** Review the v0.6.1 refinements against the agreed design, then enter S3 — Ready to implement only when the scope and acceptance checks are coherent, the current repository baseline is identified, and the Windows development environment and available test tooling have been inspected. Scaffolding is the first S4 implementation action, not evidence that the navigation or editor features are implemented.
