{#key clickedTask}
  {#if isDetailedCardOpen}
    <DetailedCardPopup 
      taskObject={clickedTask}
      on:card-close={() => isDetailedCardOpen = false}
      on:task-delete={(e) => deleteTaskNode(e.detail)}
      on:task-click={(e) => openDetailedCard(e.detail)}
      on:task-update={(e) => updateTaskNode(e.detail)}
      on:task-checkbox-change={(e) => updateTaskNode({ id: e.detail.id, keyValueChanges: { isDone: e.detail.isDone }})} 
    />
  {/if}
{/key}

<!-- Reason for 100dvh: https://stackoverflow.com/a/75648985/7812829 -->
<!-- style="padding: 6px; background-color: white; display: flex; align-items: center; justify-content: center;" -->
<div class:iphone-se-size={isTesting} 
     class:general-mobile-size={!isTesting}
     class:voice-active-highlight={isUsingVoice}
     style="height: 100dvh; position: relative; display: flex; flex-direction: column;"
>
  <div style="overflow-y: auto;">
    {#if activeTabName === 'TODO_VIEW'}
      <ListView
        on:task-click={(e) => openDetailedCard(e.detail)}
        on:task-checkbox-change={(e) => updateTaskNode({ id: e.detail.id, keyValueChanges: { isDone: e.detail.isDone }})}

        on:new-root-task={(e) => createTaskNode(e.detail)}
        on:subtask-create={(e) => createTaskNode(e.detail)}
        let:startTypingNewTask={startTypingNewTask}
      >
        <FloatingButtonWrapper on:click={startTypingNewTask} distanceFromBottom={100}>
          <span id="startButton" class="material-symbols-outlined" style="font-size: 48px; font-weight: 600;">
            add
          </span>
        </FloatingButtonWrapper>

        <FloatingButtonWrapper let:setBackgroundColor={setBackgroundColor}>
          <VoiceKeywordDetect
            on:voice-start={() => {
              isUsingVoice = true; 
              setBackgroundColor('orange');
            }}
            on:voice-end={() => isUsingVoice = false}
            on:new-mic-result={(e) => speechResult = e.detail}
            on:new-event-today={(e) => createNewEvent(e.detail)}
            on:new-todo={(e) => createNewTodo(e.detail)}
          />
        </FloatingButtonWrapper>
      </ListView>
    {:else if activeTabName === 'FUTURE_VIEW'}
      <ScheduleView
        on:task-duration-adjusted
        on:task-click={(e) => openDetailedCard(e.detail)}
      />
    {:else if activeTabName === 'CALENDAR_VIEW'}
      <TheMobileCalendar
        on:new-root-task={(e) => createTaskNode(e.detail)}
        on:task-click={(e) => openDetailedCard(e.detail)}
        on:task-update={(e) => updateTaskNode({
            id: e.detail.id,
            keyValueChanges: e.detail.keyValueChanges
          })
        }
      />
    {:else if activeTabName === 'AI_VIEW'}
      <AI />
    {/if}
  </div>

  <div class="bottom-navbar">
    <div on:click={() => activeTabName = 'TODO_VIEW'} class="bottom-nav-tab" class:active-nav-tab={activeTabName === 'TODO_VIEW'}>
      <div style="text-align: center;">
        <span class="material-symbols-outlined nav-tab-icon">
          summarize
        </span>
        <div class="nav-tab-desc">
          To-do
        </div>
      </div>
    </div>

    <div class="bottom-nav-tab" 
      on:click={() => {
        hasInitialScrolled.set(false)
        activeTabName = 'CALENDAR_VIEW'
      }} on:keydown
      class:active-nav-tab={activeTabName === 'CALENDAR_VIEW'}
    >
      <div style="text-align: center;">
        <span class="material-symbols-outlined nav-tab-icon">
          house
        </span>
        <div class="nav-tab-desc">
          Calendar
        </div>
      </div>
    </div>

    <div class="bottom-nav-tab" on:click={() => activeTabName = 'FUTURE_VIEW'} class:active-nav-tab={activeTabName === 'FUTURE_VIEW'}>
      <div style="text-align: center;">
        <span class=" material-icons nav-tab-icon">
          upcoming
        </span>
        <div class="nav-tab-desc">
          Events
        </div>
      </div>
    </div>

    <div class="bottom-nav-tab" on:click={() => activeTabName = 'AI_VIEW'} class:active-nav-tab={activeTabName === 'AI_VIEW'}>
      <div style="text-align: center;">
        <span class=" material-symbols-outlined nav-tab-icon">
          smart_toy
        </span>
        <div class="nav-tab-desc">
          Robot
        </div>
      </div>
    </div>
  </div>
</div>

<script>
  import TheMobileCalendar from '$lib/TheFunctionalCalendar/TheMobileCalendar.svelte'
  import AI from '$lib/AI/AI.svelte'
  import ScheduleView from '$lib/MobileMode/ScheduleView.svelte'
  import ListView from '$lib/MobileMode/ListView.svelte'
  import VoiceKeywordDetect from '$lib/VoiceKeywordDetect.svelte'
  import DetailedCardPopup from '$lib/DetailedCardPopup/DetailedCardPopup.svelte'
  import FloatingButtonWrapper from './FloatingButtonWrapper.svelte'

  import { getRandomID, getDateInMMDD } from '/src/helpers/everythingElse.js'
  import { user, todoMemoryTree, hasInitialScrolled } from '/src/store'
  import { onDestroy, onMount } from 'svelte'
  import { createTaskNode, updateTaskNode, deleteTaskNode } from '/src/helpers/crud.js'
  import { fetchMobileTodoTasks, fetchMobileCalTasks } from '$lib/MainPage/handleTasks.js'

  let isTesting = false
  let activeTabName = 'CALENDAR_VIEW' // probably the new user default, butthen persists the user's preference e.g. I prefer the to-do
  let unsub
  
  let isUsingVoice = false
  let speechResult = ''
  
  let isDetailedCardOpen = false
  let clickedTask = {}

  onMount(async () => {
    fetchMobileTodoTasks($user.uid)
    
    fetchMobileCalTasks($user.uid)

    // note, we fetch future events inside that component as a quicckfix, so
    // it'll react to changes in calendar and todo
  })

  onDestroy(() => {
    if (unsub) unsub()
  })

  function createNewEvent ({ name, startTime }) {
    const newTaskObj = {
      name,
      parentID: '',
      startTime,
      startDate: getDateInMMDD(new Date())
    }
    createTaskNode({ id: getRandomID(), newTaskObj })
  }

  // should be a function exposed by the `GrandTreeTodoReusableList` component
  function createNewTodo ({ name }) {
    const dueInHowManyDays = 7
    const d = new Date()
    d.setDate(d.getDate() + dueInHowManyDays - 1)

    const newTaskObj = {
      name,
      parentID: ''
    }

    if ($todoMemoryTree.length > 0) {
      newTaskObj.orderValue = (0 + $todoMemoryTree[0].orderValue) / 1.1
    } 
    // if it's the first task, the orderValue is initialized to `maxOrder`

    createTaskNode({ id: getRandomID(), newTaskObj })
  }

  function openDetailedCard ({ task }) {
    clickedTask = task 
    isDetailedCardOpen = true
  }
</script>

<style>
  :root {
    --bottom-navbar-height: 48px;
  }

  .voice-active-highlight {
    background-color: rgb(180, 238, 221);
  }

  .iphone-se-size {
    width: 375px; 
    height: 667px;
    border: 2px solid black;
  }

  .general-mobile-size {
    height: 100vh; 
  }

  .bottom-navbar {
    margin-top: auto;
    width: 100%; 
    height: var(--bottom-navbar-height); 
    display: flex; 
    align-items: center; 
    justify-content: space-between; 
    background-color: white;
    border-top: 1px solid lightgrey;
  }

  .bottom-nav-tab {
    display: flex; 
    align-items: center;
    justify-content: center;

    height: 100%;
    flex-basis: 0;
    flex-grow: 1;
    flex-shrink: 1;

    color: rgb(110, 110, 110);

    padding-top: 4px;
    padding-bottom: 4px;
  }

  .active-nav-tab {
    color: rgb(0, 0, 0);
    font-weight: 500;
    border-top: 0px solid rgb(0, 0, 0);
  }

  .nav-tab-desc {
    font-size: 12px;
    margin-top: -4px;
  }

  .nav-tab-icon {
    font-size: 24px;
  }
</style>