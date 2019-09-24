<template>
  <!-- <v-container>
    <h1 class="fin" align="center">COMING SOON...</h1>
  </v-container>-->

  <v-container>
    <v-timeline align-top :dense="$vuetify.breakpoint.smAndDown">
      <v-timeline-item v-for="(item, i) in fixtures" :key="i" color="#91ff00" fill-dot>
        <template v-slot:icon>
          <span>
            <v-icon v-if="item.flag">mdi-checkbox-marked-circle-outline</v-icon>
          </span>
        </template>

        <v-card dark class="mx-auto">
          <v-card-title class="title fin">Match #{{ item.mnumber }}</v-card-title>
          <v-card-text class="lexend">
            <p>Date: {{ item.date }}</p>
            <p style="font-size: 20px">{{item.team1}} vs {{item.team2}}</p>
          </v-card-text>

          <v-card-actions>
            <p v-if="item.flag" class="title fin">{{ item.wonby }} won the match</p>
            <div class="flex-grow-1"></div>
            <v-btn v-if="item.flag" @click="seeScore(item)">See Scorecard</v-btn>
            <v-btn v-if="item.flag" @click="updateResult(item)">Update Result</v-btn>
          </v-card-actions>
        </v-card>
      </v-timeline-item>
    </v-timeline>
    <v-dialog v-model="dialogUpdate" persistent max-width="600px">
      <v-card>
        <v-card-title>
          <span class="headline align-center">Update Result</span>
        </v-card-title>
        <v-card-text>
          <v-container>
            <v-select
              v-model="teamwon"
              :items="[this.fixture.team1, this.fixture.team2]"
              dark
              label="Won By"
            ></v-select>
            <v-row>
              <v-col cols="12" sm="6" md="4">
                <v-text-field v-model="rwon" type="number" label="Rounds Won" required></v-text-field>
              </v-col>
              <v-col cols="12" sm="6" md="4">
                <v-text-field v-model="rlost" type="number" label="Rounds Lost" required></v-text-field>
              </v-col>
              <v-col cols="12" sm="6" md="4">
                <v-btn color="black" @click="$refs.inputUpload.click()">Choose Image</v-btn>
                
                <input
                  type="file"
                  ref="inputUpload"
                  v-show="false"
                  class="btn"
                  id="imgInp"
                  @change="onFileChange"
                />
              </v-col>
            </v-row>
          </v-container>
        </v-card-text>
        <v-card-actions>
          <v-spacer></v-spacer>
          <v-btn text @click="dialogUpdate = false">Close</v-btn>
          <v-btn text @click="updateRes">Update</v-btn>
        </v-card-actions>
      </v-card>
    </v-dialog>
    <v-dialog v-model="dialogScore">
      <v-card >
        <v-card-text>
          <img width="100%" height="auto" :src="srURL" alt="Scorecard" />
        </v-card-text>

        <v-card-actions>
          <v-btn text @click="dialogScore = false">Close</v-btn>
        </v-card-actions>
      </v-card>
    </v-dialog>
  </v-container>
</template>

<script>
import { mapState } from "vuex";
const fb = require("../firebaseConfig");
export default {
  name: "fixtures",
  data() {
    return {
      fixtures: [],
      file: null,
      dialogUpdate: false,
      dialogScore: false,
      fixture: {},
      teamwon: "",
      rwon: null,
      rlost: null,
      srURL: ""
    };
  },
  computed: {
    ...mapState(["currentUser"])
  },
  created() {
    fb.fixturesCollection
      .orderBy("mnumber")
      .get()
      .then(res => {
        res.forEach(doc => {
          this.fixtures.push(doc.data());
        });
      });
  },

  methods: {
    seeScore(item) {
      this.dialogScore = true;
      this.srURL = item.scURL;
    },
    onFileChange(e) {
      const file = e.target.files[0];
      this.file = file;
      this.url = URL.createObjectURL(file);
    },
    updateResult(data) {
      this.fixture = data;
      this.rwon = null;
      this.rlost = null;
      this.teamwon = "";
      this.dialogUpdate = true;
    },
    updateRes() {
      if (this.teamwon && this.rwon && this.rlost) {
        this.dialogUpdate = false;
        this.fixture["flag"] = true;
        this.fixture["wonby"] = this.teamwon;
        if (this.teamwon == this.fixture["team1"]) {
          this.fixture["lostby"] = this.fixture["team2"];
        } else {
          this.fixture["lostby"] = this.fixture["team1"];
        }

        fb.storage
          .ref()
          .child("scorecards/" + this.fixture.mnumber)
          .put(this.file)
          .then(r => {
            r.ref.getDownloadURL().then(link => {
              fb.fixturesCollection.doc(this.fixture["id"]).update({
                scURL: link
              });
            });
          });
        fb.fixturesCollection.doc(this.fixture["id"]).update({
          wonby: this.teamwon,
          flag: true
        });

        fb.teamsCollection
          .where("name", "==", this.fixture["team1"])
          .get()
          .then(snap => {
            snap.forEach(doc => {
              this.fixture["id1"] = doc.id;
              this.fixture["match1"] = doc.data().mplayed + 1;
              fb.teamsCollection.doc(this.fixture["id1"]).update({
                mplayed: this.fixture["match1"]
              });
            });
          });

        fb.teamsCollection
          .where("name", "==", this.fixture["team2"])
          .get()
          .then(snap => {
            snap.forEach(doc => {
              this.fixture["id2"] = doc.id;
              this.fixture["match2"] = doc.data().mplayed + 1;
              fb.teamsCollection.doc(this.fixture["id2"]).update({
                mplayed: this.fixture["match2"]
              });
            });
          });

        fb.teamsCollection
          .where("name", "==", this.fixture["wonby"])
          .get()
          .then(snap => {
            snap.forEach(doc => {
              this.fixture["idwon"] = doc.id;
              this.fixture["mwonwon"] = doc.data().mwon + 1;
              this.fixture["rwonwon"] = doc.data().rwon + parseInt(this.rwon);
              this.fixture["rlostwon"] =
                doc.data().rlost + parseInt(this.rlost);
              fb.teamsCollection.doc(this.fixture["idwon"]).update({
                mwon: this.fixture["mwonwon"],
                rwon: this.fixture["rwonwon"],
                rlost: this.fixture["rlostwon"]
              });
            });
          });

        fb.teamsCollection
          .where("name", "==", this.fixture["lostby"])
          .get()
          .then(snap => {
            snap.forEach(doc => {
              this.fixture["idlost"] = doc.id;
              this.fixture["rwonlost"] = doc.data().rwon + parseInt(this.rlost);
              this.fixture["rlostlost"] =
                doc.data().rlost + parseInt(this.rwon);
              fb.teamsCollection.doc(this.fixture["idlost"]).update({
                rwon: this.fixture["rwonlost"],
                rlost: this.fixture["rlostlost"]
              });
            });
          });
      }
    }
  }
};
</script>
