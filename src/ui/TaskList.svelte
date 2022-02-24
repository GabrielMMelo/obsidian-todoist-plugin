<script lang="ts">
  import { onDestroy } from "svelte";
  import type { TodoistApi, ITodoistMetadata, ICreateTaskOptions, } from "../api/api";
  import type { Task, ID, Section } from "../api/models";
  import type { ISettings } from "../settings";
  import NoTaskDisplay from "./NoTaskDisplay.svelte";
  import TaskRenderer from "./TaskRenderer.svelte";

  export let tasks: Task[];
  export let allSections: Section[];
  export let settings: ISettings;
  export let api: TodoistApi;
  export let sorting: string[];
  export let renderProject: boolean = true;
  export let renderNoTaskInfo: boolean = true;

  let metadata: ITodoistMetadata = null;
  const metadataUnsub = api.metadata.subscribe((value) => (metadata = value));

  onDestroy(() => {
    metadataUnsub();
  });

  let tasksPendingClose: ID[] = [];
  $: todos = tasks
    .filter((task) => !tasksPendingClose.includes(task.id))
    .sort((first: Task, second: Task) => first.compareTo(second, sorting));

  $: sections = allSections
    .sort((first: Section, second: Section) => first.order - second.order)

  async function onClickTask(task: Task) {
    tasksPendingClose.push(task.id);
    tasksPendingClose = tasksPendingClose;

    if (await api.closeTask(task.id)) {
      tasks.filter((otherTask) => otherTask.id == task.id);
      tasks = tasks;
    }

    tasksPendingClose.filter((id) => id == task.id);
    tasksPendingClose = tasksPendingClose;
  }

  async function onChangeTask(task: Task, options: ICreateTaskOptions): Promise<boolean> {
    var changed =  await api.updateTask(task.id, options);
    return changed;
  }

</script>

{#if todos.length != 0 && sections.length != 0}
  <ul class="contains-task-list todoist-task-list">
    {#each todos as todo (todo.id)}
      <TaskRenderer
        onClickTask={onClickTask}
        onChangeTask={onChangeTask}
        metadata={metadata}
        settings={settings}
        api={api}
        sorting={sorting}
        renderProject={renderProject}
        allSections={sections}
        todo={todo} />
    {/each}
  </ul>
{:else if renderNoTaskInfo}
  <NoTaskDisplay />
{/if}
