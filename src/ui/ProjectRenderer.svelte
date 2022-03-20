<script lang="ts">
  import { App, MarkdownRenderer } from "obsidian";
  import { getAPI } from "obsidian-dataview"
  import { SVGGantt, CanvasGantt, StrGantt } from 'gantt';
  import { getContext, onMount, afterUpdate } from "svelte";
  import { fade } from "svelte/transition";
  import type { ITodoistMetadata, TodoistApi } from "../api/api";
  import type { Task, Project } from "../api/models";
  import { Label } from "../api/models";
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
  let ganttEl: HTMLDivElement;
  let tasksDone: number;
  let tasksCount: number;
  let projectProgress: number;
  let labels;

  onMount(async () => {
    await renderMarkdown(project.name);
    //project.sections.filter((section) => section.name == "Done")[0].count();
    tasksCount = project.count();
    projectProgress = tasksDone/tasksCount*1000;

  });

  afterUpdate(() => {
    const dvApi = getAPI();
    console.log("dailyNote:", dvApi?.page('2022-03-20.md'))
    const data = project.tasks.map((task) => {
      return {
        id: task.id,
        text: task.content,
        start: task.rawDatetime.toDate(),
        end: task.rawDatetime.add(task.priority, 'days').toDate(),
        percent: 0.2  // implement this
      }
    })
    ganttify(data);
  })
  
  function ganttify(data) {
    const strGantt = new StrGantt(data, {
      viewMode: 'day',
      styleOptions: {
        //bgColor: '#222',
        //textColor: '#ccc'
        fontSize: '16px',
        smallFontSize: '14px',
      },
      rowHeight: 60,
      barHeight: 20,
      thickWidth: 1.8,
      maxTextWidth: 400,
    });

    //new SVGGantt('#svg-root', data, {
    //  viewMode: 'week'
    //});
    const rendered = strGantt.render();
    ganttEl.innerHTML = rendered
    //return rendered;
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

  function getDoneCount(project): number {
    //return (element.name === "Done")
    api.metadata.subscribe((value) => (labels = Label.buildTree(value.labels)));
    let doneLabels = ["Done", "Canceled"]
    let doneLabelsIDs = labels.filter((label) => doneLabels?.includes(label?.name)).map((label) => label?.labelID)
    console.log("doneLabelsID:", doneLabelsIDs)
    tasksCount = project.countByLabels(doneLabelsIDs)
    console.log("tasksDone:", tasksCount)
    return tasksCount
  }

  function getSprintCount(project): number {
    //return (element.name === "Done")
    api.metadata.subscribe((value) => (labels = Label.buildTree(value.labels)));
    let notSprintLabels = ["Backlog"]
    let notSprintLabelsIDs = labels.filter((label) => !notSprintLabels?.includes(label?.name)).map((label) => label?.labelID)
    console.log("doneLabelsID:", notSprintLabelsIDs)
    tasksCount = project.countByLabels(notSprintLabelsIDs)
    console.log("tasksDone:", tasksCount)
    return tasksCount
  }
          
 

</script>

<li
  transition:fade={{ duration: settings.fadeToggle ? 400 : 0 }}
  class="task-list-item">
  <div>
    <a aria-label-position="top" aria-label={project.name + " - Project"} data-href={project.name + " - Project"} href={project.name + " - Project"} class="internal-link" target="_blank" rel="noopener">{project.name} </a>
  </div>
  <div class="task-metadata">
   {#if project.inboxProject}
   <p>Count: {project.count()}</p>
   {:else} 
    <img class="progress-badge" src={"https://progress-bar.dev/" + ((getDoneCount(project)||0)/getSprintCount(project)*100).toFixed()  + "/?scale=" + "100" + "&title=" + project.name + "&width=400)"}/>
   {/if}
  </div>
  <div id='svg-root' bind:this={ganttEl}>
  </div>
</li>
