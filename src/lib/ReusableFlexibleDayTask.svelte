<!-- 
  mika blue color: '#0085FF'
  experimental green: '#509c13'
  Note, the HTML checkbox tick color literally cannot be changed, but it will be automatically white if it "decides" that our chosen background color is dark enough, 
  or vice versa
 -->
<div 
  on:click={() => dispatch('task-click', { task })}
  draggable="true" 
  on:dragstart|self={(e) => startDragMove(e, task.id)} 
  class:calendar-block={!isBulletPoint}
  style="
    position: relative;
    height: fit-content; 
    min-height: 12px;
    opacity: {task.isDone ? '1' : '0.7'};
    padding-left: {isBulletPoint ? '0px' : 'var(--left-padding)'};
    padding-right: var(--left-padding);
    display: flex; flex-direction: column;
  " 
  on:keydown={() => {}}
>
 <!-- As long as this parent div is correctly sized, the duration adjusting area 
   will be positioned correctly (it's glued to the bottom of this parent div)

   `min-height` prevents the parent from being super small when it's bullet point mode
 -->
 <div style="display: flex; align-items: center; width: {cssWidth};">
    <div>
      <!-- `checked` hydrates the initial value 
        `task-checkbox-change` event will just toggle the checkbox, 
        but in case we ever need the new value, it's `e.target.checked`
      -->
      <ReusableCheckbox
        value={task.isDone}
        on:change={(e) => dispatch('task-update', {
          id: task.id,
          keyValueChanges: {
            isDone: e.target.checked
          }
        }
        )}
      />
    </div>

   <div class="task-name truncate-to-one-line" style="font-size: {fontSizeInPx}px; margin-left: 4px;">
     {task.name}
   </div>
 </div>
 <!-- End of task name flexbox -->
</div>

<script>
 // Assumes `task` is hydrated
 import { createEventDispatcher } from 'svelte'
 import { yPosWithinBlock, whatIsBeingDragged, whatIsBeingDraggedID, whatIsBeingDraggedFullObj } from '/src/store'
 import ReusableCheckbox from '$lib/ReusableCheckbox.svelte'

 export let task = null
 export let pixelsPerHour = null
 export let fontSizeInPx = 12
 export let cssWidth = 'var(--calendar-day-section-width)'

 $: height = (pixelsPerHour / 60) * task.duration
 $: isBulletPoint = height < 20

 const dispatch = createEventDispatcher()

 function startDragMove (e, id) {
   e.dataTransfer.setData("text/plain", id)

   // record distance from the top of the element
   const rect = e.target.getBoundingClientRect()
   const y = e.clientY - rect.top // y position within el ement

   whatIsBeingDraggedID.set(id)
   whatIsBeingDragged.set('room')
   whatIsBeingDraggedFullObj.set(task)

   yPosWithinBlock.set(y)
 }
</script> 

<style>
 :root {
   --left-padding: 6px;
 }

 .calendar-block {
   width: 100%;
   padding-top: var(--left-padding);
   cursor: pointer;
   border-radius: var(--left-padding);
 }

 .task-name {
   margin-top: -1px; 
   margin-left: 4px; 
   width: 100%;
   cursor: pointer; 
 }
</style>