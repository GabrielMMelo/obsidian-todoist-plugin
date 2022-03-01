<script lang="ts">
  import { App, MarkdownRenderer } from "obsidian";
  import type { Moment } from "moment";
  import moment from "moment";
  import { getContext, onMount } from "svelte";
  import { fade } from "svelte/transition";
  import type { ITodoistMetadata, TodoistApi, IUpdateTaskOptions } from "../api/api";
  import type { Task, Section, Label } from "../api/models";
  import { UnknownProject, UnknownSection, UnknownLabel } from "../api/raw_models";
  import DateSelector from "../modals/createTask/DateSelector.svelte";
  import CalendarPicker from "../modals/createTask/CalendarPicker.svelte";
  import { showTaskContext } from "../contextMenu";
  import type { ISettings } from "../settings";
  import { APP_CONTEXT_KEY } from "../utils";
  import TaskList from "./TaskList.svelte";


  export let metadata: ITodoistMetadata;
  export let settings: ISettings;
  export let api: TodoistApi;
  export let sorting: string[];
  export let renderProject: boolean;
  export let onClickTask: (task: Task) => Promise<void>;
  export let onChangeTask: (task: Task, options: IUpdateTaskOptions) => Promise<boolean>;
  export let isChild: boolean;
  export let labels: Label[];

  export let todo: Task;

  const app = getContext<App>(APP_CONTEXT_KEY);

  $: isCompletable = !todo.content.startsWith("*");

  // todo: create `routine projects` option on settings e make if customizable
  $: isRoutine = ['routine-tasks'].includes(metadata.projects.get_or_default(todo?.projectID, UnknownProject)?.name);

  $: momentDate = moment(todo.rawDatetime);
  $: momentHour = moment(todo.rawDatetime).hour();
  $: momentMinute = moment(todo.rawDatetime).minute();

  let taskContentEl: HTMLDivElement;
  let drawerOpen = false;
  let offset_hours: number = -3;

  onMount(async () => {
    await renderMarkdown(todo.content);
  });


  function setDate(date: Moment) {
    //selected = date;
    console.log(date);
    let opts: IUpdateTaskOptions = {
      due_date: date.format("YYYY-MM-DD")
    };
    onChangeTask(todo, opts)
    drawerOpen = false;
  }

  function setHour(event: Event) {
    const element = event.currentTarget as HTMLInputElement
    const hour = parseInt(element.value)
    console.log(hour);
    momentDate.set({hour:hour + (-1 * offset_hours)})
    let opts: IUpdateTaskOptions = {
      due_datetime: momentDate.format("YYYY-MM-DDTHH:mm:ss") + "Z"
    };
    onChangeTask(todo, opts)
  }

  function setMinute(event: Event) {
    const element = event.currentTarget as HTMLInputElement
    const minute = parseInt(element.value)
    //selected = date;
    console.log(minute);

    momentDate.set({hour:momentDate.hour() + (-1 * offset_hours), minute:minute})
    let opts: IUpdateTaskOptions = {
      due_datetime: momentDate.format("YYYY-MM-DDTHH:mm:ss") + "Z"
    };
    onChangeTask(todo, opts)
  }

  async function renderMarkdown(content: string): Promise<void> {
    // Escape leading '#' or '-' so they aren't rendered as headers/bullets.
    if (content.startsWith("#") || content.startsWith("-")) {
      content = `\\${content}`;
    }

    // A task starting with '*' signifies that it cannot be completed, so we should strip it from the front of the task.
    if (content.startsWith("*")) {
      content = content.substr(1);
    }

    await MarkdownRenderer.renderMarkdown(content, taskContentEl, "", null);

    // Remove the wrapping '<p>' tag to force it to inline.
    const markdownContent = taskContentEl.querySelector("p");

    if (markdownContent) {
      markdownContent.parentElement.removeChild(markdownContent);
      taskContentEl.innerHTML += markdownContent.innerHTML;
    }
  }

  // For some reason, the Todoist API returns priority in reverse order from
  // the p1/p2/p3/p4 fluent entry notation.
  function getPriorityClass(priority: number): string {
    switch (priority) {
      case 1:
        return "todoist-p4";
      case 2:
        return "todoist-p3";
      case 3:
        return "todoist-p2";
      case 4:
        return "todoist-p1";
    }
  }

  function onClickTaskContainer(evt: MouseEvent) {
    evt.stopPropagation();
    evt.preventDefault();

    showTaskContext(
      app,
      {
        task: todo,
        onClickTask: onClickTask,
      },
      {
        x: evt.pageX,
        y: evt.pageY,
      }
    );
  }

  function onChangeTaskContainer(evt) {
    evt.stopPropagation();
    evt.preventDefault();
    console.log("evt", evt.target.value);
    
    let opts: IUpdateTaskOptions = {
      label_ids: [parseInt(evt.target.value)]
    };
    onChangeTask(todo, opts)
  }
</script>

<li
  on:contextmenu={onClickTaskContainer}
  transition:fade={{ duration: settings.fadeToggle ? 400 : 0 }}
  class="task-list-item {todo.isOverdue() ? 'task-overdue' : ''}
          {todo.hasTime ? 'has-time' : 'has-no-time'}">
  <div class={getPriorityClass(todo.priority)}>
    {#if isRoutine}
    <input
      disabled={!isCompletable}
      data-line="1"
      class="task-list-item-checkbox"
      type="checkbox"
      on:click|preventDefault={async () => {
        await onClickTask(todo);
      }} />
    {/if}
    <a aria-label-position="top" aria-label={todo.content} data-href={todo.content} href={todo.content} class="internal-link" target="_blank" rel="noopener">{todo.content}</a> <span> <a class="link-to-todoist" href="{todo.url}">↳</a></span>
  </div>
  <div class="task-metadata">
    {#if settings.renderProject && renderProject}
      <div class="task-project">
        {#if settings.renderProjectIcon}
          <svg
            class="task-project-icon"
            xmlns="http://www.w3.org/2000/svg"
            viewBox="0 0 20 20"
            fill="currentColor">
            <path
              fill-rule="evenodd"
              d="M5 3a2 2 0 00-2 2v10a2 2 0 002 2h10a2 2 0 002-2V5a2 2 0 00-2-2H5zm0 2h10v7h-2l-1 2H8l-1-2H5V5z"
              clip-rule="evenodd" />
          </svg>
        {/if}
        {metadata.projects.get_or_default(todo.projectID, UnknownProject).name}
        {#if !isChild && !isRoutine}
        <span>
          |
          {#if settings.renderLabelsIcon}
            <svg
              class="task-labels-icon"
              xmlns="http://www.w3.org/2000/svg"
              viewBox="0 0 20 20"
              fill="currentColor">
              <path
                fill-rule="evenodd"
                d="M17.707 9.293a1 1 0 010 1.414l-7 7a1 1 0 01-1.414 0l-7-7A.997.997 0 012 10V5a3 3 0 013-3h5c.256 0 .512.098.707.293l7 7zM5 6a1 1 0 100-2 1 1 0 000 2z"
                clip-rule="evenodd" />
            </svg>
          {/if}
            <!--
            <select class="todoist-task-section" on:change={onChangeTaskContainer}>
              {#each Array(sections?.filter((section) => section.projectID == todo.projectID)?.length) as _, i}
              <option value="{sections?.filter((section) => section.projectID == todo.projectID)?.[i]?.sectionID}" selected={true ? metadata.sections.get_or_default(todo.sectionID, UnknownSection).id == sections?.filter((section) => section.projectID == todo.projectID)?.[i]?.sectionID : false}>{sections?.filter((section) => section.projectID == todo.projectID)?.[i]?.name}</option>
              {/each}
            </select>
        -->
          <select class="todoist-task-label-select" on:change={onChangeTaskContainer}>
            <option class="todoist-task-option" value="-1" selected={todo.labelIDs?.length == 0 ? true : false}> </option>
              {#each Array(labels?.length) as _, i}
              <option class="todoist-task-option" value="{labels?.[i]?.labelID}" selected={todo.labelIDs?.[0] == labels?.[i]?.labelID ? true : false}>{labels?.[i]?.name}</option>
              {/each}
            </select>
        </span>
        {/if}
      </div>
    {/if}
    {#if settings.renderDate && todo.date}
      <div class="task-date {todo.isOverdue() ? 'task-overdue' : ''}"
        >
        {#if settings.renderDateIcon}
          <svg
            class="task-calendar-icon"
            xmlns="http://www.w3.org/2000/svg"
            viewBox="0 0 20 20"
            fill="currentColor">
            <path
              fill-rule="evenodd"
              d="M6 2a1 1 0 00-1 1v1H4a2 2 0 00-2 2v10a2 2 0 002 2h12a2 2 0 002-2V6a2 2 0 00-2-2h-1V3a1 1 0 10-2 0v1H7V3a1 1 0 00-1-1zm0 5a1 1 0 000 2h8a1 1 0 100-2H6z"
              clip-rule="evenodd" />
          </svg>
        {/if}
        <span
          on:click={(ev) => {
            if (!drawerOpen) {
              ev.stopPropagation();
              drawerOpen = true;
            }
            else {
              ev.stopPropagation();
              drawerOpen = false;
            }
          }}>
          {momentDate.format('YYYY-MM-DD')}
        </span>
        <span style="margin-left: 10px;">
          <select class="todoist-task-time-select todoist-task-hour-select" on:change={(e) => setHour(e)}>
            {#each ([...Array(24).keys()].map(x => x++)) as hour}
              <option class="todoist-task-option" selected={momentHour == hour} value={hour}>{hour > 9 ? hour : '0' + hour}</option>
            {/each}
          </select>
          :
          <select class="todoist-task-time-select todoist-task-minute-select" on:change={(e) => setMinute(e)}>
            {#each ([...Array(12).keys()].map(x => x*5)) as minute}
              <option class="todoist-task-option" selected={momentMinute == minute} value={minute}>{minute > 9 ? minute : '0' + minute}</option>
            {/each}
          </select>
        </span>
        {#if drawerOpen}
        <div class="calendar-container">
          <CalendarPicker bind:selected={momentDate} on:selectDate={(ev) => setDate(ev.detail)} />
        </div>
        {/if}
      </div>
    {/if}
    <!--
    {#if settings.renderLabels && todo.labelIDs.length > 0}
      <div class="task-labels">
        {#if settings.renderLabelsIcon}
          <svg
            class="task-labels-icon"
            xmlns="http://www.w3.org/2000/svg"
            viewBox="0 0 20 20"
            fill="currentColor">
            <path
              fill-rule="evenodd"
              d="M17.707 9.293a1 1 0 010 1.414l-7 7a1 1 0 01-1.414 0l-7-7A.997.997 0 012 10V5a3 3 0 013-3h5c.256 0 .512.098.707.293l7 7zM5 6a1 1 0 100-2 1 1 0 000 2z"
              clip-rule="evenodd" />
          </svg>
        {/if}
        {#each todo.labelIDs as labelID, i}
          {metadata.labels.get_or_default(labelID, 'UnknownLabel')}{#if i != todo.labelIDs.length - 1}
            ,
          {/if}
        {/each}
      </div>
    {/if}
  -->
  </div>
  {#if todo.children.length != 0}
    <TaskList
      tasks={todo.children}
      {settings}
      {api}
      {sorting}
      {renderProject}
      isChild={true}/>
  {/if}
</li>
