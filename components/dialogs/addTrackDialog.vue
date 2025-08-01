<template>
  <v-dialog
    :model-value="dialog"
    max-width="500"
    @update:model-value="$emit('update:dialog', $event)"
  >
    <v-card>
      <v-card-title>
        <v-row align="center">
          <v-col>
            <span>{{ $t("venuePlayingQueue.addSong") }}</span>
          </v-col>
          <v-col cols="auto">
            <v-btn
              variant="flat"
              density="comfortable"
              icon="close"
              @click="$emit('update:dialog', false)"
            />
          </v-col>
        </v-row>
      </v-card-title>
      <v-divider />
      <v-card-text>
        <v-row v-if="websiteStore.venue.tracksLimit && isClientLogged">
          <v-col>
            <p class="text-body-2 text-center">
              {{
                $t("venuePlayingQueue.search.songsAdded", {
                  count: websiteStore.songsAdded,
                  total: websiteStore.venue.tracksLimit,
                })
              }}
            </p>
          </v-col>
        </v-row>
        <v-row>
          <v-col>
            <v-text-field
              v-model="searchText"
              :placeholder="$t('search')"
              append-inner-icon="search"
              hide-details="auto"
              clearable
              @update:model-value="searchTracks"
            />
          </v-col>
        </v-row>
        <v-row
          v-if="searchResult && searchText"
          style="max-height: calc(100vh - 200px); overflow-y: auto"
        >
          <v-col>
            <h4 class="ml-3">
              {{ $t("venuePlayingQueue.search.results") }}
            </h4>
            <v-list>
              <track-item
                v-for="(track, index) in searchResult.items"
                :key="index"
                :track="track"
                add-to-queue
                @update:add-to-queue="$emit('update:add-to-queue', $event)"
              />
            </v-list>
          </v-col>
        </v-row>
        <v-row v-else class="text-center">
          <v-col>
            <v-icon size="x-large" class="mb-2">music_note</v-icon>
            <p>{{ $t("venuePlayingQueue.search.placeholder") }}</p>
          </v-col>
        </v-row>
      </v-card-text>
    </v-card>
  </v-dialog>
</template>
<script>
import { mapStores } from "pinia";
import { useSpotifyStore } from "~/store/spotify.ts";
import { useWebsiteStore } from "~/store/website.ts";
import TrackItem from "~/components/venues/trackItem.vue";

const { fetchTracks } = useSpotify();

export default {
  components: { TrackItem },
  props: {
    dialog: {
      type: Boolean,
      default: false,
    },
  },
  emits: ["update:dialog", "update:add-to-queue"],
  data() {
    return {
      loading: false,
      searchText: "",
      searchResult: null,
    };
  },
  computed: {
    ...mapStores(useSpotifyStore, useWebsiteStore),
    isClientLogged() {
      return this.websiteStore.loggedAuthority === "client";
    },
  },
  methods: {
    searchTracks() {
      const storedToken = this.spotifyStore.token;

      this.loading = true;
      fetchTracks(storedToken, this.searchText)
        .then((response) => {
          this.searchResult = response.tracks;
        })
        .catch((error) => {
          console.error(error);
        })
        .finally(() => {
          this.loading = false;
        });
    },
  },
};
</script>
