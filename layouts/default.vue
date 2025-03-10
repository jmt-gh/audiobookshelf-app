<template>
  <div class="w-full layout-wrapper bg-bg text-white">
    <app-appbar />
    <div id="content" class="overflow-hidden relative" :class="playerIsOpen ? 'playerOpen' : ''">
      <Nuxt />
    </div>
    <app-audio-player-container ref="streamContainer" />
    <modals-libraries-modal />
    <app-side-drawer />
    <readers-reader />
  </div>
</template>

<script>
export default {
  data() {
    return {
      attemptingConnection: false,
      inittingLibraries: false,
      hasMounted: false,
      disconnectTime: 0
    }
  },
  watch: {
    networkConnected: {
      handler(newVal, oldVal) {
        if (!this.hasMounted) {
          // watcher runs before mount, handling libraries/connection should be handled in mount
          return
        }
        if (newVal) {
          console.log(`[default] network connected changed ${oldVal} -> ${newVal}`)
          if (!this.user) {
            this.attemptConnection()
          } else if (!this.currentLibraryId) {
            this.initLibraries()
          } else {
            var timeSinceDisconnect = Date.now() - this.disconnectTime
            if (timeSinceDisconnect > 5000) {
              console.log('Time since disconnect was', timeSinceDisconnect, 'sync with server')
              setTimeout(() => {
                // TODO: Some issue here
                this.syncLocalMediaProgress()
              }, 4000)
            }
          }
        } else {
          console.log(`[default] lost network connection`)
          this.disconnectTime = Date.now()
        }
      }
    }
  },
  computed: {
    playerIsOpen() {
      return this.$store.state.playerLibraryItemId
    },
    routeName() {
      return this.$route.name
    },
    networkConnected() {
      return this.$store.state.networkConnected
    },
    user() {
      return this.$store.state.user.user
    },
    currentLibraryId() {
      return this.$store.state.libraries.currentLibraryId
    },
    isSocketConnected() {
      return this.$store.state.socketConnected
    }
  },
  methods: {
    initialStream(stream) {
      if (this.$refs.streamContainer && this.$refs.streamContainer.audioPlayerReady) {
        this.$refs.streamContainer.streamOpen(stream)
      }
    },
    async loadSavedSettings() {
      var userSavedServerSettings = await this.$localStore.getServerSettings()
      if (userSavedServerSettings) {
        this.$store.commit('setServerSettings', userSavedServerSettings)
      }

      var userSavedSettings = await this.$localStore.getUserSettings()
      if (userSavedSettings) {
        this.$store.commit('user/setSettings', userSavedSettings)
      }
    },
    async attemptConnection() {
      console.warn('[default] attemptConnection')
      if (!this.networkConnected) {
        console.warn('[default] No network connection')
        return
      }
      if (this.attemptingConnection) {
        return
      }
      this.attemptingConnection = true

      var deviceData = await this.$db.getDeviceData()
      var serverConfig = null
      if (deviceData && deviceData.lastServerConnectionConfigId && deviceData.serverConnectionConfigs.length) {
        serverConfig = deviceData.serverConnectionConfigs.find((scc) => scc.id == deviceData.lastServerConnectionConfigId)
      }
      if (!serverConfig) {
        // No last server config set
        this.attemptingConnection = false
        return
      }

      console.log(`[default] Got server config, attempt authorize ${serverConfig.address}`)

      var authRes = await this.$axios.$post(`${serverConfig.address}/api/authorize`, null, { headers: { Authorization: `Bearer ${serverConfig.token}` } }).catch((error) => {
        console.error('[Server] Server auth failed', error)
        var errorMsg = error.response ? error.response.data || 'Unknown Error' : 'Unknown Error'
        this.error = errorMsg
        return false
      })
      if (!authRes) {
        this.attemptingConnection = false
        return
      }

      const { user, userDefaultLibraryId } = authRes

      // Set library - Use last library if set and available fallback to default user library
      var lastLibraryId = await this.$localStore.getLastLibraryId()
      if (lastLibraryId && (!user.librariesAccessible.length || user.librariesAccessible.includes(lastLibraryId))) {
        this.$store.commit('libraries/setCurrentLibrary', lastLibraryId)
      } else if (userDefaultLibraryId) {
        this.$store.commit('libraries/setCurrentLibrary', userDefaultLibraryId)
      }
      var serverConnectionConfig = await this.$db.setServerConnectionConfig(serverConfig)

      this.$store.commit('user/setUser', user)
      this.$store.commit('user/setServerConnectionConfig', serverConnectionConfig)

      this.$socket.connect(serverConnectionConfig.address, serverConnectionConfig.token)

      console.log('[default] Successful connection on last saved connection config', JSON.stringify(serverConnectionConfig))
      await this.initLibraries()
      this.attemptingConnection = false
    },
    itemRemoved(libraryItem) {
      if (this.$route.name.startsWith('item')) {
        if (this.$route.params.id === libraryItem.id) {
          this.$router.replace(`/bookshelf`)
        }
      }
    },
    userLoggedOut() {
      // Only cancels stream if streamining not playing downloaded
      this.$eventBus.$emit('close-stream')
    },
    socketConnectionUpdate(isConnected) {
      console.log('Socket connection update', isConnected)
    },
    socketConnectionFailed(err) {
      this.$toast.error('Socket connection error: ' + err.message)
    },
    socketInit(data) {
      console.log('Socket init', data)
    },
    async initLibraries() {
      if (this.inittingLibraries) {
        return
      }
      this.inittingLibraries = true
      await this.$store.dispatch('libraries/load')
      console.log(`[default] initLibraries loaded ${this.currentLibraryId}`)
      await this.$store.dispatch('libraries/fetch', this.currentLibraryId)
      this.$eventBus.$emit('library-changed')
      this.inittingLibraries = false
    },
    async syncLocalMediaProgress() {
      if (!this.user) {
        console.log('[default] No need to sync local media progress - not connected to server')
        return
      }

      console.log('[default] Calling syncLocalMediaProgress')
      var response = await this.$db.syncLocalMediaProgressWithServer()
      if (!response) {
        if (this.$platform != 'web') this.$toast.error('Failed to sync local media with server')
        return
      }
      const { numLocalMediaProgressForServer, numServerProgressUpdates, numLocalProgressUpdates } = response
      if (numLocalMediaProgressForServer > 0) {
        if (numServerProgressUpdates > 0 || numLocalProgressUpdates > 0) {
          console.log(`[default] ${numServerProgressUpdates} Server progress updates | ${numLocalProgressUpdates} Local progress updates`)
        } else {
          console.log('[default] syncLocalMediaProgress No updates were necessary')
        }
      } else {
        console.log('[default] syncLocalMediaProgress No local media progress to sync')
      }
    },
    async userUpdated(user) {
      if (this.user && this.user.id == user.id) {
        this.$store.commit('user/setUser', user)
      }
    },
    async userMediaProgressUpdated(prog) {
      console.log(`[default] userMediaProgressUpdate checking for local media progress ${prog.id}`)

      // Update local media progress if exists
      var localProg = this.$store.getters['globals/getLocalMediaProgressByServerItemId'](prog.libraryItemId, prog.episodeId)
      var newLocalMediaProgress = null
      if (localProg && localProg.lastUpdate < prog.lastUpdate) {
        // Server progress is more up-to-date
        console.log(`[default] syncing progress from server with local item for "${prog.libraryItemId}" ${prog.episodeId ? `episode ${prog.episodeId}` : ''}`)
        const payload = {
          localMediaProgressId: localProg.id,
          mediaProgress: prog
        }
        newLocalMediaProgress = await this.$db.syncServerMediaProgressWithLocalMediaProgress(payload)
      } else {
        // Check if local library item exists
        var localLibraryItem = await this.$db.getLocalLibraryItemByLLId(prog.libraryItemId)
        if (localLibraryItem) {
          if (prog.episodeId) {
            // If episode check if local episode exists
            var lliEpisodes = localLibraryItem.media.episodes || []
            var localEpisode = lliEpisodes.find((ep) => ep.serverEpisodeId === prog.episodeId)
            if (localEpisode) {
              // Add new local media progress
              const payload = {
                localLibraryItemId: localLibraryItem.id,
                localEpisodeId: localEpisode.id,
                mediaProgress: prog
              }
              newLocalMediaProgress = await this.$db.syncServerMediaProgressWithLocalMediaProgress(payload)
            }
          } else {
            // Add new local media progress
            const payload = {
              localLibraryItemId: localLibraryItem.id,
              mediaProgress: prog
            }
            newLocalMediaProgress = await this.$db.syncServerMediaProgressWithLocalMediaProgress(payload)
          }
        } else {
          console.log(`[default] userMediaProgressUpdate no local media progress or lli found for this server item ${prog.id}`)
        }
      }

      if (newLocalMediaProgress && newLocalMediaProgress.id) {
        console.log(`[default] local media progress updated for ${newLocalMediaProgress.id}`)
        this.$store.commit('globals/updateLocalMediaProgress', newLocalMediaProgress)
      }
    }
  },
  async mounted() {
    this.$socket.on('connection-update', this.socketConnectionUpdate)
    this.$socket.on('initialized', this.socketInit)
    this.$socket.on('user_updated', this.userUpdated)
    this.$socket.on('user_media_progress_updated', this.userMediaProgressUpdated)

    if (this.$store.state.isFirstLoad) {
      this.$store.commit('setIsFirstLoad', false)
      await this.$store.dispatch('setupNetworkListener')

      if (this.$store.state.user.serverConnectionConfig) {
        console.log(`[default] server connection config set - call init libraries`)
        await this.initLibraries()
      } else {
        console.log(`[default] no server connection config - call attempt connection`)
        await this.attemptConnection()
      }

      console.log(`[default] finished connection attempt or already connected ${!!this.user}`)
      await this.syncLocalMediaProgress()
      this.$store.dispatch('globals/loadLocalMediaProgress')
      this.loadSavedSettings()
      this.hasMounted = true
    }
  },
  beforeDestroy() {
    this.$socket.off('connection-update', this.socketConnectionUpdate)
    this.$socket.off('initialized', this.socketInit)
    this.$socket.off('user_updated', this.userUpdated)
    this.$socket.off('user_media_progress_updated', this.userMediaProgressUpdated)
  }
}
</script>
