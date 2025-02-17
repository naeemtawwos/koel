<template>
  <div v-koel-focus class="about text-secondary" data-testid="about-modal" tabindex="0" @keydown.esc="close">
    <header>
      <h1 class="text-primary">About Bayan FM</h1>
    </header>

    <main>
      <div class="logo">
        <img alt="Koel's logo" src="@/../img/logo.svg" width="128">
      </div>

      <p class="current-version">Koel {{ currentVersion }}</p>

      <p v-if="shouldNotifyNewVersion" data-testid="new-version-about">
        <a :href="latestVersionReleaseUrl" target="_blank">
          A new version of Koel is available ({{ latestVersion }})!
        </a>
      </p>

      <p class="author">
        Made with ❤️ by
        <a href="https://github.com/naeemtawwos" rel="noopener" target="_blank">Naeem</a>.

      </p>


      <SponsorList />

      <p>
        Loving Bayan FM? Please consider supporting us by
        <a href="" rel="noopener" target="_blank">Donating</a>
        and subscribing our
        <a href="https://islamqatamil.com" rel="noopener" target="_blank">website</a>.
      </p>
    </main>

    <footer>
      <Btn data-testid="close-modal-btn" red rounded @click.prevent="close">Close</Btn>
    </footer>
  </div>
</template>

<script lang="ts" setup>
import { orderBy } from 'lodash'
import { onMounted, ref } from 'vue'
import { isDemo } from '@/utils'
import { useNewVersionNotification } from '@/composables'
import { http } from '@/services'

import SponsorList from '@/components/meta/SponsorList.vue'
import Btn from '@/components/ui/Btn.vue'

type DemoCredits = {
  name: string
  url: string
}

const credits = ref<DemoCredits[] | null>(null)

const {
  shouldNotifyNewVersion,
  currentVersion,
  latestVersion,
  latestVersionReleaseUrl
} = useNewVersionNotification()

const emit = defineEmits<{ (e: 'close'): void }>()
const close = () => emit('close')

onMounted(async () => {
  credits.value = isDemo() ? orderBy(await http.get<DemoCredits[]>('demo/credits'), 'name') : null
})
</script>

<style lang="scss" scoped>
.about {
  text-align: center;
  max-width: 480px;
  overflow: hidden;
  position: relative;

  main {
    padding: 1.8rem;

    p {
      margin: 1rem 0;
    }
  }

  footer {
    padding: 1rem;
    background: rgba(255, 255, 255, .02);
  }

  a {
    color: var(--color-text-primary);

    &:hover {
      color: var(--color-accent);
    }
  }
}

.credit-wrapper {
  max-height: 9rem;
  overflow: auto;
}

.credits, .credits li {
  display: inline;
}

.credits {
  display: inline;

  li {
    display: inline;

    &:last-child {
      &::before {
        content: ', and '
      }

      &::after {
        content: '.';
      }
    }
  }

  li + li {
    &::before {
      content: ', ';
    }
  }
}

.sponsors {
  margin-top: 1rem;
}
</style>
