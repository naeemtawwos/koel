<template>
  <section v-if="album" id="albumWrapper">
    <ScreenHeaderSkeleton v-if="loading" />

    <ScreenHeader v-if="!loading && album" :layout="songs.length === 0 ? 'collapsed' : headerLayout">
      {{ album.name }}
      <ControlsToggle v-model="showingControls" />

      <template #thumbnail>
        <AlbumThumbnail :entity="album" />
      </template>

      <template #meta>
        <a v-if="isNormalArtist" :href="`#/artist/${album.artist_id}`" class="artist">{{ album.artist_name }}</a>
        <span v-else class="nope">{{ album.artist_name }}</span>
        <span>{{ pluralize(songs, 'song') }}</span>
        <span>{{ duration }}</span>

        <a
          v-if="allowDownload"
          class="download"
          role="button"
          title="Download all songs in album"
          @click.prevent="download"
        >
          Download All
        </a>
      </template>

      <template #controls>
        <SongListControls
          v-if="songs.length && (!isPhone || showingControls)"
          @play-all="playAll"
          @play-selected="playSelected"
        />
      </template>
    </ScreenHeader>

    <ScreenTabs>
      <template #header>
        <label :class="{ active: activeTab === 'Songs' }">
          Audios
          <input type="radio" name="tab" value="Songs" v-model="activeTab"/>
        </label>
        <label :class="{ active: activeTab === 'OtherAlbums' }">
          Other Albums
          <input v-model="activeTab" type="radio" name="tab" value="OtherAlbums">
        </label>
        <label v-if="useLastfm" :class="{ active: activeTab === 'Info' }">
          Information
          <input v-model="activeTab" type="radio" name="tab" value="Info">
        </label>
      </template>

      <div v-show="activeTab === 'Songs'">
        <SongListSkeleton v-if="loading" />
        <SongList
          v-else
          ref="songList"
          @sort="sort"
          @press:enter="onPressEnter"
          @scroll-breakpoint="onScrollBreakpoint"
        />
      </div>

      <div v-show="activeTab === 'OtherAlbums'" class="albums-pane" data-testid="albums-pane">
        <template v-if="otherAlbums">
          <ul v-if="otherAlbums.length" class="as-list">
            <li v-for="a in otherAlbums" :key="a.id">
              <AlbumCard :album="a" layout="compact" />
            </li>
          </ul>
          <p v-else class="text-secondary">No other albums by {{ album.artist_name }} found in the library.</p>
        </template>
        <ul v-else class="as-list">
          <li v-for="i in 12" :key="i">
            <AlbumCardSkeleton layout="compact" />
          </li>
        </ul>
      </div>

      <div v-show="activeTab === 'Info'" v-if="useLastfm && album" class="info-pane">
        <AlbumInfo :album="album" mode="full" />
      </div>
    </ScreenTabs>
  </section>
</template>

<script lang="ts" setup>
import { computed, defineAsyncComponent, onMounted, ref, toRef, watch } from 'vue'
import { eventBus, logger, pluralize } from '@/utils'
import { albumStore, artistStore, commonStore, songStore } from '@/stores'
import { downloadService } from '@/services'
import { useDialogBox, useRouter, useSongList } from '@/composables'

import ScreenHeader from '@/components/ui/ScreenHeader.vue'
import AlbumThumbnail from '@/components/ui/AlbumArtistThumbnail.vue'
import ScreenHeaderSkeleton from '@/components/ui/skeletons/ScreenHeaderSkeleton.vue'
import SongListSkeleton from '@/components/ui/skeletons/SongListSkeleton.vue'
import ScreenTabs from '@/components/ui/ArtistAlbumScreenTabs.vue'

type Tab = 'Songs' | 'OtherAlbums' | 'Info'
const activeTab = ref<Tab>('Songs')

const AlbumInfo = defineAsyncComponent(() => import('@/components/album/AlbumInfo.vue'))
const AlbumCard = defineAsyncComponent(() => import('@/components/album/AlbumCard.vue'))
const AlbumCardSkeleton = defineAsyncComponent(() => import('@/components/ui/skeletons/ArtistAlbumCardSkeleton.vue'))

const { showErrorDialog } = useDialogBox()
const { getRouteParam, go, onRouteChanged } = useRouter()

const albumId = ref<number>()
const album = ref<Album>()
const songs = ref<Song[]>([])
const loading = ref(false)
let otherAlbums = ref<Album[]>()
let info = ref<ArtistInfo | null>()

const {
  SongList,
  SongListControls,
  ControlsToggle,
  headerLayout,
  songList,
  showingControls,
  isPhone,
  duration,
  sort,
  onPressEnter,
  playAll,
  playSelected,
  onScrollBreakpoint
} = useSongList(songs)

const useLastfm = toRef(commonStore.state, 'use_last_fm')
const allowDownload = toRef(commonStore.state, 'allow_download')

const isNormalArtist = computed(() => {
  if (!album.value) return true
  return !artistStore.isVarious(album.value.artist_id) && !artistStore.isUnknown(album.value.artist_id)
})

const download = () => downloadService.fromAlbum(album.value!)

watch(activeTab, async tab => {
  if (tab === 'OtherAlbums' && !otherAlbums.value) {
    const albums = await albumStore.fetchForArtist(album.value!.artist_id)
    otherAlbums.value = albums.filter(a => a.id !== album.value!.id)
  }
})

watch(albumId, async id => {
  if (!id) return

  album.value = undefined
  info.value = undefined
  otherAlbums.value = undefined
  activeTab.value = 'Songs'

  loading.value = true

  try {
    [album.value, songs.value] = await Promise.all([
      albumStore.resolve(id),
      songStore.fetchForAlbum(id)
    ])

    sort('track')
  } catch (error) {
    logger.error(error)
    showErrorDialog('Failed to load album. Please try again.', 'Error')
  } finally {
    loading.value = false
  }
})

onMounted(async () => (albumId.value = parseInt(getRouteParam('id')!)))

onRouteChanged(route => route.screen === 'Album' && (albumId.value = parseInt(getRouteParam('id')!)))

// if the current album has been deleted, go back to the list
eventBus.on('SONGS_UPDATED', () => albumStore.byId(albumId.value!) || go('albums'))
</script>

<style lang="scss" scoped>
#albumWrapper {
  @include artist-album-info-wrapper();
}

.albums-pane {
  padding: 1.8rem;

  > ul {
    @include artist-album-wrapper();
  }
}

.info-pane {
  padding: 1.8rem;
}
</style>
