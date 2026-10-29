<script setup>
import { computed, nextTick, onMounted, onUnmounted, ref, watch } from "vue";

const API_URL = "https://itunes.apple.com/search";
const STORAGE_KEYS = {
  favorites: "onde.favorites",
  playlists: "onde.playlists",
};

const songs = ref([]);
const searchInput = ref("");
const submittedQuery = ref("");
const activeView = ref("home");
const selectedPlaylistId = ref(null);
const favorites = ref([]);
const playlists = ref([]);
const currentTrack = ref(null);
const queue = ref([]);
const audio = ref(null);
const isPlaying = ref(false);
const isLoading = ref(false);
const errorMessage = ref("");
const toastMessage = ref("");
const expandedTrackId = ref(null);
const isCreatePlaylistOpen = ref(false);
const newPlaylistName = ref("");
const playlistNameInput = ref(null);
const shuffleEnabled = ref(false);
const repeatMode = ref("off");
const currentTime = ref(0);
const duration = ref(30);
const volume = ref(0.68);
const mainContent = ref(null);
let searchController;
let searchTimer;
let toastTimer;

const activePlaylist = computed(() =>
  playlists.value.find((playlist) => playlist.id === selectedPlaylistId.value) ?? null,
);

const displayTracks = computed(() => {
  if (activeView.value === "favorites") return favorites.value;
  if (activeView.value === "playlist") return activePlaylist.value?.tracks ?? [];
  return songs.value;
});

const pageTitle = computed(() => {
  if (activeView.value === "favorites") return "Vos titres favoris";
  if (activeView.value === "playlist") return activePlaylist.value?.name ?? "Playlist";
  if (activeView.value === "search" && submittedQuery.value) {
    return `Résultats pour « ${submittedQuery.value} »`;
  }
  return "À découvrir";
});

const progress = computed(() =>
  duration.value > 0 ? Math.min((currentTime.value / duration.value) * 100, 100) : 0,
);

function readStorage(key) {
  try {
    const stored = localStorage.getItem(key);
    const value = stored ? JSON.parse(stored) : [];
    if (!Array.isArray(value)) return [];
    if (key === STORAGE_KEYS.playlists) {
      return value.filter((item) => item && typeof item.id === "string" && typeof item.name === "string" && Array.isArray(item.tracks));
    }
    return value.filter((item) => item && item.id && item.title && item.artist);
  } catch (error) {
    console.warn(`Impossible de lire la bibliothèque (${key}).`, error);
    return [];
  }
}

function writeStorage(key, value) {
  try {
    localStorage.setItem(key, JSON.stringify(value));
  } catch (error) {
    console.error(`Impossible d'enregistrer la bibliothèque (${key}).`, error);
    showToast("La bibliothèque n'a pas pu être enregistrée sur cet appareil.");
  }
}

watch(favorites, (value) => writeStorage(STORAGE_KEYS.favorites, value), { deep: true });
watch(playlists, (value) => writeStorage(STORAGE_KEYS.playlists, value), { deep: true });

function formatTime(seconds) {
  const safeSeconds = Number.isFinite(seconds) ? Math.floor(seconds) : 0;
  return `${Math.floor(safeSeconds / 60)}:${String(safeSeconds % 60).padStart(2, "0")}`;
}

function showToast(message) {
  toastMessage.value = message;
  clearTimeout(toastTimer);
  toastTimer = setTimeout(() => {
    toastMessage.value = "";
  }, 2600);
}

function normalizeSong(item) {
  return {
    id: item.trackId,
    title: item.trackName,
    artist: item.artistName,
    album: item.collectionName ?? "Single",
    artwork: item.artworkUrl100?.replace("100x100bb", "600x600bb") ?? "",
    previewUrl: item.previewUrl ?? "",
    duration: 30,
    genre: item.primaryGenreName ?? "",
    releaseDate: item.releaseDate ?? "",
  };
}

async function searchMusic(term, view = "search") {
  const query = term.trim();
  if (!query) {
    errorMessage.value = "Saisissez un titre ou un artiste pour lancer la recherche.";
    activeView.value = "search";
    return;
  }

  searchController?.abort();
  const controller = new AbortController();
  searchController = controller;
  isLoading.value = true;
  errorMessage.value = "";
  activeView.value = view;
  submittedQuery.value = query;

  const url = new URL(API_URL);
  url.searchParams.set("term", query);
  url.searchParams.set("entity", "song");
  url.searchParams.set("limit", "30");
  url.searchParams.set("country", "FR");

  try {
    const response = await fetch(url, { signal: controller.signal });
    if (!response.ok) {
      throw new Error(`L'API musicale a répondu avec le statut ${response.status}.`);
    }

    const data = await response.json();
    songs.value = (data.results ?? [])
      .filter((item) => item.wrapperType === "track" && item.trackId && item.trackName)
      .map(normalizeSong);
    if (songs.value.length === 0) {
      errorMessage.value = "Aucun morceau trouvé. Essayez un autre titre ou artiste.";
    }
  } catch (error) {
    if (error.name === "AbortError") return;
    console.error("La recherche musicale a échoué.", error);
    songs.value = [];
    errorMessage.value = "Impossible de charger la musique. Vérifiez votre connexion puis réessayez.";
  } finally {
    if (!controller.signal.aborted) isLoading.value = false;
  }
}

function submitSearch() {
  clearTimeout(searchTimer);
  void searchMusic(searchInput.value);
}

function scheduleSearch() {
  if (activeView.value !== "search") return;
  clearTimeout(searchTimer);
  searchTimer = setTimeout(() => {
    if (searchInput.value.trim()) void searchMusic(searchInput.value);
  }, 500);
}

function navigate(view) {
  activeView.value = view;
  selectedPlaylistId.value = null;
  errorMessage.value = "";
  if (view === "home" && submittedQuery.value !== "indie pop france") {
    void searchMusic("indie pop france", "home");
  }
  if (view === "search") {
    nextTick(() => document.querySelector("#search-input")?.focus());
  }
}

function openPlaylist(playlist) {
  activeView.value = "playlist";
  selectedPlaylistId.value = playlist.id;
  errorMessage.value = "";
}

function toggleFavorite(song) {
  const index = favorites.value.findIndex((item) => item.id === song.id);
  if (index >= 0) {
    favorites.value.splice(index, 1);
    showToast("Retiré de vos titres favoris");
  } else {
    favorites.value.unshift(song);
    showToast("Ajouté à vos titres favoris");
  }
}

function isFavorite(song) {
  return favorites.value.some((item) => item.id === song.id);
}

function createPlaylist() {
  newPlaylistName.value = "";
  isCreatePlaylistOpen.value = true;
  nextTick(() => playlistNameInput.value?.focus());
}

function submitPlaylistCreation() {
  const trimmedName = newPlaylistName.value.trim();
  if (!trimmedName) {
    showToast("Le nom de la playlist ne peut pas être vide.");
    return;
  }
  const playlist = {
    id: globalThis.crypto?.randomUUID?.() ?? `${Date.now()}`,
    name: trimmedName,
    tracks: [],
  };
  playlists.value.push(playlist);
  openPlaylist(playlist);
  isCreatePlaylistOpen.value = false;
  showToast(`Playlist « ${trimmedName} » créée`);
}

function addToPlaylist(song, playlist) {
  if (playlist.tracks.some((item) => item.id === song.id)) {
    showToast("Ce titre est déjà dans cette playlist.");
    return;
  }
  playlist.tracks.push(song);
  expandedTrackId.value = null;
  showToast(`Ajouté à « ${playlist.name} »`);
}

function removeFromPlaylist(song) {
  if (!activePlaylist.value) return;
  activePlaylist.value.tracks = activePlaylist.value.tracks.filter((item) => item.id !== song.id);
  if (currentTrack.value?.id === song.id) stopPlayback();
  showToast("Titre retiré de la playlist");
}

async function playTrack(song, sourceQueue = displayTracks.value) {
  if (!song.previewUrl) {
    showToast("Aucun extrait audio n'est disponible pour ce titre.");
    return;
  }

  if (currentTrack.value?.id === song.id && audio.value) {
    if (audio.value.paused) {
      try {
        await audio.value.play();
      } catch (error) {
        console.error("La lecture audio a échoué.", error);
        showToast("La lecture a échoué. Réessayez dans un instant.");
      }
    } else {
      audio.value.pause();
    }
    return;
  }

  queue.value = sourceQueue.length ? [...sourceQueue] : [song];
  currentTrack.value = song;
  currentTime.value = 0;
  duration.value = song.duration || 30;
  await nextTick();

  if (!audio.value) {
    showToast("Le lecteur audio n'est pas disponible.");
    return;
  }

  audio.value.src = song.previewUrl;
  audio.value.volume = volume.value;
  audio.value.load();
  try {
    await audio.value.play();
  } catch (error) {
    isPlaying.value = false;
    console.error("La lecture audio a échoué.", error);
    showToast("Impossible de lire cet extrait. Vous pouvez essayer un autre titre.");
  }
}

function stopPlayback() {
  audio.value?.pause();
  if (audio.value) audio.value.currentTime = 0;
  isPlaying.value = false;
  currentTime.value = 0;
}

async function playNext() {
  if (!queue.value.length || !currentTrack.value) return;
  if (repeatMode.value === "one") {
    audio.value.currentTime = 0;
    await audio.value.play();
    return;
  }

  const index = queue.value.findIndex((song) => song.id === currentTrack.value.id);
  let nextIndex;
  if (shuffleEnabled.value && queue.value.length > 1) {
    const choices = queue.value.map((_, itemIndex) => itemIndex).filter((itemIndex) => itemIndex !== index);
    nextIndex = choices[Math.floor(Math.random() * choices.length)];
  } else if (index < queue.value.length - 1) {
    nextIndex = index + 1;
  } else if (repeatMode.value === "all") {
    nextIndex = 0;
  } else {
    isPlaying.value = false;
    return;
  }
  await playTrack(queue.value[nextIndex], queue.value);
}

async function playPrevious() {
  if (!queue.value.length || !currentTrack.value) return;
  if (audio.value.currentTime > 3) {
    audio.value.currentTime = 0;
    return;
  }
  const index = queue.value.findIndex((song) => song.id === currentTrack.value.id);
  const previousIndex = index > 0 ? index - 1 : repeatMode.value === "all" ? queue.value.length - 1 : 0;
  await playTrack(queue.value[previousIndex], queue.value);
}

function handleEnded() {
  if (repeatMode.value === "one") {
    audio.value.currentTime = 0;
    void audio.value.play();
    return;
  }
  void playNext();
}

function handleSeek(event) {
  if (!audio.value || !Number.isFinite(audio.value.duration)) return;
  audio.value.currentTime = (Number(event.target.value) / 100) * audio.value.duration;
}

function handleVolume(event) {
  volume.value = Number(event.target.value) / 100;
  if (audio.value) audio.value.volume = volume.value;
}

function toggleRepeat() {
  repeatMode.value = repeatMode.value === "off" ? "all" : repeatMode.value === "all" ? "one" : "off";
  showToast(`Répétition : ${repeatMode.value === "off" ? "désactivée" : repeatMode.value === "all" ? "toute la file" : "un titre"}`);
}

function handleAudioError() {
  if (!currentTrack.value) return;
  isPlaying.value = false;
  showToast("Cet extrait n'est pas disponible pour le moment.");
}

function handleKeydown(event) {
  if ((event.metaKey || event.ctrlKey) && event.key.toLowerCase() === "k") {
    event.preventDefault();
    navigate("search");
  } else if (
    event.code === "Space" &&
    !["INPUT", "BUTTON", "TEXTAREA"].includes(document.activeElement?.tagName)
  ) {
    event.preventDefault();
    if (currentTrack.value && audio.value) {
      if (audio.value.paused) void audio.value.play();
      else audio.value.pause();
    }
  }
}

onMounted(() => {
  favorites.value = readStorage(STORAGE_KEYS.favorites);
  playlists.value = readStorage(STORAGE_KEYS.playlists);
  window.addEventListener("keydown", handleKeydown);
  void searchMusic("indie pop france", "home");
});

onUnmounted(() => {
  searchController?.abort();
  clearTimeout(searchTimer);
  clearTimeout(toastTimer);
  window.removeEventListener("keydown", handleKeydown);
  audio.value?.pause();
});
</script>

<template>
  <div class="app-shell">
    <aside class="sidebar">
      <a class="brand" href="#" aria-label="Onde, accueil" @click.prevent="navigate('home')">
        <span class="brand-mark" aria-hidden="true">◖</span>
        <span>onde</span>
      </a>

      <nav class="main-nav" aria-label="Navigation principale">
        <button class="nav-link" :class="{ active: activeView === 'home' }" @click="navigate('home')">
          <span aria-hidden="true">⌂</span> Accueil
        </button>
        <button class="nav-link" :class="{ active: activeView === 'search' }" @click="navigate('search')">
          <span aria-hidden="true">⌕</span> Rechercher
        </button>
        <button class="nav-link" :class="{ active: activeView === 'favorites' || activeView === 'playlist' }" @click="navigate('favorites')">
          <span aria-hidden="true">▤</span> Bibliothèque
        </button>
      </nav>

      <div class="library-heading">
        <h2>Votre bibliothèque</h2>
        <button class="icon-button add-button" type="button" aria-label="Créer une playlist" title="Créer une playlist" @click="createPlaylist">＋</button>
      </div>
      <button class="library-filter selected" type="button" @click="navigate('favorites')">Titres favoris</button>
      <div class="playlist-list">
        <button
          v-for="playlist in playlists"
          :key="playlist.id"
          class="playlist-row"
          :class="{ 'playlist-row-active': selectedPlaylistId === playlist.id }"
          type="button"
          @click="openPlaylist(playlist)"
        >
          <span class="mini-cover liked-cover">♫</span>
          <span><b>{{ playlist.name }}</b><small>Playlist · {{ playlist.tracks.length }} titre{{ playlist.tracks.length === 1 ? "" : "s" }}</small></span>
        </button>
        <p v-if="playlists.length === 0" class="empty-library">Créez une playlist pour y garder vos découvertes.</p>
      </div>
      <div class="sidebar-note">
        <span class="note-icon">♫</span>
        <p><b>La musique en liberté</b><br>Des extraits, des découvertes, à votre façon.</p>
      </div>
    </aside>

    <main ref="mainContent" class="main-content">
      <header class="topbar">
        <div class="history-buttons" aria-label="Historique de navigation">
          <button class="icon-button" type="button" aria-label="Accueil" @click="navigate('home')">‹</button>
          <button class="icon-button" type="button" aria-label="Recherche" @click="navigate('search')">›</button>
        </div>
        <form class="search-box" role="search" @submit.prevent="submitSearch">
          <span aria-hidden="true">⌕</span>
          <input
            id="search-input"
            v-model="searchInput"
            type="search"
            placeholder="Que souhaitez-vous écouter ?"
            aria-label="Rechercher un titre ou un artiste"
            @input="scheduleSearch"
          >
          <kbd>Ctrl K</kbd>
        </form>
        <div class="profile-button"><span class="profile-avatar">♫</span> Ma musique</div>
      </header>

      <section class="welcome-section">
        <div>
          <p class="eyebrow"><span class="live-dot"></span> VOTRE ESPACE MUSICAL</p>
          <h1>{{ activeView === "home" ? "Bonjour, bienvenue" : "Trouvez votre prochain titre" }} <span class="wave">✳</span></h1>
          <p class="welcome-copy">Parcourez des morceaux et écoutez leurs extraits gratuitement.</p>
        </div>
        <button
          class="mood-button"
          type="button"
          :disabled="!songs.length"
          @click="playTrack(songs[Math.floor(Math.random() * songs.length)], songs)"
        >✦ <span>Surprenez-moi</span></button>
      </section>

      <section class="quick-picks" aria-label="Suggestions de recherche">
        <button v-for="suggestion in ['Indie pop', 'Afrobeats', 'Jazz', 'Pop française']" :key="suggestion" class="quick-card" type="button" @click="searchInput = suggestion; searchMusic(suggestion)">
          <span class="quick-art art-sun">♫</span><b>{{ suggestion }}</b><span class="quick-play" aria-hidden="true">→</span>
        </button>
      </section>

      <section class="music-section" id="titres">
        <div class="section-heading">
          <div><p class="eyebrow">{{ activeView === "search" ? "RECHERCHE MUSICALE" : activeView === "favorites" ? "VOTRE COLLECTION" : activeView === "playlist" ? "VOTRE PLAYLIST" : "PROPULSÉ PAR L'API ITUNES" }}</p><h2>{{ pageTitle }}</h2></div>
          <button v-if="displayTracks.length" class="text-action" type="button" @click="playTrack(displayTracks[0], displayTracks)">Tout lire <span aria-hidden="true">▶</span></button>
        </div>

        <div v-if="isLoading" class="status-message"><span class="loading-spinner"></span> Recherche de morceaux…</div>
        <div v-else-if="errorMessage" class="status-message error-message">
          <span>{{ errorMessage }}</span>
          <button v-if="submittedQuery" class="retry-button" type="button" @click="searchMusic(searchInput || submittedQuery, activeView)">Réessayer</button>
        </div>
        <div v-else-if="displayTracks.length === 0" class="status-message">
          {{ activeView === "favorites" ? "Vos titres favoris apparaîtront ici." : activeView === "playlist" ? "Cette playlist est vide. Ajoutez-y des titres depuis les résultats." : "Aucun morceau à afficher." }}
        </div>

        <div v-else class="track-list">
          <article v-for="(song, index) in displayTracks" :key="song.id" class="track-row" :class="{ 'track-row-current': currentTrack?.id === song.id }">
            <button class="track-number" type="button" :aria-label="currentTrack?.id === song.id && isPlaying ? 'Mettre en pause' : `Lire ${song.title}`" @click="playTrack(song, displayTracks)">
              {{ currentTrack?.id === song.id && isPlaying ? "Ⅱ" : String(index + 1).padStart(2, "0") }}
            </button>
            <button class="track-art-button" type="button" :aria-label="`Lire ${song.title}`" @click="playTrack(song, displayTracks)">
              <img v-if="song.artwork" class="track-thumb" :src="song.artwork" :alt="`Pochette de ${song.album}`" loading="lazy">
              <span v-else class="track-thumb cover-lilac">♫</span>
            </button>
            <button class="track-details" type="button" @click="playTrack(song, displayTracks)">
              <b>{{ song.title }}</b><small>{{ song.artist }} · {{ song.album }}</small>
            </button>
            <span class="track-album">{{ song.genre || song.album }}</span>
            <span class="track-duration">{{ formatTime(song.duration) }}</span>
            <button class="row-action" type="button" :aria-label="isFavorite(song) ? 'Retirer des favoris' : 'Ajouter aux favoris'" :title="isFavorite(song) ? 'Retirer des favoris' : 'Ajouter aux favoris'" @click="toggleFavorite(song)">
              {{ isFavorite(song) ? "♥" : "♡" }}
            </button>
            <div class="playlist-action">
              <button class="row-action" type="button" aria-label="Ajouter à une playlist" title="Ajouter à une playlist" @click="expandedTrackId = expandedTrackId === song.id ? null : song.id">＋</button>
              <div v-if="expandedTrackId === song.id" class="playlist-popover">
                <button v-for="playlist in playlists" :key="playlist.id" type="button" @click="addToPlaylist(song, playlist)">{{ playlist.name }}</button>
                <button type="button" @click="createPlaylist">＋ Créer une playlist</button>
              </div>
            </div>
            <button v-if="activeView === 'playlist'" class="row-action remove-action" type="button" aria-label="Retirer de la playlist" @click="removeFromPlaylist(song)">×</button>
          </article>
        </div>
        <p class="api-note">Les extraits audio sont fournis par l’API iTunes. Leur durée varie selon le morceau ; la lecture complète nécessite une plateforme disposant des droits.</p>
      </section>
    </main>
  </div>

  <footer class="player">
    <div class="now-playing">
      <img v-if="currentTrack?.artwork" class="player-cover" :src="currentTrack.artwork" :alt="`Pochette de ${currentTrack.album}`">
      <span v-else class="player-cover cover-lilac">♫</span>
      <span class="now-playing-details">
        <b>{{ currentTrack?.title ?? "Choisissez un titre" }}</b>
        <small>{{ currentTrack?.artist ?? "Votre lecteur Onde" }}</small>
      </span>
      <button v-if="currentTrack" class="icon-button favorite-button" :class="{ 'is-favorite': isFavorite(currentTrack) }" type="button" :aria-label="isFavorite(currentTrack) ? 'Retirer des favoris' : 'Ajouter aux favoris'" @click="toggleFavorite(currentTrack)">{{ isFavorite(currentTrack) ? "♥" : "♡" }}</button>
    </div>
    <div class="player-center">
      <div class="player-controls">
        <button class="player-icon shuffle-button" :class="{ selected: shuffleEnabled }" type="button" aria-label="Lecture aléatoire" @click="shuffleEnabled = !shuffleEnabled; showToast(shuffleEnabled ? 'Lecture aléatoire activée' : 'Lecture aléatoire désactivée')">⤨</button>
        <button class="player-icon" type="button" aria-label="Titre précédent" @click="playPrevious">|◀</button>
        <button class="play-button" type="button" :aria-label="isPlaying ? 'Mettre en pause' : 'Lire'" :disabled="!currentTrack" @click="audio?.paused ? audio.play() : audio?.pause()">{{ isPlaying ? "Ⅱ" : "▶" }}</button>
        <button class="player-icon" type="button" aria-label="Titre suivant" @click="playNext">▶|</button>
        <button class="player-icon repeat-button" :class="{ selected: repeatMode !== 'off' }" type="button" :aria-label="`Mode répétition : ${repeatMode}`" @click="toggleRepeat">{{ repeatMode === "one" ? "↻¹" : "↻" }}</button>
      </div>
      <div class="progress-row">
        <span>{{ formatTime(currentTime) }}</span>
        <input type="range" min="0" max="100" :value="progress" aria-label="Position dans le titre" :disabled="!currentTrack" @input="handleSeek">
        <span>{{ formatTime(duration) }}</span>
      </div>
    </div>
    <div class="player-extras"><span aria-hidden="true">◖))</span><input type="range" min="0" max="100" :value="volume * 100" aria-label="Volume" @input="handleVolume"></div>
    <audio
      ref="audio"
      preload="none"
      @play="isPlaying = true"
      @pause="isPlaying = false"
      @timeupdate="currentTime = audio?.currentTime ?? 0"
      @loadedmetadata="duration = Number.isFinite(audio?.duration) ? audio.duration : (currentTrack?.duration ?? 30)"
      @ended="handleEnded"
      @error="handleAudioError"
    ></audio>
  </footer>
  <div v-if="isCreatePlaylistOpen" class="modal-backdrop" @click.self="isCreatePlaylistOpen = false">
    <section class="playlist-modal" role="dialog" aria-modal="true" aria-labelledby="playlist-modal-title" @keydown.esc="isCreatePlaylistOpen = false">
      <button class="modal-close" type="button" aria-label="Fermer" @click="isCreatePlaylistOpen = false">×</button>
      <p class="eyebrow">VOTRE BIBLIOTHÈQUE</p>
      <h2 id="playlist-modal-title">Créer une playlist</h2>
      <p>Donnez un nom à votre nouvelle collection musicale.</p>
      <form @submit.prevent="submitPlaylistCreation">
        <label for="playlist-name">Nom de la playlist</label>
        <input id="playlist-name" ref="playlistNameInput" v-model="newPlaylistName" maxlength="60" placeholder="Ex. Mes découvertes" required>
        <button class="create-button" type="submit">Créer</button>
      </form>
    </section>
  </div>
  <div class="toast" :class="{ visible: toastMessage }" role="status" aria-live="polite">{{ toastMessage }}</div>
</template>
