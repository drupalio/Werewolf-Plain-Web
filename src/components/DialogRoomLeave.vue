<template>
  <v-dialog
    persistent
    :fullscreen="$viewport.width < 450"
    v-model="dialog"
    max-width="600">
    <template v-slot:activator="{ on }">
      <v-btn 
        text
        v-on="on">
        <span class="header-text">{{ $t('DialogRoomLeave.leave') }}</span>
      </v-btn>
    </template>
    <v-card color="#36393F">
      <v-card-title>
        <span>{{ $t('DialogRoomLeave.title') }}</span>
      </v-card-title>
      <v-card-text>
        <v-container>
          <span>{{ $t('DialogRoomLeave.para1') }}</span>
        </v-container>
      </v-card-text>
      <v-card-actions>
        <div class="flex-grow-1"></div>
        <v-btn
          text
          @click="leaveRoom">
          <span>OK</span>
        </v-btn>
        <v-btn
          text
          @click="cancel">
          <span>{{ $t('DialogRoomLeave.cancel') }}</span>
        </v-btn>
      </v-card-actions>
    </v-card> 
  </v-dialog>
</template>

<script>
  import firebase from 'firebase/app'
  import 'firebase/auth'
  import 'firebase/firestore'

  export default {
    props: [
      'myself',
      'room',
    ],
    data() {
      return {
        dialog: false,
      }
    },
    methods: {
      leaveRoom() {
        // Remove the player's document from the collection
        var db = firebase.firestore()
        var docRef = db.collection('rooms').doc(this.$route.params.id)
        var promises0 = []
        var promises1 = []

        var sendMessage = 
          docRef.collection('messages').add({
            from: 'GM',
            timestamp: firebase.firestore.FieldValue.serverTimestamp(),
            body: this.$t('DialogRoomLeave.playerLeft', this.room.language, [this.myself.name]),
            gameName: 'GM',
            avatar: '',
            isFromGrave: false,
          })

        var removePlayer = docRef.collection('players').doc(firebase.auth().currentUser.uid).delete()

        promises0.push(sendMessage)
        promises0.push(removePlayer)

        Promise.all(promises0).then(() => {
            docRef.get().then((doc) => {
              // Remove the room document and its subcollection when the owner left
              if (doc.data().ownerId == firebase.auth().currentUser.uid) {
                var deleteRoom = docRef.delete()

                var deletePlayers = 
                  docRef.collection('players').get()
                    .then((querySnapShot) => {
                      Promise.all(querySnapShot.docs.map((doc) => {
                        doc.ref.delete()
                      }))
                    })

                var deleteMessages = 
                  docRef.collection('messages').get()
                    .then((querySnapShot) => {
                      Promise.all(querySnapShot.docs.map((doc) => {
                        doc.ref.delete()
                      }))
                    })

                promises1.push(deleteRoom)
                promises1.push(deletePlayers)
                promises1.push(deleteMessages)
              } else {
                var updateRoom = 
                  docRef.update({
                    numberOfParticipants: firebase.firestore.FieldValue.increment(-1),
                    ipList: doc.data().ipList.filter(data => data.uid !== firebase.auth().currentUser.uid),
                  })

                promises1.push(updateRoom)
              }

              Promise.all(promises1).then(() => {
                this.$router.push({
                  name: 'room-list',
                })
            })
          })
        })
      },
      cancel() {
        this.dialog = false
      }
    }
  }
</script>

<style scoped>
  span {
    color: #FFFFFF;
  }

  .header-text {
    color: #757575;
  }

  .v-card__text span {
    color: #DCDDDE;
  }
</style>
