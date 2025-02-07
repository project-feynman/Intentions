<div
  bind:this={ReorderDropzone} 
  style="height: {heightInPx}px; border-radius: {heightInPx / 2}px; border: 0px solid {colorForDebugging};" 
  on:dragenter={() => {
    // quickfix as even if it's an invalid operation it's unintuitive to not see the drag area highlight
    if (!isInvalidReorderDrop() || true) {
      ReorderDropzone.style.background = 'rgb(87, 172, 247)' 
    }
  }}
  on:dragleave={() => ReorderDropzone.style.background = '' }
  on:dragover={(e) => dragover_handler(e)}
  on:drop|stopPropagation={(e) => onReorderDrop(e)}
>

</div>

<script>
  import { updateLocalState } from '/src/helpers/maintainState.js'
  import {
    maintainValidSubtreeDeadlines,
    correctDeadlineIfNecessary
  } from '/src/helpers/subtreeDragDrop.js'
  import { 
    whatIsBeingDragged, 
    whatIsBeingDraggedID, 
    whatIsBeingDraggedFullObj,
    user 
  } from '/src/store'
  import { increment, writeBatch, doc } from 'firebase/firestore'
  import {db} from '../back-end/firestoreConnection'

  export let ancestorRoomIDs
  export let roomsInThisLevel
  export let idxInThisLevel
  export let parentID = ''
  export let parentObj
  export let colorForDebugging = "red"
  export let dueInHowManyDays // tells us which todo-list we're in
  export let isMilestoneMode = false
  export let heightInPx = 18

  let ReorderDropzone
  let batch = writeBatch(db)

  $: n = roomsInThisLevel.length

  function isInvalidReorderDrop () {
    return !['room', 'top-level-task-within-this-todo-list'].includes($whatIsBeingDragged) || ancestorRoomIDs.includes($whatIsBeingDraggedID)
  }

  function dragover_handler (e) {
    e.preventDefault()
    e.stopPropagation()
  }

  async function onReorderDrop (e) {
    e.preventDefault()
    e.stopPropagation()
    if (isInvalidReorderDrop()) {
      alert('A parent task cannot become its own descendant')
      return
    }
    ReorderDropzone.style.background = ''

    batch = writeBatch(db)

    const data = e.dataTransfer.getData('text/plain')
    const keyValuePairs = data.split(']')

    const [key1, value1] = keyValuePairs[0].split(':')
    const draggedRoomID = value1

    // before updating `orderValue`, we update
    // the counter showing how many subfolders a folder has
    const droppedRoomDoc = roomsInThisLevel[idxInThisLevel]

    const initialNumericalDifference = 3
    let newVal 


    // TO-DO: need the last drop zone to be manually added
    const dropZoneIdx = idxInThisLevel
    // copy `PopupRearrangeVideos` 
    if (dropZoneIdx === 0) {
      const topOfOrderDoc = roomsInThisLevel[0]
      if (topOfOrderDoc) {
        // halve the value so it never gets to 0 
        newVal = (topOfOrderDoc.orderValue || 3) / 1.1
      } else {
        // you're dragging a new subtask into a parent that previously had ZERO children, which is valid
        newVal = 3
      }
    }
    else if (dropZoneIdx === n) {
      const bottomOfOrderDoc = roomsInThisLevel[n-1]
      newVal = (bottomOfOrderDoc.orderValue || 0) + initialNumericalDifference
      
      // keep track fo the highest possible maxOrdervalue for this $user
      if (!$user.maxOrderValue || $user.maxOrderValue < newVal) {
        batch.update(
          doc(db, `/users/${$user.uid}/`), {
            maxOrderValue: increment(initialNumericalDifference)
          }
        )
      }

      newVal = Math.max(newVal, $user.maxOrderValue)
    }
    else {
      let topNeighborDoc = roomsInThisLevel[dropZoneIdx - 1]
      let botNeighborDoc = roomsInThisLevel[dropZoneIdx]
      const order1 = botNeighborDoc.orderValue || 3
      const order2 = topNeighborDoc.orderValue || 3 + initialNumericalDifference
      newVal = (order1 + order2) / 2
    }

    // First, we ensure the dragged-dropped task is updated correctly.
    // Then, we recursively ensure the deadlines are valid throughout its subtree.

    // 1. ORDER VALUE (and startTime)
    // only applies to the subtree's root
    const { deadlineDate, deadlineTime, id, subtreeDeadlineInMsElapsed } = $whatIsBeingDraggedFullObj

    let updateObj = {
      orderValue: newVal,
      deadlineDate: deadlineDate || '', // read below
      deadlineTime: deadlineTime || '', // the '' optionality makes this usable for milestones
      id,
      subtreeDeadlineInMsElapsed
    }
    
    // 2. UNSCHEDULE: when you drag to the to-do list, it always unschedules it from the calendar
    // updateObj.startDateISO = ''
    // updateObj.startTime = ''

    // 3. KEEP CONSISTENT DEADLINE HANDLING API WITH THE ROOT TODO DROPZONE
    // yes, this is horrific code
    if (!isMilestoneMode) {
      updateObj = correctDeadlineIfNecessary({ 
        node: updateObj, 
        todoListUpperBound: dueInHowManyDays, 
        parentObj,
        batch,
        userDoc: $user
      })


      // THE BELOW CODE CAUSES BUGS
      // updateObj = breakParentRelationIfNecessary(updateObj)
    }

    // NOTE `updateObj` will be deprecated and unused
    const betaUpdateObj = {
      orderValue: newVal,
      startDateISO: '',
      startTime: ''
    }

    // 4. PARENTID
    if ($whatIsBeingDragged === 'top-level-task-within-this-todo-list' && ancestorRoomIDs.length === 1) {
      // preserve parent relationship
    } else {
      updateObj.parentID = parentID
      betaUpdateObj.parentID = parentID
    }

    // 2. HANDLE SUBTREE DEADLINES
    // NOTE: this function actually doesn't do anything
    maintainValidSubtreeDeadlines({ 
      node: $whatIsBeingDraggedFullObj, 
      todoListUpperBound: dueInHowManyDays, 
      parentObj,
      batch,
      userDoc: $user
    })

    // to make it reusable with milestones
    let ref = null
    if (isMilestoneMode) {
      ref = doc(db, `users/${$user.uid}/milestones/${$whatIsBeingDraggedID}`)
    } else {
      ref = doc(db, `users/${$user.uid}/tasks/${$whatIsBeingDraggedID}`)
    }

    batch.update(ref, betaUpdateObj) // updateObj


    try {
      batch.commit()
      updateLocalState({ 
        id, 
        keyValueChanges: betaUpdateObj
      })
      whatIsBeingDraggedFullObj.set(null)
      whatIsBeingDraggedID.set('')
      whatIsBeingDragged.set('')
    } catch (error) {
      alert('Error updating, please reload the page')
    }
  }
</script>