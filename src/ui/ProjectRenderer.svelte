<script lang="ts">
  import { App, MarkdownRenderer } from "obsidian";
  import { getContext, onMount } from "svelte";
  import { fade } from "svelte/transition";
  import type { ITodoistMetadata, TodoistApi } from "../api/api";
  import type { Task, Project } from "../api/models";
  import { UnknownProject, UnknownSection } from "../api/raw_models";
  import { showTaskContext } from "../contextMenu";
  import type { ISettings } from "../settings";
  import { APP_CONTEXT_KEY } from "../utils";
  import TaskList from "./TaskList.svelte";
  import ProjectList from "./ProjectList.svelte";

  export let metadata: ITodoistMetadata;
  export let settings: ISettings;
  export let api: TodoistApi;

  export let todo: Task;
  export let project: Project;

  const app = getContext<App>(APP_CONTEXT_KEY);

  let taskContentEl: HTMLDivElement;
  let tasksDone: number;
  let tasksCount: number;
  let projectProgress: number;

  onMount(async () => {
    await renderMarkdown(project.name);
    tasksDone = project.sections.filter((section) => section.name == "Done")[0].count();
    tasksCount = project.count();
    projectProgress = tasksDone/tasksCount*1000;

  });
    
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

  function getDone(element, index, array) {
    return (element.name === "Done")
  }
          
</script>

<li
  transition:fade={{ duration: settings.fadeToggle ? 400 : 0 }}
  class="task-list-item">
  <div>
    <a aria-label-position="top" aria-label={project.name + "-" + project.projectID} data-href={project.name + "-" + project.projectID} href={project.name + "-" + project.projectID} class="internal-link" target="_blank" rel="noopener">{project.name} </a>
  </div>
  <div class="task-metadata">
    <img src={"https://progress-bar.dev/" + (project.sections.filter(getDone)[0]?.count()||0)/project.count()*100  + "/?scale=" + "100" + "&title=" + project.name + "&width=400)"}/>
  </div>
</li>
