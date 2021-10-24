<template>
  <div class="container">
    <h1 class="text-center">Laravel Video Chat</h1>
    <div class="video-container" ref="video-container">
      <video class="video-here" ref="video-here" autoplay muted></video>
      <video class="video-there" ref="video-there" autoplay></video>
      <div class="text-right" v-for="(name,userId) in others" :key="userId">
        <button @click="startVideoChat(userId)" v-text="`Talk with ${name}`"/>
      </div>
    </div>
  </div>
</template>
<script>
import Pusher from 'pusher-js';
import Peer from 'simple-peer';
export default {
  props: ['user', 'others', 'pusherKey', 'pusherCluster'],
  data() {
    return {
      channel: null,
      stream: null,
      peers: {}
    }
  },
  mounted() {
    this.setupVideoChat();
  },
  methods: {
    startVideoChat(userId) {
      this.getPeer(userId, true);
    },

    // Creates a new peer using userId. Finds and returns a peer via a userId in the end.
    getPeer(userId, initiator) {
      if(this.peers[userId] === undefined) {
        let peer = new Peer({
          initiator,
          stream: this.stream,
          trickle: false
        });
        peer.on('signal', (data) => {
          this.channel.trigger(`client-signal-${userId}`, {
            userId: this.user.id,
            data: data
          });
        })
        .on('stream', (stream) => {
          const videoThere = this.$refs['video-there'];
          videoThere.srcObject = stream;
        })
        .on('close', () => {
          const peer = this.peers[userId];
          if(peer !== undefined) {
            peer.destroy();
          }
          delete this.peers[userId];
        });
        this.peers[userId] = peer;
      } 
      return this.peers[userId];
    },
    async setupVideoChat() {
      // To show pusher errors
      // Pusher.logToConsole = true;

      // Get users media device information (a.k.a media stream)
      const stream = await navigator.mediaDevices.getUserMedia({ video: true, audio: true });
      // Vue has refs that allows you to grab the DOM element. Right now, this is creating
      // a pointer named videoHere to the div with ref=video-here
      const videoHere = this.$refs['video-here'];
      // assign the stream(video stream from our camera) to the videoHere DOM
      // element's srcObject(responsible for showing a media instance e.g video or something)
      videoHere.srcObject = stream;
      // set this.stream equal to stream coming from our camera
      this.stream = stream;

      // Create a pusher instance
      const pusher = this.getPusherInstance();
      // Use the pusher instance to subscribe(i.e just join) to a channel called presence-video-chat
      // and store that instance in this.channel
      this.channel = pusher.subscribe('presence-video-chat');
      // binding an event called "client-signal-{id}". When the event is triggered, the callback (2nd parameter)
      // is fired. The callback is the same format as laravel echo so I am guessing it somehow is getting data from
      // some event(or maybe from pusher). (see https://laravel.com/docs/8.x/broadcasting#listening-for-events)
      // might be replacable with laravel echo "channel.listen(eventName, callback);" format"
      this.channel.bind(`client-signal-${this.user.id}`, (signal) => 
      {
         // uses simple-peer. Creates an interface for WebRTC based implementations for video conferencing 
         // and data transfer. Signal is taking userId form the channel parameter ${this.user.id}
        const peer = this.getPeer(signal.userId, false);
        // Send signal data to the channel I think
        peer.signal(signal.data);
      });
    },
    // This function just gets/creates a pusher object instance. Needs pusher keys and all. Also binds a
    // csrf-token to the onject
    getPusherInstance() {
      return new Pusher(this.pusherKey, {
        authEndpoint: '/auth/video_chat',
        cluster: this.pusherCluster,
        auth: {
          headers: {
            'X-CSRF-Token': document.head.querySelector('meta[name="csrf-token"]').content
          }
        }
      });
    }
  }
};
</script>
<style>
.video-container {
  width: 500px;
  height: 380px;
  margin: 8px auto;
  border: 3px solid #000;
  position: relative;
  box-shadow: 1px 1px 1px #9e9e9e;
}
.video-here {
  width: 130px;
  position: absolute;
  left: 10px;
  bottom: 16px;
  border: 1px solid #000;
  border-radius: 2px;
  z-index: 2;
}
.video-there {
  width: 100%;
  height: 100%;
  z-index: 1;
}
.text-right {
  text-align: right;
}
</style>