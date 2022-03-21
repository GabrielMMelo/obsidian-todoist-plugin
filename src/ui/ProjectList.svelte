<script lang="ts">
  import { onDestroy, afterUpdate } from "svelte";
  import { getAPI } from "obsidian-dataview";
  import { SVGGantt, CanvasGantt, StrGantt } from 'gantt';
  import type { TodoistApi, ITodoistMetadata } from "../api/api";
  import type { Project, Section, Task, ID } from "../api/models";
  import { Label } from "../api/models";
  import type { ISettings } from "../settings";
  import NoProjectDisplay from "./NoProjectDisplay.svelte";
  import ProjectRenderer from "./ProjectRenderer.svelte";

  export let projects: Project[];
  export let settings: ISettings;
  export let api: TodoistApi;
  export let showGantt: boolean;

  let showSubtasks: boolean = false;
  let metadata: ITodoistMetadata = null;
  const metadataUnsub = api.metadata.subscribe((value) => (metadata = value));
  let ganttEl: HTMLDivElement;
  let labels;

  onDestroy(() => {
    metadataUnsub();
  });

  $: activeProjects = projects
    .filter((project) => !["Routine"].includes(project.name))
    .sort((first: Project, second: Project) => (first.order||0) - (second.order||0));
  /*
  let projectsPendingClose: ID[] = [];
  $: todos = tasks
    .filter((task) => !tasksPendingClose.includes(task.id))
    .sort((first: Task, second: Task) => first.compareTo(second, sorting));
  */

  afterUpdate(() => {
    const dvApi = getAPI();

    if(!showGantt) return;
    api.metadata.subscribe((value) => (labels = Label.buildTree(value.labels)));
    let doneLabels = ["Done", "Canceled"]
    let doneLabelsIDs = labels.filter((label) => doneLabels?.includes(label?.name)).map((label) => label?.labelID)
    let startedLabels = ["Started", "Doing", "Blocked"]
    let startedLabelsIDs = labels.filter((label) => startedLabels?.includes(label?.name)).map((label) => label?.labelID)
    let validatingLabels = ["Validating"]
    let validatingLabelsIDs = labels.filter((label) => validatingLabels?.includes(label?.name)).map((label) => label?.labelID)

    const projectsTasks = projects.map((project) => {
      let tasks = project.tasks.map((task) => {
        let subtasks = [];
        //console.log("Task", task.content);
        //console.log("Page", dvApi?.page(task.content + '.md')?.file?.tasks?.values);
        if (showSubtasks && dvApi?.page(task.content + '.md')?.file?.tasks?.values?.length > 0) {
          //console.log("Page found!")
          subtasks = dvApi?.page(task.content + '.md').file.tasks.values.map((subtask, idx) => {
            //console.log("Subtask", subtask);
            return {
              id: parseInt(task.id.toString() + idx),
              parent: task.id,
              text: "      " + subtask.text + (subtask.completed ? ' ✅' : ''),
              //start: task.rawDatetime.toDate(),
              //end: task.rawDatetime.add(60, 'minutes').toDate(),
              //percent: subtask.completed,
              /*
              link: [{
                target: task.id,
                type: 'SF'
              }]
              */
            }
          }) 
        }

        subtasks = subtasks.reverse();
        //console.log("substasks", subtasks);
        subtasks.push({
          id: task.id,
          text: task.content + " (" + project.name + ")",
          start: task.rawDatetime?.toDate(),
          end: task.rawDatetime?.add(task.priority, 'days').toDate(),
          percent: doneLabelsIDs?.includes(task.labelIDs?.[0]) ? 1 : (startedLabelsIDs?.includes(task.labelIDs?.[0]) ? 0.2 : (validatingLabelsIDs?.includes(task.labelIDs?.[0]) ? 0.8 : 0))  // implement this
        })
        //console.log("substasks2", subtasks);

        return subtasks.reverse()//[].concat(...tasksCombined)
      })
      return [].concat(...tasks)
    })
    // console.log("projectsTasks", [].concat(...projectsTasks))
    let allTasks = [].concat(...projectsTasks)
    allTasks = allTasks.sort((a,b) => (a.start > b.start) ? 1 : ((b.start > a.start) ? -1 : 0))
    ganttify(allTasks);
  })

  function ganttify(data) {
    const strGantt = new StrGantt(data, {
      viewMode: 'day',
      styleOptions: {
        taskBack: '#b8ccba',
      },
      /*styleOptions: {
        bgColor: '#262626',           // default: '#fff'
        lineColor: '#333',         // default: '#eee'
        redLineColor: '#f04134',      // default: '#f04134'
        groupBack: '#3db9d3',         // default: '#3db9d3'
        groupFront: '#299cb4',        // default: '#299cb4'
        taskBack: '#65c16f',          // default: '#65c16f'
        taskFront: '#46ad51',         // default: '#46ad51'
        milestone: '#d33daf',         // default: '#d33daf'
        warning: '#faad14',           // default: '#faad14'
        danger: '#f5222d',            // default: '#f5222d'
        link: '#ffa011',              // default: '#ffa011'
        textColor: '#BDBDBD',         // default: '#222'
        lightTextColor: '#666',    // default: '#999'
        lineWidth: '1px',         // default: '1px'
        thickLineWidth: '1.4px',    // default: '1.4px'
        fontSize: '16px',          // default: '14px'
        smallFontSize: '14px',     // default: '12px'
        fontFamily: "'-apple-system, BlinkMacSystemFont, Segoe UI, Roboto,",        // default: '-apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, "Helvetica Neue", Arial, sans-serif'
      },*/
      rowHeight: 60,
      barHeight: 20,
      thickWidth: 1.8,
      maxTextWidth: 500,
      showLinks: true,
    });

    //new SVGGantt('#svg-root', data, {
    //  viewMode: 'week'
    //});
    const rendered = strGantt.render();
    ganttEl.innerHTML = rendered
    //return rendered;
  }
</script>

<div id='svg-root' bind:this={ganttEl}>
</div>
{#if activeProjects.length != 0}
  <ul class="contains-task-list todoist-task-list">
    
    {#each activeProjects as project (project.projectID)}
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
