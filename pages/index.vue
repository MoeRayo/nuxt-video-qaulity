<template>
 <div>
    <div class="br3 bg-white ba dark-gray b--black-10 shadow-3 w-100 w-60-m w-30-l mw6 center mt5 ph4 pv4 tc">
      <button class="f4 link dim br3 pv2 ph2 mb2 dib white bg-navy ba b--navy pointer mt3 mt0-l inline-flex items-center" @click="openUploadModal" v-if="isInitial">
        Upload video
      </button>
      <form v-if="url.length >0" class="tc ma" @click.prevent="enhanceVideo">
        <input type="text" :value="url" class="pa3 db w-100 br3 b--navy ba f4" disabled="true">
        <button class="f4 link dim br3 pv2 ph2 mb2 dib white bg-navy ba b--navy pointer mt3 inline-flex items-center">Enhance Video</button>
      </form>
    </div>  
 </div>
</template>
<script>
export default {
  data(){
    return {
      url: '',
      isInitial: true,
      cloudName: 'moerayo',
      uploadPreset: 'lan9vkdw',
      apiToken: '',
      jobId: '',
      formData: null,
    }
  },
  methods: {
    openUploadModal() {
      window.cloudinary.openUploadWidget(
        { cloudName: this.cloudName,
          uploadPreset: this.uploadPreset
        },
        (error, result) => {
          if (!error && result && result.event === "success") {
            console.log('Done uploading..: ', result.info);
            this.isInitial = false;
            this.url = result.info.url;
            console.log(this.url)
          }
        }
      ).open();
    },
    async enhanceVideo(){
      const auth = Buffer.from(`${process.env.APP_KEY}:${process.env.APP_SECRET}`).toString('base64');
      const response = await fetch('https://api.dolby.io/v1/auth/token', {
        method: 'POST',
        body: new URLSearchParams({
          grant_type: 'client_credentials',
          expires_in: 7200
        }),
        headers: {
          "Accept": 'application/json',
          'Content-Type': 'application/x-www-form-urlencoded',
          'Authorization': `Basic ${auth}`
        }
      });
      const json = await response.json();
      this.apiToken = json.access_token;

      const enhanceRequest = {
        method: "POST",
        url: 'https://api.dolby.com/media/enhance',
        headers: {
          "Authorization": `Bearer ${this.apiToken}`,
          "Content-Type": "application/json",
          "Accept": "application/json"
          },
        data:{
        input: this.url,
        output: 'dlb://out/video-enhanced.mp4'
        }
      }

      const enhancementCheck = {
        method: "GET",
        headers: {
          "Authorization": `Bearer ${this.apiToken}`,
          "Content-Type": "application/json",
          Accept: "application/json"
        },
      };

      const mediaDownload = {
        method: 'GET',
        url: 'https://api.dolby.com/media/output',
        headers: {
          'Authorization': `Bearer ${this.apiToken}`,
          'Content-Type': 'application/json',
          "Accept": "application/octet-stream"
        },
        responseType: 'stream',
        params: {
          url: 'dlb://out/video-enhanced.mp4'
        }
      };

      function wait(ms = 2000) {
        return new Promise((resolve) => {
          setTimeout(resolve, ms);
        });
      }

      let state = null;
  
      this.$axios(enhanceRequest)
      .then(async (response) =>{
        this.jobId = response.data.job_id
        while(state !== "Success"){
          await wait()
          this.$axios(`https://api.dolby.com/media/enhance?job_id=${this.jobId}`,enhancementCheck)
          .then((res)=> {
            state = res.data.status
            if(res.data.status === "Success"){
              this.$axios(mediaDownload)
              .then((response) => {
                response
              })
              .catch((err) => {
                if (err.request.responseURL.match(/mp4/)) {
                  let retry = err.request.responseURL
                  if (retry){
                    console.log("Now Downloading audio");
                    this.$axios.get(retry).then((response) => {
                      if (response.status === 200) {
                        //upload enhanced video to cloudinary
                        console.log("Now Uploading to cloudinary");
                        this.formData = new FormData();
                        this.formData.append("file", response.config.url);
                        this.formData.append("upload_preset", this.uploadPreset);
                        const cloudinaryUploadURL = `https://api.cloudinary.com/v1_1/${this.cloudName}/upload`;
                        const requestObj = {
                          url: cloudinaryUploadURL,
                          method: "POST",
                          data: this.formData,
                        };
                        //uploading to cloudinary
                        this.$axios(requestObj)
                        .then(response => {
                          response
                        })
                        .catch((err) => console.log(err));
                      }
                    }).catch((e) => {
                      e
                    })
                  }
                } else {
                  console.log("error")
                }
              });
            } 
          }).catch((err) => err)
        }
      }).catch((err) => err)
    }
  }
}
</script>
