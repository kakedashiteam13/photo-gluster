<template>
  <div id="app">
    <div style="display: flex; flex-direction: column;">
      <!-- Upload Interface -->
      <div id="upload">
        <div v-if="this.$root.$data.loading === false">
          <h1>Post Here!</h1>
          <h4>share your memories.</h4>

          <!-- Form for file choose, caption text and submission -->
          <form class="margin-sm" @submit.stop.prevent="handleSubmit">
            <div class="border-style">
              <b-form-file plain @change="captureFile"/>
            </div>
            <input type="text" v-model="userName">
            <input type="text" v-model="shutterSpeedValue">
            <input type="text" v-model="fNumber">
            <input type="text" v-model="iso">
            <b-button class="margin-xs" variant="secondary" @click="handleOk">Upload</b-button>
          </form>
        </div>
        <div v-if="this.$root.$data.loading === true" style="margin-top: 10%; margin-bottom: 5%">
          <img class="upload-load" src="https://media.giphy.com/media/2A6xoqXc9qML9gzBUE/giphy.gif">
        </div>
      </div>

      <!-- Posts Interface -->
      <ul class="home-list">
        <section class="photo-contents">
          <li v-for="item in this.$root.$data.currentPosts" :key="item.key" :item="item">
            <!-- Card UI for post's image & caption text -->
            <b-card border-variant="secondary" :img-src="item.src">
              <p>
                <span class="photo-card__meta-text--value">F{{ item.fNumber }}</span>
                <span class="photo-card__meta-text--title">SS{{ item.shutterSpeedValue }}</span>
                <span class="photo-card__meta-text--title">ISO{{ item.iso }}</span>
                {{ item.userName }}
              </p>
            </b-card>
          </li>
        </section>
      </ul>
    </div>
  </div>
</template>

<script>
import ipfs from "./contracts/ipfs";
import contract from "./contracts/contractInstance";

export default {
  name: "App",
  // data variables
  data() {
    return {
      buffer: "",
      userName: "",
      shutterSpeedValue: "",
      fNumber: "",
      iso: ""
    };
  },
  methods: {
    /* used to catch chosen image &
     * convert it to ArrayBuffer.
     */
    captureFile(file) {
      const reader = new FileReader();
      if (typeof file !== "undefined") {
        reader.readAsArrayBuffer(file.target.files[0]);
        reader.onloadend = async () => {
          this.buffer = await this.convertToBuffer(reader.result);
        };
      } else this.buffer = "";
    },
    /**
     * converts ArrayBuffer to
     * Buffer for IPFS upload.
     */
    async convertToBuffer(reader) {
      return Buffer.from(reader);
    },
    /**
     * submits buffered image & text to IPFS
     * and retrieves the hashes, then store
     * it in the Contract via sendHash().
     */
    onSubmit() {
      this.$root.loading = true;
      let imgHash;
      let userNameHash;
      let shutterSpeedValueHash;
      let fNumberHash;
      let isoHash;
      ipfs
        .add(this.buffer)
        .then(hashedImg => {
          imgHash = hashedImg[0].hash;
          console.log("imgHash: " + imgHash);
          return this.convertToBuffer(this.userName);
        })
        .then(bufferDesc =>
          ipfs.add(bufferDesc).then(hashedName => {
            userNameHash = hashedName[0].hash;
            return this.convertToBuffer(this.shutterSpeedValue);
          })
        )
        .then(bufferDesc =>
          ipfs.add(bufferDesc).then(hashedshutterSpeedValue => {
            shutterSpeedValueHash = hashedshutterSpeedValue[0].hash;
            return this.convertToBuffer(this.fNumber);
          })
        )
        .then(bufferDesc =>
          ipfs.add(bufferDesc).then(hashedfNumber => {
            fNumberHash = hashedfNumber[0].hash;
            return this.convertToBuffer(this.iso);
          })
        )
        .then(bufferDesc =>
          ipfs.add(bufferDesc).then(hashedIso => {
            isoHash = hashedIso[0].hash;
          })
        )
        .then(() => {
          this.$root.contract.methods
            .sendHash(
              imgHash,
              userNameHash,
              shutterSpeedValueHash,
              fNumberHash,
              isoHash
            )
            .send(
              { from: this.$root.currentAccount },
              (error, transactionHash) => {
                if (typeof transactionHash !== "undefined") {
                  console.log("Storing on Ethereum...");
                  this.$root.contract.once(
                    "NewPost",
                    { from: this.$root.currentAccount },
                    () => {
                      this.$root.getPosts();
                      console.log("Operation Finished! Refetching...");
                    }
                  );
                } else this.$root.loading = false;
              }
            );
        });
    },
    /**
     * validates if image & captions
     * are filled before submission.
     */
    handleOk() {
      if (!this.buffer || !this.userName) {
        alert("Please fill in the information.");
      } else {
        this.onSubmit();
      }
    }
  }
};
</script>

<style>
#app {
  font-family: "Avenir", Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  display: flex;
  justify-content: center;
  color: #2c3e50;
  margin-top: 3%;
}
.home-load {
  width: 50px;
  height: 50px;
}
.card img {
  object-fit: cover;
  width: 240px;
}
.card {
  text-align: left;
  margin: 20px;
  border: none;
}
.home-list {
  padding: 0;
  list-style: none;
}
.home-card-text {
  text-align: justify;
  margin-top: 10px;
}
#upload {
  font-family: "Avenir", Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-bottom: 5%;
  width: 500px;
}
.upload-load {
  width: 50px;
  height: 50px;
}
.margin-xs {
  margin-top: 3%;
}
.margin-sm {
  margin-top: 7%;
}
.border-style {
  border: 1px solid #ced4da;
}
</style>