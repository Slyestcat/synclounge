<template>
  <v-container class="fill-height">
    <v-row
      align="center"
      justify="center"
    >
      <v-col>
        <v-card
          class="mx-auto"
          max-width="550"
          :loading="loading"
        >
          <v-card-title>
            <v-img
              contain
              src="@/assets/images/logos/logo-long-light.png"
            />
          </v-card-title>

          <v-alert
            v-if="error"
            type="error"
          >
            {{ error }}
          </v-alert>

          <!-- Fork addition: surface the preselected media so the joiner
               knows what they're about to watch. -->
          <v-alert
            v-if="preselectTitle"
            type="info"
            text
            class="mx-4"
          >
            Joining a watch party for <strong>{{ preselectTitle }}</strong>
          </v-alert>

          <v-expansion-panels
            multiple
          >
            <v-expansion-panel
              :readonly="GET_CONFIG.force_slplayer"
            >
              <v-expansion-panel-header>
                Player: {{ GET_CHOSEN_CLIENT.name }}
              </v-expansion-panel-header>

              <v-expansion-panel-content>
                <PlexClientPicker
                  @loading-change="loading = $event"
                  @client-connectable-change="clientConnectable = $event"
                />
              </v-expansion-panel-content>
            </v-expansion-panel>
          </v-expansion-panels>

          <v-card-actions class="mt-2">
            <v-btn
              color="primary"
              :disabled="!clientConnectable"
              @click="joinInvite"
            >
              Join Invite
            </v-btn>
          </v-card-actions>
        </v-card>
      </v-col>
    </v-row>
  </v-container>
</template>

<script>
import { mapActions, mapGetters } from 'vuex';
import redirection from '@/mixins/redirection';
import { slPlayerClientId } from '@/player/constants';
import linkWithRoom from '@/mixins/linkwithroom';

export default {
  name: 'RoomJoin',

  components: {
    PlexClientPicker: () => import('@/components/PlexClientPicker.vue'),
  },

  mixins: [
    redirection,
    linkWithRoom,
  ],

  props: {
    room: {
      type: String,
      required: true,
    },

    server: {
      type: String,
      default: '',
    },
  },

  data: () => ({
    loading: false,
    error: null,

    // Default true because default client is slplayer
    clientConnectable: true,
  }),

  computed: {
    ...mapGetters([
      'GET_CONFIG',
    ]),

    ...mapGetters('plexclients', [
      'GET_CHOSEN_CLIENT_ID',
      'GET_ACTIVE_MEDIA_METADATA',
      'GET_CHOSEN_CLIENT',
    ]),

    // --- Fork addition: media preselect from URL query string ---
    //
    // When the URL looks like
    //   /#/join/abc123?machineIdentifier=XXX&ratingKey=YYY&title=The+Matrix
    // we route the user straight to the PlexMedia page for that title
    // after they join, instead of dropping them on the generic PlexHome.
    // The Discord bot (or whoever generated the invite) is responsible for
    // putting these params on the URL.
    preselectMachineIdentifier() {
      const v = this.$route.query.machineIdentifier;
      return typeof v === 'string' && v.length > 0 ? v : null;
    },
    preselectRatingKey() {
      const v = this.$route.query.ratingKey;
      return typeof v === 'string' && v.length > 0 ? v : null;
    },
    preselectTitle() {
      const v = this.$route.query.title;
      return typeof v === 'string' && v.length > 0 ? v : null;
    },
    hasPreselect() {
      return !!(this.preselectMachineIdentifier && this.preselectRatingKey);
    },
  },

  async created() {
    await this.DISCONNECT_IF_CONNECTED();
  },

  methods: {
    ...mapActions('synclounge', [
      'SET_AND_CONNECT_AND_JOIN_ROOM',
      'DISCONNECT_IF_CONNECTED',
    ]),

    async joinInvite() {
      this.error = null;
      this.loading = true;

      try {
        await this.SET_AND_CONNECT_AND_JOIN_ROOM({
          server: this.server,
          room: this.room,
        });

        if (this.$route.name === 'RoomJoin') {
          // Fork addition: if invite carried a preselected media item,
          // route there first. Falls back to default behavior if not.
          if (this.hasPreselect) {
            this.$router.push(this.linkWithRoom({
              name: 'PlexMedia',
              params: {
                machineIdentifier: this.preselectMachineIdentifier,
                ratingKey: this.preselectRatingKey,
              },
            }));
          } else if (
            this.GET_CHOSEN_CLIENT_ID === slPlayerClientId
            || !this.GET_ACTIVE_MEDIA_METADATA
          ) {
            this.$router.push(this.linkWithRoom({ name: 'PlexHome' }));
          } else {
            this.redirectToMediaPage();
          }
        }
      } catch (e) {
        this.DISCONNECT_IF_CONNECTED();
        console.error(e);
        this.error = e.message;
      }

      this.loading = false;
    },
  },
};
</script>
