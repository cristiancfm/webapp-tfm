<template>
  <v-container>
    <v-card>
      <v-card-title>
        <v-row align="center">
          <v-col>
            <span>{{ $t("venuePlayingQueue.title") }}</span>
          </v-col>
          <v-col cols="auto">
            <v-tooltip location="bottom" :text="$t('venuePlayingQueue.tvMode')">
              <template #activator="{ props }">
                <v-btn
                  v-if="playingQueue?.currently_playing && isVenueLogged"
                  class="mr-2"
                  variant="flat"
                  density="comfortable"
                  icon="tv"
                  v-bind="props"
                  @click="tvDialog = true"
                />
              </template>
            </v-tooltip>
            <v-tooltip
              location="bottom"
              :text="$t('venuePlayingQueue.refresh')"
            >
              <template #activator="{ props }">
                <v-btn
                  v-if="playingQueue"
                  class="mr-2"
                  variant="flat"
                  density="comfortable"
                  icon="refresh"
                  v-bind="props"
                  @click="getPlayingQueue"
                />
              </template>
            </v-tooltip>
            <v-tooltip
              location="bottom"
              :text="$t('venuePlayingQueue.addSong')"
            >
              <template #activator="{ props }">
                <v-btn
                  v-if="playingQueue?.currently_playing"
                  color="primary"
                  variant="flat"
                  density="comfortable"
                  icon="add"
                  v-bind="props"
                  @click="addDialog = true"
                />
              </template>
            </v-tooltip>
          </v-col>
        </v-row>
      </v-card-title>
      <v-divider />
      <v-card-text v-if="!loading">
        <v-row v-if="!playingQueue" class="text-center">
          <v-col>
            <v-icon size="x-large" class="mb-2">error</v-icon>
            <p>{{ $t("venuePlayingQueue.error.noQueue") }}</p>
            <div v-if="isVenueLogged" class="ma-4">
              <v-divider />
              <v-icon size="x-large" class="my-2">school</v-icon>
              <p>{{ $t("venuePlayingQueue.tutorial.title") }}</p>
              <br />
              <ul class="text-left">
                <li v-html="$t('venuePlayingQueue.tutorial.step1')" />
                <li v-html="$t('venuePlayingQueue.tutorial.step2')" />
                <li v-html="$t('venuePlayingQueue.tutorial.step3')" />
              </ul>
            </div>
          </v-col>
        </v-row>
        <div v-else>
          <v-row v-if="playingQueue.currently_playing">
            <v-col>
              <h4 class="ml-3">
                {{ $t("venuePlayingQueue.currentlyPlaying") }}
              </h4>
              <v-list>
                <track-item
                  :track="playingQueue.currently_playing"
                  playback
                  @update:playback="getPlayingQueue"
                />
              </v-list>
            </v-col>
          </v-row>
          <v-row v-if="playingQueue.queue.length === 0" class="text-center">
            <v-col>
              <v-icon size="x-large" class="mb-2">info</v-icon>
              <p>{{ $t("venuePlayingQueue.empty") }}</p>
              <div v-if="isVenueLogged" class="ma-4">
                <v-divider />
                <v-icon size="x-large" class="my-2">school</v-icon>
                <p>{{ $t("venuePlayingQueue.tutorial.title") }}</p>
                <br />
                <ul class="text-left">
                  <li v-html="$t('venuePlayingQueue.tutorial.step1')" />
                  <li v-html="$t('venuePlayingQueue.tutorial.step2')" />
                  <li v-html="$t('venuePlayingQueue.tutorial.step3')" />
                </ul>
              </div>
            </v-col>
          </v-row>
          <div v-if="playingQueue.queue.length > 0">
            <v-divider />
            <v-row>
              <v-col>
                <h4 class="ml-3 mt-4">{{ $t("venuePlayingQueue.nextUp") }}</h4>
                <v-list>
                  <track-item
                    v-for="(track, index) in playingQueue.queue"
                    :key="index"
                    :track="track"
                  />
                </v-list>
              </v-col>
            </v-row>
          </div>
        </div>
      </v-card-text>
      <v-card-text v-if="loading">
        <v-row class="text-center">
          <v-col>
            <v-row>
              <v-col>
                <v-progress-circular
                  color="primary"
                  indeterminate
                  class="mb-2"
                />
                <p>{{ $t("loading") }}</p>
              </v-col>
            </v-row>
          </v-col>
        </v-row>
      </v-card-text>
    </v-card>
    <add-track-dialog
      :dialog="addDialog"
      @update:dialog="addDialog = false"
      @update:add-to-queue="addToQueue"
    />
    <tv-mode-dialog
      :dialog="tvDialog"
      :playing-queue="playingQueue"
      :venue="venue"
      @update:dialog="tvDialog = false"
    />
  </v-container>
</template>

<script>
import { mapStores } from "pinia";
import { useSpotifyStore } from "~/store/spotify.js";
import { getAccessToken, redirectToAuthCodeFlow } from "~/utils/spotifyAuth.js";
import { useWebsiteStore } from "~/store/website.js";
import AddTrackDialog from "~/components/dialogs/addTrackDialog.vue";
import TrackItem from "~/components/venues/trackItem.vue";
import TvModeDialog from "~/components/dialogs/tvModeDialog.vue";

const { fetchQueue, submitTrackToQueue } = useSpotify();

export default {
  components: { TvModeDialog, AddTrackDialog, TrackItem },
  props: {
    venue: {
      type: Object,
      default: () => ({}),
    },
  },
  data() {
    return {
      loading: false,
      playingQueue: null,
      currentTrackId: null,
      queueInterval: null,
      addDialog: false,
      tvDialog: false,
    };
  },
  computed: {
    ...mapStores(useSpotifyStore, useWebsiteStore),
    isClientLogged() {
      return this.websiteStore.loggedAuthority === "client";
    },
    isVenueLogged() {
      return this.websiteStore.loggedAuthority === "venue";
    },
  },
  mounted() {
    this.getPlayingQueue();
    this.queueInterval = setInterval(this.checkForTrackChange, 5000);
  },
  beforeUnmount() {
    if (this.queueInterval) {
      clearInterval(this.queueInterval);
    }
  },
  methods: {
    getPlayingQueue() {
      const storedToken = this.spotifyStore.token;

      if (storedToken) {
        this.loading = true;
        fetchQueue(storedToken)
          .then((res) => {
            this.playingQueue = res;
            this.currentTrackId = res?.currently_playing?.id || null;
          })
          .catch((err) => {
            if (err.status === 401 && this.isVenueLogged) {
              // Token expired
              this.spotifyStore.token = null;
              this.$notify({
                title: this.$t("error"),
                text: this.$t("venuePlayingQueue.error.tokenExpired"),
                type: "error",
              });
            }
          })
          .finally(() => {
            this.loading = false;
          });
      } else {
        if (this.isVenueLogged) {
          // No token
          this.$notify({
            title: this.$t("error"),
            text: this.$t("venuePlayingQueue.error.noToken"),
            type: "error",
          });
        }
      }
    },
    checkForTrackChange() {
      const storedToken = this.spotifyStore.token;

      if (!storedToken) return;

      fetchQueue(storedToken)
        .then((res) => {
          const newTrackId = res?.currently_playing?.id;

          if (newTrackId && newTrackId !== this.currentTrackId) {
            this.playingQueue = res;
            this.currentTrackId = newTrackId;
          }
        })
        .catch((err) => {
          if (err.status === 401 && this.isVenueLogged) {
            // Token expired
            this.spotifyStore.token = null;
            this.$notify({
              title: this.$t("error"),
              text: this.$t("venuePlayingQueue.error.tokenExpired"),
              type: "error",
            });
          }
        });
    },
    addToQueue(track) {
      if (
        this.isClientLogged &&
        this.websiteStore.venue.tracksLimit !== null &&
        this.websiteStore.songsAdded >= this.websiteStore.venue.tracksLimit
      ) {
        this.$notify({
          title: this.$t("error"),
          text: this.$t("venuePlayingQueue.error.limitReached"),
          type: "error",
        });
        return;
      }

      const storedToken = this.spotifyStore.token;

      if (storedToken) {
        submitTrackToQueue(storedToken, track.uri)
          .then(() => {
            this.websiteStore.songsAdded++;
            this.$notify({
              title: this.$t("success"),
              text: this.$t("venuePlayingQueue.songAdded"),
              type: "success",
            });
            setTimeout(() => {
              this.getPlayingQueue();
            }, 300);
          })
          .catch((err) => {
            if (err.status === 401) {
              // Token expired
              this.spotifyStore.token = null;
              this.$notify({
                title: this.$t("error"),
                text: this.$t("venuePlayingQueue.error.tokenExpired"),
                type: "error",
              });
            }
          });
      }
    },
    async spotifyLogin() {
      const clientId = this.spotifyStore.clientId;
      const redirectUri = `${import.meta.env.VITE_APP_URL}/venue/${
        this.$route.params.id
      }`;

      const params = new URLSearchParams(window.location.search);
      const code = params.get("code");

      if (!code) {
        await redirectToAuthCodeFlow(clientId, redirectUri);
      } else {
        const verifier = localStorage.getItem("verifier");
        this.spotifyStore.token = await getAccessToken({
          clientId,
          code,
          redirectUri,
          verifier,
        });
      }
    },
  },
};
</script>
