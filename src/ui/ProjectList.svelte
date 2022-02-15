<script lang="ts">
  import { onDestroy } from "svelte";
  import type { TodoistApi, ITodoistMetadata } from "../api/api";
  import type { Project, Section, Task, ID } from "../api/models";
  import type { ISettings } from "../settings";
  import NoProjectDisplay from "./NoProjectDisplay.svelte";
  import ProjectRenderer from "./ProjectRenderer.svelte";

  export let projects: Project[];
  export let settings: ISettings;
  export let api: TodoistApi;

  let metadata: ITodoistMetadata = null;
  const metadataUnsub = api.metadata.subscribe((value) => (metadata = value));

  onDestroy(() => {
    metadataUnsub();
  });

  /*
  let projectsPendingClose: ID[] = [];
  $: todos = tasks
    .filter((task) => !tasksPendingClose.includes(task.id))
    .sort((first: Task, second: Task) => first.compareTo(second, sorting));
  */

</script>

{#if projects.length != 0}
  <ul class="contains-task-list todoist-task-list">
    {#each projects as project (project.projectID)}
      <ProjectRenderer
        {metadata}
        {settings}
        {api}
        {project} />
    {/each}
  </ul>
{:else}
  <NoProjectDisplay />
{/if}
