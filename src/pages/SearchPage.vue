<template>
  <q-page class="search-page">
    <!-- 未搜索时显示热门标签 -->
    <div v-if="!hasSearched" class="search-header">
      <div class="hot-tags">
        <span class="tags-label">热门搜索：</span>
        <q-chip
          v-for="tag in hotTags" :key="tag"
          clickable dense color="dark" text-color="grey-4" class="hot-tag"
          @click="searchByTag(tag)"
        >{{ tag }}</q-chip>
      </div>
    </div>

    <div class="search-results">
      <div v-if="loading" class="search-loading">
        <q-spinner-dots color="primary" size="40px" />
        <p>正在搜索...</p>
      </div>

      <div v-else-if="!hasSearched" class="search-empty">
        <q-icon name="mdi-music-search" size="80px" color="grey-8" />
        <p class="empty-title">搜索 B 站音乐</p>
        <p class="empty-sub">输入歌名、UP 主名或关键词</p>
      </div>

      <div v-else-if="error" class="search-empty">
        <q-icon name="mdi-wifi-off" size="60px" color="grey-7" />
        <p class="empty-title">搜索失败</p>
        <p class="empty-sub">{{ error }}</p>
        <q-btn flat color="primary" label="重试" @click="doSearch" />
      </div>

      <div v-else-if="results.length === 0" class="search-empty">
        <q-icon name="mdi-magnify-remove-outline" size="80px" color="grey-8" />
        <p class="empty-title">未找到相关内容</p>
        <p class="empty-sub">试试其他关键词</p>
      </div>

      <template v-else>
        <div class="results-header">
          <span class="results-count">找到 {{ results.length }} 个结果</span>
        </div>

        <div class="result-list">
          <div
            v-for="item in results" :key="item.bvid"
            class="result-item"
            :class="{ playing: playerStore.currentTrack?.bvid === item.bvid }"
            @click="handlePlay(item)"
          >
            <div class="result-cover">
              <img :src="item.cover" :alt="item.title" />
              <div class="play-overlay">
                <q-spinner-dots v-if="loadingBvid === item.bvid" color="white" size="24px" />
                <q-btn v-else round unelevated color="primary" icon="mdi-play" size="sm" />
              </div>
              <div class="duration-badge">{{ item.duration }}</div>
            </div>

            <div class="result-info">
              <div class="result-title">{{ item.title }}</div>
              <div class="result-meta">
                <q-avatar size="18px"><img :src="item.authorFace" /></q-avatar>
                <span class="result-author">{{ item.author }}</span>
                <span class="result-sep">·</span>
                <span class="result-play">
                  <q-icon name="mdi-play-circle-outline" size="14px" />
                  {{ formatCount(item.play) }}
                </span>
                <span class="result-sep">·</span>
                <span>{{ item.pubdate }}</span>
              </div>
              <div class="result-desc">{{ item.description }}</div>
            </div>

            <div class="result-actions" @click.stop>
              <q-btn
                flat round dense size="sm" icon="mdi-call-split" color="grey-6"
                :disable="loadingSplitBvid === item.bvid"
                @click="openSplit(item)"
              >
                <q-spinner v-if="loadingSplitBvid === item.bvid" />
                <q-tooltip v-else>拆分合集为单曲</q-tooltip>
              </q-btn>
              <q-btn
                flat round dense size="sm"
                :icon="favStore.isFavorited(item.bvid) ? 'mdi-heart' : 'mdi-heart-outline'"
                :color="favStore.isFavorited(item.bvid) ? 'primary' : 'grey-6'"
                @click="toggleFav(item)"
              >
                <q-tooltip>{{ favStore.isFavorited(item.bvid) ? '取消收藏' : '收藏' }}</q-tooltip>
              </q-btn>
              <q-btn flat round dense size="sm" icon="mdi-playlist-plus" color="grey-6">
                <q-tooltip>添加到歌单</q-tooltip>
                <q-menu dark>
                  <q-list style="min-width:160px">
                    <q-item clickable v-close-popup @click="addToQueue(item)">
                      <q-item-section avatar><q-icon name="mdi-playlist-music" /></q-item-section>
                      <q-item-section>加入播放队列</q-item-section>
                    </q-item>
                    <q-separator dark />
                    <q-item
                      v-for="pl in libraryStore.playlists" :key="pl.id"
                      clickable v-close-popup
                      @click="addToPlaylist(item, pl.id)"
                    >
                      <q-item-section avatar><q-icon name="mdi-music-note" /></q-item-section>
                      <q-item-section>{{ pl.name }}</q-item-section>
                    </q-item>
                    <q-item v-if="!libraryStore.playlists.length" disable>
                      <q-item-section>暂无歌单</q-item-section>
                    </q-item>
                  </q-list>
                </q-menu>
              </q-btn>
            </div>
          </div>
        </div>
      </template>
    </div>

    <!-- 合集拆分对话框 -->
    <q-dialog v-model="splitOpen">
      <q-card dark class="split-card">
        <q-card-section class="split-header">
          <div class="split-head-info">
            <div class="split-title">{{ splitTitle }}</div>
            <div class="split-sub">共 {{ splitSongs.length }} 首 · 点击任意一首播放</div>
          </div>
          <q-space />
          <q-btn flat dense icon="mdi-playlist-plus" label="全部加入队列" color="primary" no-caps @click="addAllSplitToQueue" />
          <q-btn flat round dense icon="mdi-close" color="grey-6" v-close-popup />
        </q-card-section>
        <q-separator dark />
        <q-card-section class="split-list">
          <div v-if="splitPlaying" class="split-loading">
            <q-spinner-dots color="primary" size="32px" />
            <span>正在获取音频...</span>
          </div>
          <div
            v-else
            v-for="(song, i) in splitSongs" :key="song.bvid + '-' + song.cid"
            class="split-item"
            :class="{ active: playerStore.currentTrack?.id === splitSongId(song) }"
            @click="playSplitSong(i)"
          >
            <span class="split-idx">
              <q-icon v-if="playerStore.currentTrack?.id === splitSongId(song)" name="mdi-volume-high" color="primary" />
              <template v-else>{{ i + 1 }}</template>
            </span>
            <span class="split-song-title">{{ song.title }}</span>
            <span class="split-dur">{{ fmtSec(song.duration) }}</span>
            <q-btn flat round dense size="sm" icon="mdi-play" color="primary" @click.stop="playSplitSong(i)">
              <q-tooltip>播放</q-tooltip>
            </q-btn>
            <q-btn flat round dense size="sm" icon="mdi-playlist-plus" color="grey-6" @click.stop="addSplitToQueue(i)">
              <q-tooltip>加入队列</q-tooltip>
            </q-btn>
          </div>
        </q-card-section>
      </q-card>
    </q-dialog>
  </q-page>
</template>

<script setup lang="ts">
import { ref, onMounted, watch } from 'vue'
import { useRoute, useRouter } from 'vue-router'
import { Notify } from 'quasar'
import { searchVideos, getVideoCid, getVideoDetail, getAudioUrl, type SearchResult, type SplitSong } from '../api/bilibili'
import { usePlayerStore } from '../stores/player'
import { useFavoritesStore } from '../stores/favorites'
import { useLibraryStore } from '../stores/library'

const route = useRoute()
const router = useRouter()
const playerStore = usePlayerStore()
const favStore = useFavoritesStore()
const libraryStore = useLibraryStore()

const keyword = ref('')
const loading = ref(false)
const hasSearched = ref(false)
const error = ref('')
const results = ref<SearchResult[]>([])
const loadingBvid = ref('')
const loadingSplitBvid = ref('')

// 拆分对话框状态
const splitOpen = ref(false)
const splitTitle = ref('')
const splitSongs = ref<SplitSong[]>([])
const splitAuthor = ref('')
const splitCover = ref('')
const splitPlaying = ref(false)

const hotTags = ['周杰伦', '陈奕迅', '邓紫棋', '翻唱', '钢琴', 'OST', '国风', '说唱', 'Vocaloid', '纯音乐']

function searchByTag(tag: string) {
  router.push({ path: '/search', query: { q: tag } })
}

function formatCount(n: number): string {
  if (n >= 10000) return `${(n / 10000).toFixed(1)}万`
  return String(n)
}

async function doSearch() {
  if (!keyword.value.trim()) return
  loading.value = true
  hasSearched.value = true
  error.value = ''
  try {
    results.value = await searchVideos(keyword.value.trim())
  } catch (e: unknown) {
    error.value = e instanceof Error ? e.message : '请求失败，请检查网络'
  } finally {
    loading.value = false
  }
}

function splitSongId(song: SplitSong): string {
  return `${song.bvid}::${song.cid}`
}

function fmtSec(sec: number): string {
  if (!sec) return '--:--'
  return `${Math.floor(sec / 60)}:${Math.floor(sec % 60).toString().padStart(2, '0')}`
}

// ─── 合集拆分 ────────────────────────────────────────────────────────────────

interface SplitPick { songs: SplitSong[]; title: string }

async function pickSplitSongs(bvid: string): Promise<SplitPick | null> {
  const detail = await getVideoDetail(bvid)
  // 优先分 P，其次合集
  if (detail.pages.length > 1) return { songs: detail.pages, title: detail.title }
  if (detail.seasonEpisodes.length > 1) return { songs: detail.seasonEpisodes, title: detail.seasonTitle || detail.title }
  return null
}

async function openSplit(item: SearchResult) {
  if (loadingSplitBvid.value) return
  loadingSplitBvid.value = item.bvid
  try {
    const pick = await pickSplitSongs(item.bvid)
    if (!pick) {
      Notify.create({ message: '这个视频没有分 P，也不属于合集', icon: 'mdi-information-outline', color: 'dark', timeout: 2000 })
      return
    }
    splitTitle.value = pick.title
    splitSongs.value = pick.songs
    splitAuthor.value = item.author
    splitCover.value = item.cover
    splitOpen.value = true
  } catch (e: unknown) {
    Notify.create({ type: 'negative', message: '获取合集信息失败', caption: e instanceof Error ? e.message : '', timeout: 3000 })
  } finally {
    loadingSplitBvid.value = ''
  }
}

function splitTrack(i: number) {
  const song = splitSongs.value[i]!
  return {
    id: splitSongId(song),
    bvid: song.bvid,
    cid: song.cid,
    title: song.title,
    author: splitAuthor.value,
    cover: splitCover.value,
    duration: song.duration,
  }
}

async function playSplitSong(i: number) {
  const song = splitSongs.value[i]!
  splitPlaying.value = true
  try {
    const url = await getAudioUrl(song.bvid, song.cid)
    const track = splitTrack(i)
    playerStore.addToPlaylist(track)
    await playerStore.playTrack(track, url)
    splitOpen.value = false
  } catch (e: unknown) {
    Notify.create({ type: 'negative', message: '播放失败', caption: e instanceof Error ? e.message : '', timeout: 3000 })
  } finally {
    splitPlaying.value = false
  }
}

function addSplitToQueue(i: number) {
  playerStore.addToPlaylist(splitTrack(i))
  Notify.create({ message: '已加入播放队列', icon: 'mdi-playlist-plus', color: 'dark', timeout: 1200 })
}

function addAllSplitToQueue() {
  for (let i = 0; i < splitSongs.value.length; i++) {
    playerStore.addToPlaylist(splitTrack(i))
  }
  Notify.create({ message: `已加入 ${splitSongs.value.length} 首到播放队列`, icon: 'mdi-playlist-plus', color: 'dark', timeout: 1500 })
}

async function handlePlay(item: SearchResult) {
  if (loadingBvid.value) return
  loadingBvid.value = item.bvid
  try {
    // 先看是否是多 P / 合集，是则弹出拆分列表
    const pick = await pickSplitSongs(item.bvid)
    if (pick) {
      splitTitle.value = pick.title
      splitSongs.value = pick.songs
      splitAuthor.value = item.author
      splitCover.value = item.cover
      splitOpen.value = true
      return
    }
    const cid = await getVideoCid(item.bvid)
    const url = await getAudioUrl(item.bvid, cid)
    const track = {
      id: item.bvid,
      bvid: item.bvid,
      cid,
      title: item.title,
      author: item.author,
      cover: item.cover,
      duration: 0,
    }
    playerStore.addToPlaylist(track)
    await playerStore.playTrack(track, url)
  } catch (e: unknown) {
    const msg = e instanceof Error ? e.message : '未知错误'
    Notify.create({
      type: 'negative',
      message: '播放失败',
      caption: msg,
      timeout: 4000,
    })
    console.error('播放失败详情:', e)
  } finally {
    loadingBvid.value = ''
  }
}

function toggleFav(item: SearchResult) {
  const track = {
    id: item.bvid, bvid: item.bvid,
    title: item.title, author: item.author,
    cover: item.cover, duration: 0,
  }
  const added = favStore.toggle(track)
  Notify.create({
    message: added ? '已添加到收藏' : '已取消收藏',
    icon: added ? 'mdi-heart' : 'mdi-heart-outline',
    color: 'dark', timeout: 1500,
  })
}

function addToQueue(item: SearchResult) {
  playerStore.addToPlaylist({
    id: item.bvid, bvid: item.bvid,
    title: item.title, author: item.author,
    cover: item.cover, duration: 0,
  })
  Notify.create({ message: '已加入播放队列', icon: 'mdi-playlist-plus', color: 'dark', timeout: 1500 })
}

function addToPlaylist(item: SearchResult, playlistId: string) {
  libraryStore.addTrackToPlaylist(playlistId, {
    id: item.bvid, bvid: item.bvid,
    title: item.title, author: item.author,
    cover: item.cover, duration: 0,
  })
  const pl = libraryStore.playlists.find((p) => p.id === playlistId)
  Notify.create({ message: `已添加到「${pl?.name}」`, icon: 'mdi-check', color: 'dark', timeout: 1500 })
}

onMounted(() => {
  const q = route.query.q as string
  if (q) { keyword.value = q; doSearch() }
})

// 顶栏重新输入搜索词时，路由 query 变化 → 自动触发新搜索
watch(() => route.query.q, (q) => {
  if (q && typeof q === 'string') {
    keyword.value = q
    doSearch()
  }
})
</script>

<style lang="scss" scoped>
.search-page { background: var(--color-bg); min-height: 100vh; }
.search-header { padding: 20px 24px 0; }
.hot-tags {
  display: flex;
  align-items: center;
  flex-wrap: wrap;
  gap: 8px;
  .tags-label { font-size: 13px; color: var(--color-text-muted); }
  .hot-tag {
    font-size: 12px;
    cursor: pointer;
    transition: all 0.2s;
    &:hover { background: rgba(251,114,153,0.15) !important; color: var(--color-primary) !important; }
  }
}
.search-results { padding: 20px 24px; }
.search-loading, .search-empty {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  padding: 80px 0;
  gap: 12px;
  color: var(--color-text-muted);
  .empty-title { font-size: 18px; font-weight: 500; color: var(--color-text); margin: 0; }
  .empty-sub { font-size: 14px; margin: 0; }
  p { font-size: 14px; margin: 0; }
}
.results-header {
  display: flex;
  align-items: center;
  margin-bottom: 12px;
  .results-count { font-size: 13px; color: var(--color-text-muted); }
}
.result-list { display: flex; flex-direction: column; gap: 2px; }
.result-item {
  display: flex;
  align-items: flex-start;
  gap: 14px;
  padding: 10px;
  border-radius: 10px;
  cursor: pointer;
  transition: background 0.2s;
  &:hover, &.playing {
    background: var(--color-bg-card);
    .play-overlay { opacity: 1; }
    .result-actions { opacity: 1; }
  }
  &.playing .result-title { color: var(--color-primary); }
}
.result-cover {
  position: relative;
  width: 150px;
  height: 85px;
  border-radius: 8px;
  overflow: hidden;
  flex-shrink: 0;
  img { width: 100%; height: 100%; object-fit: cover; }
  .play-overlay {
    position: absolute;
    inset: 0;
    background: rgba(0,0,0,0.45);
    display: flex;
    align-items: center;
    justify-content: center;
    opacity: 0;
    transition: opacity 0.25s;
  }
  .duration-badge {
    position: absolute;
    bottom: 4px;
    right: 6px;
    background: rgba(0,0,0,0.7);
    color: white;
    font-size: 11px;
    padding: 1px 5px;
    border-radius: 4px;
  }
}
.result-info {
  flex: 1;
  overflow: hidden;
  .result-title {
    font-size: 14px;
    font-weight: 500;
    color: var(--color-text);
    margin-bottom: 6px;
    display: -webkit-box;
    -webkit-line-clamp: 2;
    -webkit-box-orient: vertical;
    overflow: hidden;
    transition: color 0.2s;
  }
  .result-meta {
    display: flex;
    align-items: center;
    gap: 5px;
    font-size: 12px;
    color: var(--color-text-muted);
    margin-bottom: 5px;
    .result-author { color: var(--color-text); }
    .result-sep { opacity: 0.4; }
    .result-play { display: flex; align-items: center; gap: 3px; }
  }
  .result-desc {
    font-size: 12px;
    color: var(--color-text-muted);
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
  }
}
.result-actions {
  display: flex;
  flex-direction: column;
  gap: 4px;
  opacity: 0;
  transition: opacity 0.2s;
}

// ─── 拆分对话框 ──────────────────────────────────────────────────────────────
.split-card {
  width: 560px;
  max-width: 90vw;
  background: var(--color-bg-card, #1e1e24);
  border-radius: 12px;
  .split-header {
    display: flex;
    align-items: center;
    padding-bottom: 8px;
    .split-head-info { min-width: 0; margin-right: 12px; }
    .split-title {
      font-size: 15px;
      font-weight: 500;
      color: var(--color-text);
      white-space: nowrap;
      overflow: hidden;
      text-overflow: ellipsis;
    }
    .split-sub { font-size: 12px; color: var(--color-text-muted); margin-top: 2px; }
  }
  .split-list {
    padding-top: 4px;
    max-height: 55vh;
    overflow-y: auto;
  }
  .split-loading {
    display: flex;
    align-items: center;
    justify-content: center;
    gap: 10px;
    padding: 32px 0;
    color: var(--color-text-muted);
    font-size: 13px;
  }
  .split-item {
    display: flex;
    align-items: center;
    gap: 10px;
    padding: 7px 8px;
    border-radius: 8px;
    cursor: pointer;
    transition: background 0.15s;
    &:hover { background: rgba(255, 255, 255, 0.06); }
    &.active .split-song-title { color: var(--color-primary); }
    .split-idx {
      width: 26px;
      flex-shrink: 0;
      text-align: center;
      font-size: 12px;
      color: var(--color-text-muted);
      display: flex;
      justify-content: center;
    }
    .split-song-title {
      flex: 1;
      font-size: 13px;
      color: var(--color-text);
      white-space: nowrap;
      overflow: hidden;
      text-overflow: ellipsis;
      transition: color 0.15s;
    }
    .split-dur {
      flex-shrink: 0;
      font-size: 11px;
      color: var(--color-text-muted);
      font-variant-numeric: tabular-nums;
    }
  }
}
</style>
