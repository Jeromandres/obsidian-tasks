<script lang="ts">
    import { onMount } from 'svelte';
    import { defaultEditModalShowSettings } from '../Config/EditModalShowSettings';

    import { TASK_FORMATS, getSettings } from '../Config/Settings';
    import type { Status } from '../Statuses/Status';
    import type { Task } from '../Task/Task';
    import { settingsStore } from './SettingsStore';
    import DateEditor from './DateEditor.svelte';
    import Dependency from './Dependency.svelte';
    import { EditableTask } from './EditableTask';
    import { labelContentWithAccessKey } from './EditTaskHelpers';
    import PriorityEditor from './PriorityEditor.svelte';
    import RecurrenceEditor from './RecurrenceEditor.svelte';
    import StatusEditor from './StatusEditor.svelte';
    import type { App, TFile } from 'obsidian';
    import { SuggestModal } from 'obsidian';

    // These exported variables are passed in as props by TaskModal.onOpen():
    export let app: App; // <-- AJOUT
    export let task: Task;
    export let onSubmit: (updatedTasks: Task[]) => void | Promise<void>;
    export let statusOptions: Status[];
    export let allTasks: Task[];

    const {
        // NEW_TASK_FIELD_EDIT_REQUIRED
        startDateSymbol,
        scheduledDateSymbol,
        dueDateSymbol,
        cancelledDateSymbol,
        createdDateSymbol,
        doneDateSymbol,
    } = TASK_FORMATS.tasksPluginEmoji.taskSerializer.symbols;

    let descriptionInput: HTMLTextAreaElement;

    let editableTask = EditableTask.fromTask(task, allTasks);

    let isDescriptionValid: boolean = true;

    let isCancelledDateValid: boolean = true;
    let isCreatedDateValid: boolean = true;
    let isDoneDateValid: boolean = true;
    let isDueDateValid: boolean = true;
    let isScheduledDateValid: boolean = true;
    let isStartDateValid: boolean = true;

    let isRecurrenceValid: boolean = true;

    let withAccessKeys: boolean = true;
    let formIsValid: boolean = true;

    let mountComplete = false;

    $: accesskey = (key: string) => (withAccessKeys ? key : null);
    $: formIsValid =
        isDueDateValid &&
        isRecurrenceValid &&
        isScheduledDateValid &&
        isStartDateValid &&
        isDescriptionValid &&
        isCancelledDateValid &&
        isCreatedDateValid &&
        isDoneDateValid;
    $: isDescriptionValid = editableTask.description.trim() !== '';

    $: isShownInEditModal = { ...defaultEditModalShowSettings, ...$settingsStore.isShownInEditModal };

    onMount(() => {
        settingsStore.set(getSettings());

        const { provideAccessKeys } = getSettings();
        withAccessKeys = provideAccessKeys;

        mountComplete = true;

        setTimeout(() => {
            descriptionInput.focus();
        }, 10);
    });

    const _onClose = () => {
        onSubmit([]);
    };

    const _onDescriptionKeyDown = (e: KeyboardEvent) => {
        if (e.key === 'Enter' && !e.isComposing) {
            e.preventDefault();
            if (formIsValid) _onSubmit();
        }
    };

    // this is called, when text is pasted or dropped into
    // the description field, to remove any linebreaks
    const _removeLinebreaksFromDescription = () => {
        // wrapped into a timer to run after the paste/drop event
        setTimeout(() => {
            editableTask.description = editableTask.description.replace(/[\r\n]+/g, ' ');
        }, 0);
    };

    const _onSubmit = async () => {
        const newTasks = await editableTask.applyEdits(task, allTasks);
        onSubmit(newTasks);
    };

    type Trigger =
    | { kind: 'link'; from: number; to: number; query: string }
    | { kind: 'tag'; from: number; to: number; query: string };

    function _findTrigger(text: string, cursor: number): Trigger | null {
        const before = text.slice(0, cursor);
        const windowStart = Math.max(0, before.length - 300);
        const w = before.slice(windowStart);
    
        // [[ ... (not yet closed)
        const linkIdx = w.lastIndexOf('[[');
        if (linkIdx >= 0) {
            const after = w.slice(linkIdx + 2);
            if (!after.includes(']]')) {
                return { kind: 'link', from: windowStart + linkIdx, to: cursor, query: after };
            }
        }
    
        // #tag (with a reasonable boundary)
        const m = w.match(/(^|[\s([{"'.,;:!?])#([^\s#()[\]{}"',;:!?]*)$/);
        if (m) {
            const hashPos = w.lastIndexOf('#');
            return { kind: 'tag', from: windowStart + hashPos, to: cursor, query: m[2] ?? '' };
        }
    
        return null;
    }
    
    function _replaceRange(value: string, from: number, to: number, insert: string) {
        return value.slice(0, from) + insert + value.slice(to);
    }
    
    class _LinkSuggestModal extends SuggestModal<TFile> {
        constructor(app: App, private files: TFile[]) {
            super(app);
            this.setPlaceholder('Lien…');
        }
        getSuggestions(query: string): TFile[] {
            const q = (query ?? '').toLowerCase();
            const hits = q
                ? this.files.filter((f) => f.basename.toLowerCase().includes(q) || f.path.toLowerCase().includes(q))
                : this.files;
            return hits.slice(0, 50);
        }
        renderSuggestion(file: TFile, el: HTMLElement) {
            el.createDiv({ text: file.path });
        }
        onChooseSuggestion(file: TFile) {
            this.onChoose?.(file);
        }
        onChoose?: (file: TFile) => void;

        public setQuery(query: string) {
          // SuggestModal a bien inputEl au runtime, mais les typings peuvent être incomplets
          // eslint-disable-next-line @typescript-eslint/no-explicit-any
          const self: any = this;
          if (self.inputEl) {
            self.inputEl.value = query;
            self.inputEl.dispatchEvent(new Event("input"));
          }
        }
    }
    
    class _TagSuggestModal extends SuggestModal<string> {
        constructor(app: App, private tags: string[]) {
            super(app);
            this.setPlaceholder('Tag…');
        }
        getSuggestions(query: string): string[] {
            const q = (query ?? '').toLowerCase();
            const hits = q ? this.tags.filter((t) => t.toLowerCase().includes(q)) : this.tags;
            return hits.slice(0, 50);
        }
        renderSuggestion(tag: string, el: HTMLElement) {
            el.createDiv({ text: `#${tag}` });
        }
        onChooseSuggestion(tag: string) {
            this.onChoose?.(tag);
        }
        onChoose?: (tag: string) => void;

        public setQuery(query: string) {
          // SuggestModal a bien inputEl au runtime, mais les typings peuvent être incomplets
          // eslint-disable-next-line @typescript-eslint/no-explicit-any
          const self: any = this;
          if (self.inputEl) {
            self.inputEl.value = query;
            self.inputEl.dispatchEvent(new Event("input"));
          }
        }
    }
    
    let _isSuggestOpen = false;
    
    function _getAllTags(): string[] {
      const tags = new Set<string>();
    
      for (const file of app.vault.getMarkdownFiles()) {
        const cache = app.metadataCache.getFileCache(file);
        const rawTags = cache?.tags ?? [];
        for (const t of rawTags) {
          const tag = (t.tag ?? "").replace(/^#/, "");
          if (tag) tags.add(tag);
        }
      }
    
      return [...tags].sort();
    }
    
    function _openSuggestForDescriptionIfNeeded(e: KeyboardEvent) {
        if (_isSuggestOpen) return;
    
        // déclenchement keyup demandé, filtrage touches non-textes
        if (e.key.length > 1 && e.key !== 'Backspace') return;
    
        if (!descriptionInput) return;
    
        const cursor = descriptionInput.selectionStart ?? 0;
        const trig = _findTrigger(descriptionInput.value, cursor);
        if (!trig) return;
    
        _isSuggestOpen = true;
    
        if (trig.kind === 'link') {
            const modal = new _LinkSuggestModal(app, app.vault.getMarkdownFiles());
            modal.setQuery(trig.query);
            modal.onChoose = (file) => {
                const insert = `[[${file.basename}]]`; // <-- choix acté
                const newVal = _replaceRange(descriptionInput.value, trig.from, trig.to, insert);
                const newCursor = trig.from + insert.length;
    
                editableTask.description = newVal; // <-- important: Svelte store la vérité ici
                // le bind:value mettra à jour le textarea, mais on recale aussi le curseur
                setTimeout(() => {
                    descriptionInput.setSelectionRange(newCursor, newCursor);
                    descriptionInput.focus();
                }, 0);
            };
            modal.onClose = () => {
                _isSuggestOpen = false;
                descriptionInput?.focus();
            };
            modal.open();
            return;
        }
    
        if (trig.kind === 'tag') {
            const modal = new _TagSuggestModal(app, _getAllTags());
            modal.setQuery(trig.query);
            modal.onChoose = (tag) => {
                const insert = `#${tag}`;
                const newVal = _replaceRange(descriptionInput.value, trig.from, trig.to, insert);
                const newCursor = trig.from + insert.length;
    
                editableTask.description = newVal;
                setTimeout(() => {
                    descriptionInput.setSelectionRange(newCursor, newCursor);
                    descriptionInput.focus();
                }, 0);
            };
            modal.onClose = () => {
                _isSuggestOpen = false;
                descriptionInput?.focus();
            };
            modal.open();
            return;
        }
    
        _isSuggestOpen = false;
    }
</script>

<!--
Availability of access keys:
- A: Start
- B: Before this
- C: Created
- D: Due
- E: After this
- F: Only future dates
- G:
- H: High
- I: Highest
- J:
- K:
- L: Low
- M: Medium
- N: Normal
- O: Lowest
- P:
- Q:
- R: Recurs
- S: Scheduled
- T: Description
- U: Status
- V:
- W:
- X: Done
- Y:
- Z:
- -: Cancelled
-->

<form class="tasks-modal" on:submit|preventDefault={_onSubmit}>
    <!-- NEW_TASK_FIELD_EDIT_REQUIRED -->

    <!-- --------------------------------------------------------------------------- -->
    <!--  Description  -->
    <!-- --------------------------------------------------------------------------- -->
    <section class="tasks-modal-description-section">
        <label for="description">{@html labelContentWithAccessKey('Description', accesskey('t'))}</label>
        <!-- svelte-ignore a11y-accesskey -->
        <textarea
            bind:value={editableTask.description}
            bind:this={descriptionInput}
            id="description"
            class="tasks-modal-description"
            placeholder="Take out the trash"
            accesskey={accesskey('t')}
            on:keydown={_onDescriptionKeyDown}
            on:keyup={_openSuggestForDescriptionIfNeeded}
            on:paste={_removeLinebreaksFromDescription}
            on:drop={_removeLinebreaksFromDescription}
        />
    </section>

    <!-- --------------------------------------------------------------------------- -->
    <!--  Priority  -->
    <!-- --------------------------------------------------------------------------- -->
    {#if isShownInEditModal.priority}
        <section class="tasks-modal-priority-section">
            <PriorityEditor bind:priority={editableTask.priority} {withAccessKeys} />
        </section>
        <hr id="line-after-priority" />
    {/if}

    <!-- --------------------------------------------------------------------------- -->
    <!--  Dates  -->
    <!-- --------------------------------------------------------------------------- -->
    <section class="tasks-modal-dates-section">
        <!-- --------------------------------------------------------------------------- -->
        <!--  Recurrence  -->
        <!-- --------------------------------------------------------------------------- -->
        {#if isShownInEditModal.recurrence}
            <RecurrenceEditor {editableTask} bind:isRecurrenceValid accesskey={accesskey('r')} />
        {/if}
        <!-- --------------------------------------------------------------------------- -->
        <!--  Due Date  -->
        <!-- --------------------------------------------------------------------------- -->
        {#if isShownInEditModal.due}
            <DateEditor
                id="due"
                dateSymbol={dueDateSymbol}
                bind:date={editableTask.dueDate}
                bind:isDateValid={isDueDateValid}
                forwardOnly={editableTask.forwardOnly}
                accesskey={accesskey('d')}
            />
        {/if}

        <!-- --------------------------------------------------------------------------- -->
        <!--  Scheduled Date  -->
        <!-- --------------------------------------------------------------------------- -->
        {#if isShownInEditModal.scheduled}
            <DateEditor
                id="scheduled"
                dateSymbol={scheduledDateSymbol}
                bind:date={editableTask.scheduledDate}
                bind:isDateValid={isScheduledDateValid}
                forwardOnly={editableTask.forwardOnly}
                accesskey={accesskey('s')}
            />
        {/if}

        <!-- --------------------------------------------------------------------------- -->
        <!--  Start Date  -->
        <!-- --------------------------------------------------------------------------- -->
        {#if isShownInEditModal.start}
            <DateEditor
                id="start"
                dateSymbol={startDateSymbol}
                bind:date={editableTask.startDate}
                bind:isDateValid={isStartDateValid}
                forwardOnly={editableTask.forwardOnly}
                accesskey={accesskey('a')}
            />
        {/if}

        <!-- --------------------------------------------------------------------------- -->
        <!--  Only future dates  -->
        <!-- --------------------------------------------------------------------------- -->
        {#if isShownInEditModal.due || isShownInEditModal.scheduled || isShownInEditModal.start}
            <div class="future-dates-only" id="only-future-dates">
                <label for="forwardOnly">{@html labelContentWithAccessKey('Only future dates:', accesskey('f'))}</label>
                <!-- svelte-ignore a11y-accesskey -->
                <input
                    bind:checked={editableTask.forwardOnly}
                    id="forwardOnly"
                    type="checkbox"
                    class="task-list-item-checkbox tasks-modal-checkbox"
                    accesskey={accesskey('f')}
                />
            </div>
        {/if}
    </section>
    {#if isShownInEditModal.due || isShownInEditModal.scheduled || isShownInEditModal.start}
        <hr id="line-after-happens-dates" />
    {/if}

    <!-- --------------------------------------------------------------------------- -->
    <!--  Dependencies  -->
    <!-- --------------------------------------------------------------------------- -->
    <section class="tasks-modal-dependencies-section">
        {#if allTasks.length > 0 && mountComplete}
            <!-- --------------------------------------------------------------------------- -->
            <!--  Blocked By Tasks  -->
            <!-- --------------------------------------------------------------------------- -->
            {#if isShownInEditModal.before_this}
                <Dependency
                    id="before_this"
                    type="blockedBy"
                    labelText="Before this"
                    {task}
                    {editableTask}
                    {allTasks}
                    {_onDescriptionKeyDown}
                    accesskey={accesskey('b')}
                    placeholder="Search for tasks that the task being edited depends on..."
                />
            {/if}

            <!-- --------------------------------------------------------------------------- -->
            <!--  Blocking Tasks  -->
            <!-- --------------------------------------------------------------------------- -->
            {#if isShownInEditModal.after_this}
                <Dependency
                    id="after_this"
                    type="blocking"
                    labelText="After this"
                    {task}
                    {editableTask}
                    {allTasks}
                    {_onDescriptionKeyDown}
                    accesskey={accesskey('e')}
                    placeholder="Search for tasks that depend on this task being done..."
                />
            {/if}
        {:else}
            <div><i>Blocking and blocked by fields are disabled when vault tasks is empty</i></div>
        {/if}
    </section>
    {#if isShownInEditModal.before_this || isShownInEditModal.after_this}
        <hr id="line-after-dependencies" />
    {/if}

    <section class="tasks-modal-dates-section">
        <!-- --------------------------------------------------------------------------- -->
        <!--  Status  -->
        <!-- --------------------------------------------------------------------------- -->
        {#if isShownInEditModal.status}
            <StatusEditor {task} bind:editableTask {statusOptions} accesskey={accesskey('u')} />
        {/if}

        <!-- --------------------------------------------------------------------------- -->
        <!--  Created Date  -->
        <!-- --------------------------------------------------------------------------- -->
        {#if isShownInEditModal.created}
            <DateEditor
                id="created"
                dateSymbol={createdDateSymbol}
                bind:date={editableTask.createdDate}
                bind:isDateValid={isCreatedDateValid}
                forwardOnly={editableTask.forwardOnly}
                accesskey={accesskey('c')}
            />
        {/if}

        <!-- --------------------------------------------------------------------------- -->
        <!--  Done Date  -->
        <!-- --------------------------------------------------------------------------- -->
        {#if isShownInEditModal.done}
            <DateEditor
                id="done"
                dateSymbol={doneDateSymbol}
                bind:date={editableTask.doneDate}
                bind:isDateValid={isDoneDateValid}
                forwardOnly={editableTask.forwardOnly}
                accesskey={accesskey('x')}
            />
        {/if}

        <!-- --------------------------------------------------------------------------- -->
        <!--  Cancelled Date  -->
        <!-- --------------------------------------------------------------------------- -->
        {#if isShownInEditModal.cancelled}
            <DateEditor
                id="cancelled"
                dateSymbol={cancelledDateSymbol}
                bind:date={editableTask.cancelledDate}
                bind:isDateValid={isCancelledDateValid}
                forwardOnly={editableTask.forwardOnly}
                accesskey={accesskey('-')}
            />
        {/if}
    </section>

    <section class="tasks-modal-button-section">
        <button disabled={!formIsValid} type="submit" class="mod-cta">Apply </button>
        <button type="button" on:click={_onClose}>Cancel</button>
    </section>
</form>
