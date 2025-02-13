<script>
  import '/src/app.css'
  import { db } from '../back-end/firestoreConnection'
  import { user, loadingTasks } from '../store/index.js'
  import { goto } from '$app/navigation'
  import { getAuth, onAuthStateChanged } from 'firebase/auth'
  import { doc, setDoc, onSnapshot } from 'firebase/firestore'
  import { onMount } from 'svelte'
  import posthog from 'posthog-js'
  let unsubUserSnapListener = null
  let doingAuth = true

  onMount(() => {
    // fetching user takes around 300 - 500 ms
    onAuthStateChanged(getAuth(), async (resultUser) => {
      if (!resultUser) {
        user.set({})
        goto('/')

        // see how new visitors interacts with home page demos
        posthog.init('phc_Cm2c1eB0MCZLTjJDYHklZ7GUp0Ar7p5bIpF5hkCJPdo', {
          api_host: 'https://us.i.posthog.com',
          person_profiles: 'always' // or 'always' to create profiles for anonymous users as well
        })
      } else {
        goto(`/${resultUser.uid}/${isMobile() ? 'mobile' : ''}`)
        user.set({
          phoneNumber: resultUser.phoneNumber || '',
          uid: resultUser.uid
        })
        // handle the snapshot listener
        const ref = doc(db, '/users/' + resultUser.uid)
        unsubUserSnapListener = onSnapshot(ref, async (snap) => {
          if (!snap.exists()) {
            initializeNewFirestoreUser(ref, resultUser)
          } else {
            user.set({ ...snap.data() }) // augment with id, path, etc. when needed in the future
          }
        })
      }
      doingAuth = false
    })
  })

  function isMobile() {
    return window.innerWidth <= 768 // You can adjust the width threshold as needed
  }

  async function initializeNewFirestoreUser(ref, resultUser) {
    return await setDoc(
      ref,
      {
        uid: resultUser.uid,
        phoneNumber: resultUser.phoneNumber || '',
        email: resultUser.email || ''
        // allTasks: []
      },
      { merge: true }
    ).catch((err) => console.error('error in initializeNewFirestoreUser', err))
  }
</script>

<div
  id="loading-screen-logo-start"
  style="z-index: 99999; background: white; width: 100vw; height: 100vh"
  class="center"
  class:invisible={!(
    doingAuth || $loadingTasks
  )}
>
  <img
    src="/trueoutput-square-nobg.png"
    class="app-loading-logo elementToFadeInAndOut center"
    alt="logo"
    style="width: 48px; height: 48px;"
  />
</div>

<div>
  <slot></slot>
   <!-- <h1>Website out for maintenance</h1> -->
</div>

<style>
  .invisible {
    visibility: hidden;
  }
  /* From Prabhakar's centering solution that works for iOS unlike StackOverflow
  https://github.com/project-feynman/v3/blob/d864f54d9a69e6cdf0beb7818e8b07a85cebb7eb/src/components/SeeExplanation.vue */
  .center {
    position: absolute;
    top: 50%;
    left: 50%;
    transform: translate(-50%, -50%);
  }

  .elementToFadeInAndOut {
    animation: fadeInOut 1.4s ease-out 99 forwards;
  }

  @keyframes fadeInOut {
    0% {
      opacity: 0;
    }
    50% {
      opacity: 1;
    }
    100% {
      opacity: 0;
    }
  }
  @media screen and (min-width: 320px) {
    .app-loading-logo {
      width: 110px;
      height: 110px;
      border-radius: 18px;
    }
  }
  @media screen and (min-width: 768px) {
    .app-loading-logo {
      width: 250px;
      height: 250px;
      border-radius: 40px;
    }
  }
</style>
