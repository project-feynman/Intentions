<script>
  import ReusableCreateTaskDirectly from '$lib/ReusableCreateTaskDirectly.svelte'
  import ReusableFlexibleDayTask from '$lib/ReusableFlexibleDayTask.svelte'
  import FunctionalDoodleIcon from '$lib/FunctionalDoodleIcon.svelte'
  import { createEventDispatcher } from 'svelte'
  import {
    tasksScheduledOn,
    whatIsBeingDraggedFullObj,
    whatIsBeingDragged,
    whatIsBeingDraggedID
  } from '/src/store'
  import { DateTime } from 'luxon'

  export let ISODate
  export let isShowingDockingArea
  export let isCompact = false

  const dispatch = createEventDispatcher()
  let isDirectlyCreatingTask = false

  function dragover_handler(e) {
    e.preventDefault()
    e.stopPropagation()
    e.dataTransfer.dropEffect = 'move'
  }

  function drop_handler(e, ISODate) {
    const id = e.dataTransfer.getData('text/plain')
    if (!id) return // it means we're adjusting the duration but it triggers a drop event, and a dragend event must be followed by a drop event

    e.preventDefault()
    e.stopPropagation()

    const dt = DateTime.fromISO(ISODate)

    dispatch('task-update', {
      id,
      keyValueChanges: {
        startTime: '',
        startDateISO: dt.toFormat('yyyy-MM-dd')
      }
    })

    whatIsBeingDraggedFullObj.set(null)
    whatIsBeingDraggedID.set('')
    whatIsBeingDragged.set('')
  }
</script>

<div
  class="day-header sticky-day-of-week-abbreviation"
  style:padding={isCompact ? '0px 0px 0px 0px' : 'var(--main-content-top-margin) 0px 18px 0px'}
  on:click|self={() => (isDirectlyCreatingTask = true)} on:keydown
  on:dragover={(e) => dragover_handler(e)}
  on:drop={(e) => drop_handler(e, ISODate)}
>
  <div class="unselectable" class:compact-horizontal={isCompact} style:display={isCompact ? 'flex' : 'block'}>
    <div
      class="center-flex day-name-label"
      class:active-day-name={ISODate <= DateTime.now().toFormat('yyyy-MM-dd')}
    >
      {DateTime.fromISO(ISODate).toFormat('ccc')}
    </div>

    <div class="center-flex" style="font-size: 16px; font-weight: 300">
      <div
        class="center-flex"
        style="padding: 8px 0px; width: 28px; height: 36px;"
        class:active-date-number={ISODate <= DateTime.now().toFormat('yyyy-MM-dd')}
        class:highlighted-circle={false}
      >
        {DateTime.fromISO(ISODate).toFormat('dd')}
      </div>
    </div>
  </div>

  {#if isShowingDockingArea}
    <div style="overflow: hidden; margin-top: {isCompact ? '0' : '4'}px;">
      {#if $tasksScheduledOn}
        {#if $tasksScheduledOn[ISODate]}
          <div style="display: flex; flex-wrap: wrap;">
            {#each $tasksScheduledOn[ISODate].noStartTime.hasIcon as iconTask (iconTask.id)}
              <FunctionalDoodleIcon {iconTask} on:task-click on:task-update />
            {/each}
          </div>

          <div style="display: flex; flex-direction: column;">
            {#each $tasksScheduledOn[ISODate].noStartTime.noIcon as flexibleDayTask (flexibleDayTask.id)}
              <div
                on:click={() => dispatch('task-click', { task: flexibleDayTask })}
                style="width: var(--calendar-day-section-width); 
                font-size: 12px; 
                display: flex; gap: 4px; 
                margin-top: 0px; margin-left: 4px; margin-right: 4px; margin-bottom: {isCompact ? '4px' : '8px'};
              "
              >
                <ReusableFlexibleDayTask
                  task={flexibleDayTask}
                  on:task-click
                  on:task-update
                />
              </div>
            {/each}
          </div>
        {/if}
      {/if}
    </div>
  {/if}
  <!-- END OF DOCKING AREA -->

  {#if isDirectlyCreatingTask}
    <div
      id="calendar-direct-task-div"
      style="
        width: 90%; 
        padding-left: 0px; 
        padding-right: 0px;
      "
    >
      <ReusableCreateTaskDirectly
        newTaskStartTime={''}
        resultantDateClassObject={DateTime.fromISO(ISODate).toJSDate()}
        on:new-root-task
        on:reset={() => (isDirectlyCreatingTask = false)}
      />
    </div>
  {/if}
</div>

<style>
  .compact-horizontal {
    display: flex; 
    justify-content: center;
  }

  .day-header {
    width: var(--calendar-day-section-width);
    padding-top: var(--main-content-top-margin);
    padding-bottom: 18px;
  }

  .sticky-day-of-week-abbreviation {
    font-size: 1.4em;
    background-color: var(--calendar-bg-color);
    color: #6d6d6d;

    position: sticky;

    /* FIGURE THIS OUT TOMORROW */
    top: 0px;
    z-index: 1;
  }

  .day-name-label {
    font-size: 16px;
    margin-bottom: 0px;
    font-weight: 400;
  }

  .active-day-name {
    color: rgb(30, 30, 30);
  }

  .active-date-number {
    font-weight: 300;
    color: rgb(60, 60, 60);
  }

  .highlighted-circle {
    border-radius: 25px;
  }

  .center-flex {
    display: flex;
    justify-content: center;
    align-items: center;
  }
</style>
