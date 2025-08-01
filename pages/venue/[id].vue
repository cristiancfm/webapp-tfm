<template>
  <v-container v-if="!loading" class="py-10">
    <v-row class="text-center">
      <v-col>
        <h1>{{ venue.name }}</h1>
      </v-col>
    </v-row>
    <v-row justify="center">
      <v-col cols="12" md="4">
        <p class="mb-4">
          <v-img :src="venue.imageURL" width="100%" max-height="300" contain />
        </p>
        <p class="mb-4">{{ venue.description }}</p>
        <p class="mb-4">
          <v-row justify="center">
            <v-col v-if="venue.socialURLs.instagram" cols="auto">
              <a
                :href="venue.socialURLs.instagram"
                target="_blank"
                title="Instagram"
              >
                <v-img
                  src="/icons/socials/instagram.svg"
                  height="30px"
                  width="30px"
                />
              </a>
            </v-col>
            <v-col v-if="venue.socialURLs.facebook" cols="auto">
              <a
                :href="venue.socialURLs.facebook"
                target="_blank"
                title="Facebook"
              >
                <v-img
                  src="/icons/socials/facebook.svg"
                  height="30px"
                  width="30px"
                />
              </a>
            </v-col>
            <v-col v-if="venue.socialURLs.twitter" cols="auto">
              <a
                :href="venue.socialURLs.twitter"
                target="_blank"
                title="Twitter"
              >
                <v-img
                  src="/icons/socials/twitter.svg"
                  height="30px"
                  width="30px"
                />
              </a>
            </v-col>
            <v-col v-if="venue.socialURLs.tiktok" cols="auto">
              <a :href="venue.socialURLs.tiktok" target="_blank" title="TikTok">
                <v-img
                  src="/icons/socials/tiktok.svg"
                  height="30px"
                  width="30px"
                />
              </a>
            </v-col>
          </v-row>
        </p>
      </v-col>
      <v-col cols="12" md="6">
        <v-tabs v-model="tab">
          <v-tab value="playingQueue">{{ $t("venue.playingQueue") }}</v-tab>
          <v-tab v-if="isVenueLogged" value="settings">{{
            $t("venue.settings")
          }}</v-tab>
        </v-tabs>

        <v-tabs-window v-model="tab">
          <v-tabs-window-item value="playingQueue">
            <venue-playing-queue :venue="venue" />
          </v-tabs-window-item>
          <v-tabs-window-item value="settings">
            <venue-settings :venue="venue" />
          </v-tabs-window-item>
        </v-tabs-window>
      </v-col>
    </v-row>
  </v-container>
  <loading-page v-else />
</template>

<script>
import { mapStores } from "pinia";
import { useWebsiteStore } from "~/store/website.ts";
import { useSpotifyStore } from "~/store/spotify.ts";
import { getAccessToken } from "~/utils/spotifyAuth.js";
import VenuePlayingQueue from "~/components/venues/venuePlayingQueue.vue";
import VenueSettings from "~/components/venues/venueSettings.vue";

export default {
  components: { VenueSettings, VenuePlayingQueue },
  data() {
    return {
      loading: false,
      venue: {
        name: "",
        description: "",
        imageURL: "",
        socialURLs: [],
      },
      tab: null,
    };
  },
  computed: {
    ...mapStores(useWebsiteStore, useSpotifyStore),
    isClientLogged() {
      return this.websiteStore.loggedAuthority === "client";
    },
    isVenueLogged() {
      return this.websiteStore.loggedAuthority === "venue";
    },
  },
  mounted() {
    this.checkSpotifyCode();
    this.fetchVenue();
  },
  methods: {
    fetchVenue() {
      this.loading = true;
      const id = this.$route.params.id;
      $fetch(`${this.$config.public.apiBase}/venue/${id}`, {
        headers: {
          AccessCode: this.websiteStore.accessCode,
        },
      })
        .then((response) => {
          this.venue = response;
          this.websiteStore.venue = response;
        })
        .catch((err) => {
          this.$notify({
            title: this.$t("error"),
            text: this.$t(err.data.message),
            type: "error",
          });
          this.$router.push("/");
        })
        .finally(() => {
          this.loading = false;
        });
    },
    async checkSpotifyCode() {
      const code = this.$route.query.code;

      const clientId = this.spotifyStore.clientId;
      const id = this.$route.params.id;
      const redirectUri = `${import.meta.env.VITE_APP_URL}/venue/${id}`;
      const verifier = localStorage.getItem("verifier");

      if (code && verifier) {
        this.spotifyStore.token = await getAccessToken({
          clientId,
          code,
          redirectUri,
          verifier,
        });

        // Remove code from URL
        const newUrl = window.location.href.split("?")[0];
        window.history.replaceState({}, document.title, newUrl);
      }
    },
  },
};
</script>
