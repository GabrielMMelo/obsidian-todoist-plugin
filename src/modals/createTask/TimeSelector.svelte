<script lang="ts">
  import type { Moment } from "moment";
  import moment from "moment";
  import CalendarPicker from "./CalendarPicker.svelte";

  export let selected: Moment = null;

  let container: HTMLDivElement = null;
  let target: HTMLDivElement = null;
  let drawerOpen = false;

  function setTime(event: Event) {
    const element = event.currentTarget as HTMLInputElement
    const value = element.value
    selected = moment(value, "HH:mm");
  }
</script>

<style>
  .date-input {
    border: 1px solid var(--background-primary-alt);
    background-color: var(--background-modifier-form-field);
    line-height: 42px;
  }

  .date-input:hover {
    border: 1px solid var(--interactive-accent);
    background-color: var(--background-modifier-form-highlighted);
  }

  .date-preview {
    margin-left: 1em;
  }

  .reset-date-button {
    float: right;
    height: 42px;
    width: 20px;
    margin-right: 10px;
  }

  .date-placeholder {
    margin-left: 1em;
    color: var(--text-muted);
  }

  .drawer-target {
    position: fixed;
    z-index: 2;
    height: auto;
    background-color: var(--background-primary);
  }

  .specific-date-option {
    height: 42px;
    line-height: 42px;
    padding: 0 20px;
  }

  .specific-date-option:hover {
    background-color: var(--background-secondary);
  }

  .calendar-container {
    padding: 20px;
    border-top: 2px solid var(--background-modifier-border);
  }
</style>

<div class="date-input">
  <input on:change={(e) => {setTime(e)}} id="appt-time" type="time" name="appt-time"
         min="00:00" max="23:00">
</div>
