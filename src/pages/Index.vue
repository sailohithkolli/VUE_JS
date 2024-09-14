<template>
  <q-page class="bg-grey-3 column">

    <div class="row q-pa-sm bg-primary">
      <q-input filled @keyup.enter="addUser" v-model="username" placeholder="Name" maxlength="12" dense bg-color="white" class="col" square></q-input>
      <q-input filled v-model="phoneNumber" placeholder="Phone Number" dense bg-color="white" class="col" square ></q-input>
      <q-input filled v-model="reason"  placeholder="Reason" dense bg-color="white" class="col" square>  
      <template v-slot:append>
          <q-btn round dense flat @click="addUser" icon="add" />
        </template>
      </q-input>
    </div>

    <div class="q-pa-md q-gutter-md">
      <q-list bordered separator class="bg-white rounded-borders">
        <q-item-label header>Current Waiting Queue</q-item-label>

        <q-item v-for="user in ActiveUsernames" :key="user.id" clickable v-ripple>
          <q-item-section avatar>
            <q-avatar>
              <img src="~assets/generic_avatar.png" />
            </q-avatar>
          </q-item-section>

          <q-item-section>
            <q-item-label lines="1">{{ user.name }}</q-item-label>
            <q-item-label caption lines="2">
              <div>
      <span>Phone: {{ user.phoneNumber }}</span> <!-- Display the phone number -->
      <br />
      <span>Reason: {{ user.reason }}</span> <!-- You may also want to display the reason if available -->
    </div>
            </q-item-label>
          </q-item-section>

          <q-item-section side top>
            <DateDiff :date="new Date(user.timestamp)" />
          </q-item-section>
        
          <q-item-section side top>
            <q-btn round dense flat @click="cancelUser(user.id)" label="Cancel Appointment" />
          </q-item-section>
        </q-item>
      </q-list>
    </div>
  </q-page>
</template>

<script>
import DateDiff from 'components/DateDiff.vue';
import { date } from 'quasar';
import db from 'boot/firebase';

export default {
  components: {
    DateDiff,
  },
  data() {
    return {
      username: '',
      usernames: [],
      reason : '',
      phoneNumber : '',
    };
  },
  methods: {
    async initialize() {
      await db
        .collection('waiting_details')
        .orderBy('timestamp', 'asc')
        .onSnapshot((snapshot) => {
          snapshot.docChanges().forEach((change) => {
            if (change.type === 'added') {
              // ADD
              const source = change.doc.metadata.hasPendingWrites ? 'Local' : 'Server';

              if (source === 'Server') {
                let user = change.doc.data();
                user.id = change.doc.id;
                this.usernames.push(user);
              }
            }

            if (change.type === 'modified') {
              // UPDATE
              const index = this.usernames.findIndex((item) => item.id == change.doc.id);
              let user = change.doc.data();
              user.id = change.doc.id;
              this.usernames.splice(index, 1, user);
            }

            if (change.type === 'removed') {
              // REMOVE
              const index = this.usernames.findIndex((item) => item.id == change.doc.id);
              if (index >= 0) {
                this.usernames.splice(index, 1);
              }
            }
          });
        });
    },
    async addUser() {
      if (!this.username || this.username.trim() === '' || !this.phoneNumber || this.phoneNumber.trim() === '') {
        window.alert('Username and Phone Number cannot be empty.');
        return;
      }
      let newuser = {
        name: this.username,
        phoneNumber:this.phoneNumber,
        reason:this.reason,
        active: true,
        timestamp: Date.now(),
      };
      this.usernames.push(newuser);
      await db
        .collection('waiting_details')
        .add(newuser)
        .then()
        .catch((error) => {
          console.error('Error adding user:', error);
        });
      this.username = '';
      this.phoneNumber = '';
      this.reason = '';
    },
    async cancelUser(userId) {
      await db.collection('waiting_details').doc(userId).update({
        active: false,
      });
    },
    computeTimeDiff(date1) {
      let date2 = Date.now();
      return date.getDateDiff(date2, date1, 'minutes');
    },
  },
  computed: {
    ActiveUsernames: function () {
      return this.usernames.filter((status) => status.active == true);
    },
  },
  created() {
    this.initialize();
  },
};
</script>
